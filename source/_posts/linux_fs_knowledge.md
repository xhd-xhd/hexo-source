---
title: linux文件系统知识
tags: [linux, 文件系统]
index_img: /img/Dolphin.png
date: 2026-04-26 23:00:00
author: xhd
---


# 🚀 Linux 文件系统知识大揭秘

> 本文档整理自与AI的聊天记录，深入探索开发板启动、文件系统、分区等核心知识。

---

## 🔥 问题 1: 开发板从SD卡启动的奥秘

**用户提问：**  
某些开发板可以从SD卡运行操作系统，我想问的是，SD卡里是我们烧录的img文件，开发板有BIOS吗？他是有SD卡的驱动？他默认从SD卡处运行代码？还是自带的SD卡驱动把img从SD卡加载到内存运行？

### 💡 AI 回答

**核心结论：**  
ARM开发板**没有PC那种BIOS**；芯片内部ROM固化了一段极小Bootloader；它自带SD卡硬件控制器+只读ROM驱动；开机先初始化SD卡，再把SD卡里的系统镜像读到DDR内存，最后跳转到内存运行Linux。

#### 🖥️ PC电脑 vs ARM开发板对比
- **PC电脑**：有BIOS，负责自检、初始化硬盘、读取硬盘系统、加载到内存。
- **ARM开发板**：没有BIOS，只有芯片内部Mask ROM（出厂固化只读ROM）。

#### ⚡ 完整开机全过程（一步一步）
1. **上电第一秒**：芯片内部Mask ROM运行，初始化CPU、时钟、自带内置SD卡控制器驱动。默认启动顺序：SD卡 → eMMC → Flash → UART下载。
2. **ROM固件读取SD卡里的引导程序（U-Boot）**：img镜像结构包括U-Boot、设备树、Linux内核、根文件系统。ROM用自带SD驱动把U-Boot读到DDR内存。
3. **U-Boot运行**：继续读SD卡，把Linux内核、设备树读到内存。
4. **U-Boot启动Linux内核**：CPU跳转到内存里的Linux内核。
5. **Linux内核启动后**：加载Linux版本SD卡驱动，挂载SD卡里的根文件系统。

#### ❓ 具体问题解答
- **开发板有BIOS吗？** 没有，只有芯片内部Mask ROM固化启动代码。
- **开发板有SD卡驱动吗？** 有三层：ROM固化最小SD只读驱动、U-Boot里的SD驱动、Linux内核里的SD驱动。
- **默认从SD卡运行代码？** 是，默认优先从SD启动。
- **自带SD驱动把img从SD加载到内存运行？** 是，Linux在内存运行，SD卡是仓库。

#### 🏗️ 层级总结
1. **芯片内部Mask ROM**：最底层、固化、自带SD驱动。
2. **U-Boot引导程序**：第二级、裸机、有SD驱动。
3. **Linux内核**：操作系统、有SD卡驱动。
4. **应用层**：用户程序。

---

## 🔥 问题 2: fire-config机制解密

**用户提问：**  
fire-config旨在提供一些常见的系统功能配置服务，在进行配置过程中，这可能会导致/boot/uEnv.txt或者是其他各种标准的linux配置文件被自动更改了，某些选项需要重启才能生效，如果您修改了其中一个，fire-config 会在<Finish> 按钮被选择时，询问您是否要立即重启，如果您希望配置马上生效，确定重启系统即可。  
我的理解是fire是用户态进程？还是uboot里的某个程序？不管怎么说，他肯定是修改了某个文件，而这个文件是系统启动的时候会读取的配置文件对吧？

### 💡 AI 回答

fire-config 是**Linux 用户态进程（应用程序）**，不是U-Boot里的程序。它修改 `/boot/uEnv.txt` 和其他配置文件，而uEnv.txt是U-Boot启动阶段读取的配置文件。

#### 🔗 逻辑链条
- Linux运行后，用户执行 `sudo fire-config`。
- fire-config修改 `/boot/uEnv.txt`。
- uEnv.txt是U-Boot用的，必须重启让U-Boot重新读取。

#### 🏗️ 三层关系
1. **fire-config**：Linux用户态应用程序，修改SD卡里的 /boot/uEnv.txt。
2. **uEnv.txt**：SD卡 /boot 分区里的文本配置文件，U-Boot启动时读取。
3. **U-Boot**：开机读uEnv.txt，按配置加载内核。

