---
title: easy_flash移植
tags: [embedded, debug]
index_img: /img/easyflash/code.jpg
date: 2026-03-31 15:57:20
author: xhd
---

## EasyFlash 移植与踩坑记录

### 1. 简介

EasyFlash 是一个基于键值（Key-Value）存储的 Flash 管理库，非常适合用于嵌入式设备的标定数据管理。

**我的平台环境：**
- 32位单片机
- Flash 扇区大小：512B
- Flash 写入粒度：2 word = 8 byte
- 只使用 env 模式

### 2. 基本移植步骤

按照官方 README，将相关文件移植到项目中，实现 `ef_port.c`。
```c
EfErrCode ef_port_read(uint32_t addr, uint32_t *buf, size_t size) {
    //简单
}

EfErrCode ef_port_erase(uint32_t addr, size_t size) {
    //简单
}

EfErrCode ef_port_write(uint32_t addr, const uint32_t *buf, size_t size) {
    EfErrCode result = EF_NO_ERR;
    size_t i;
    uint64_t write_data, read_data;
    EF_ASSERT(addr % 8 == 0);
    for (i = 0; i < size; i += 8) {
        //由于目标单片机一次写入两个字（2 word），`ef_port_write` 需要注意字节序：
        write_data = ((uint64_t)buf[1] << 32) | buf[0];
        FLASH_Write2WordsWithECC(addr, write_data);
        read_data = *(uint64_t *)addr;
        if (read_data != write_data) {
            result = EF_WRITE_ERR;
            break;
        }
        buf += 2;
        addr += 8;
    }
    return result;
}

void ef_log_debug(const char *file, const long line, const char *format, ...) {
#ifdef PRINT_DEBUG
    va_list args;
    va_start(args, format);
    vprintf(format, args);
    va_end(args);
#endif
}

void ef_log_info(const char *format, ...) {
#ifdef PRINT_DEBUG
    va_list args;
    va_start(args, format);
    vprintf(format, args);
    va_end(args);
#endif
}

void ef_print(const char *format, ...) {
#ifdef PRINT_DEBUG
    va_list args;
    va_start(args, format);
    vprintf(format, args);
    va_end(args);
#endif
}
```

打开 IAR 半主机模式，直接使用打印函数vprintf(format, args);通过 JLink 的 SWD 接口与 PC 通信。
![IAR半主机模式](/img/easyflash/iar_semihosted.png)


### 3. 问题一：env 丢失

接口移植完成后，用 demo 测试，`boot_times` 每次重启都能读取并自增。但发现：一个扇区只能存放 5 条 `boot_times` 记录，第 6 次写入前，理论上应读取第 5 条记录并自增写入`boot_times = 6`到第二个扇区，但实际写到第二个扇区的 `boot_times = 1`。

![env丢失](/img/easyflash/ef_env_lost.png)

#### 问题分析

调试发现，第五次写入 flash 没问题，问题出在下一次重启后，`ef_get_env` 读取第一个扇区时，前 4 条记录都是无效的（已删除），准备读取下一条记录时（`ef_env.c -> find_next_env_addr()`），返回地址失败。

![find_next_env_addr问题](/img/easyflash/ef_find_next.png)

原因：
- 该函数在给定地址范围内寻找 env magic，但参数传递时 `end = sector->addr + SECTOR_SIZE - SECTOR_HDR_DATA_SIZE`，而 magic 恰好在 end 之后，导致没找到有效记录。
- 按理说扇区元数据应在扇区头（起始地址），不应在这里减去 `SECTOR_HDR_DATA_SIZE`。

**修正：**
```c
// xhd add
addr = find_next_env_addr(addr, sector->addr + SECTOR_SIZE - 0);
```
![运行正常](/img/easyflash/ef_boot_times_ok.png)

问题解决，第二个扇区正确写入 `boot_times = 6`。

### 问题二：垃圾回收 Hard Fault

给 EasyFlash 分配 4 个扇区，起始地址0xf800, `EF_GC_EMPTY_SEC_THRESHOLD = 1`。不断更改 env 时，一直追加到第 3 个扇区快满，触发垃圾回收，程序崩溃。

垃圾回收会将有效数据搬运到最后一个空扇区，依赖 `move_env` 实现。调试发现第一次 `move_env` 成功，第二次崩溃。

调用链：`alloc_env -> sector_iterator -> traversal_env = true -> read_sector_meta_data -> get_next_env_addr -> find_next_env_addr`

又回到了 `find_next_env_addr`，此时参数 `start = pre_env->addr.start + pre_env->len`，`end = 本扇区结束地址`。该函数内部读取数据时，访问了 end 之后的地址，而此扇区正好是 flash 最后一个扇区，直接导致 hard fault。

![内存越界](/img/easyflash/ef_mem_out.png)

**修正：保证读取不越界**
```c
static uint32_t find_next_env_addr(uint32_t start, uint32_t end) {
    uint8_t buf[32];
    uint32_t start_bak = start, i;
    uint32_t magic;

#ifdef EF_ENV_USING_CACHE
    uint32_t empty_env;
    if (get_sector_from_cache(EF_ALIGN_DOWN(start, SECTOR_SIZE), &empty_env)
            && start == empty_env) {
        return FAILED_ADDR;
    }
#endif

    if (end < sizeof(uint32_t) || start >= end - sizeof(uint32_t) + 1) {
        return FAILED_ADDR;
    }

    for (; start < end; start += (sizeof(buf) - sizeof(uint32_t))) {
        uint32_t readable = end - start;
        uint32_t read_len = sizeof(buf);
        if (readable < read_len) read_len = readable;
        uint32_t aligned_len = (read_len + 3u) & ~3u;
        if (aligned_len > sizeof(buf)) aligned_len = sizeof(buf);
        ef_port_read(start, (uint32_t *) buf, aligned_len);

        for (i = 0; i < sizeof(buf) - sizeof(uint32_t); i++) {
            if (start + i + sizeof(uint32_t) > end) break;
            if (i + sizeof(uint32_t) > aligned_len) break;
#ifndef EF_BIG_ENDIAN
            magic = buf[i]
                  + ((uint32_t)buf[i + 1] << 8)
                  + ((uint32_t)buf[i + 2] << 16)
                  + ((uint32_t)buf[i + 3] << 24);
#else
            magic = buf[i + 3]
                  + ((uint32_t)buf[i + 2] << 8)
                  + ((uint32_t)buf[i + 1] << 16)
                  + ((uint32_t)buf[i]     << 24);
#endif
            if (magic == ENV_MAGIC_WORD && (start + i - ENV_MAGIC_OFFSET) >= start_bak) {
                return start + i - ENV_MAGIC_OFFSET;
            }
        }
    }
    return FAILED_ADDR;
}
```

修正后垃圾回收正常。


### 4. 总结

| 问题 | 原因 | 修正 |
|------|------|------|
| env 丢失，boot_times 重置为 1 | `find_next_env_addr` 的 end 参数偏小，漏掉最后一条有效记录 | end 不减 `SECTOR_HDR_DATA_SIZE` |
| 垃圾回收 Hard Fault | `find_next_env_addr` 读取越界，访问了 flash 末尾之后的地址 | 读取前检查边界，限制读取长度 |