---

## 🔥 问题 3: uEnv.txt的位置和SD卡分区探秘

**用户提问：**  
uEnv.txt是在sd卡的哪个位置？sd卡有哪几个分区？sd卡里的内容是不是按照特定格式存储，比如你说的/boot分区下的uEnv.txt文件，这个/boot分区是什么？这个/boot分区是不是uboot可以读取，后面的linux也可以读取？

### 💡 AI 回答

SD卡分成多个分区，默认2个：BOOT分区（FAT32，存放uEnv.txt等）和RootFS分区（ext4，根文件系统）。

- **uEnv.txt位置**：SD卡第一个FAT32分区根目录，Linux路径 `/boot/uEnv.txt`。
- **分区结构**：
  - 分区1：FAT32 /boot启动分区（U-Boot可读，Linux可读写）。
  - 分区2：ext4根文件系统分区（U-Boot不能读，Linux专属）。
- **/boot分区**：U-Boot专用FAT32启动分区，存放开机文件。
- **读写权限**：U-Boot只能读FAT32的/boot，Linux能读写/boot和ext4。

---

## 🔥 问题 4: 分区和文件系统的深度理解

**用户提问：**  
我好像知道了，所谓分区，其实都是逻辑概念，实际上就一个sd卡，这个sd卡上烧录了两个文件系统，一个fat,一个ext4,不同的文件系统里存放了不同的内容，对于软件或者系统来说，他们认识的sd卡只是一个硬件而已，里面的文件系统才是他们想要的理想的格式的数据，rom是怎么从sd卡里搞到uboot的？他不仅有sd卡驱动，也有fat文件系统驱动？不然他怎么从磁盘里搞到uboot呢？在uboot运行的时候，uboot肯定也有fat驱动和sd卡驱动，从而访问fat里的数据，当linux运行之后，linux将fat挂载到/boot,自己的跟根文件系统挂载到/，所以linux里操作/boot下的文件其实就是操作fat下的文件，重启之后uboot就会看到linux改动过后的文件了？

### 💡 AI 回答

理解完全正确。分区是逻辑划分，SD卡物理一块空间。ROM有SD驱动和极简FAT驱动，能读取FAT分区里的u-boot.bin。U-Boot有SD+FAT驱动，能访问FAT数据。Linux挂载FAT到/boot，ext4到/，修改/boot文件就是修改SD卡FAT分区，重启后U-Boot看到修改。

#### 🔄 终极闭环逻辑
1. SD物理一块空间 → 逻辑分成FAT32 /boot、ext4 rootfs。
2. ROM：SD+FAT驱动 → 读FAT里的U-Boot → 加载到内存。
3. U-Boot：SD+FAT驱动 → 读FAT里uEnv.txt等 → 启动Linux。
4. Linux：SD+FAT+ext4驱动 → 挂载FAT到/boot，ext4到/。
5. fire-config修改 /boot/uEnv.txt → 写入SD卡FAT。
6. 重启 → U-Boot读取修改后的uEnv.txt。

---

## 🔥 问题 5: 目录和dentry的内核级理解

**用户提问：**  
目录也是一个特殊文件，他的信息记录在结构体dentry里面，这个结构体描述了这个目录的信息，比如inode指针，是否挂载到了其他文件系统以及驱动，当用户访问/boot/xxx的时候，访问到/会去ext4里找boot，找到了，发现boot的dentry信息里表明这个目录是挂载到sd卡第一个分区fat的，所以就会换一套fat驱动，去第一个分区找文件，是不是？

### 💡 AI 回答

完全正确。目录是特殊文件，信息在dentry结构体中，包括inode指针、挂载标记等。访问 /boot/xxx 时，从 / 开始在ext4找boot，发现dentry标记挂载到FAT，内核切换文件系统到FAT查找文件。

#### 🧠 内核逻辑
访问 /boot/uEnv.txt：
1. 从 /（ext4）开始。
2. 查找boot目录。
3. 查看dentry，发现mnt指针非空（挂载点）。
4. 切换到FAT32文件系统。
5. 在FAT32查找uEnv.txt。

#### 📝 总结
挂载 = 给目录dentry打标记，跳转到另一个文件系统。路径解析 = 一层找目录，遇挂载标记切换文件系统。

---

> 🎉 恭喜你！通过这些问题，你已经掌握了Linux文件系统的核心奥秘。继续探索吧！ 🚀