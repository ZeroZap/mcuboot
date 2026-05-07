# MCUboot 详细中文指导文档

> 版本: 基于 MCUboot v2.4.0
> 许可: Apache-2.0

---

## 目录

1. [MCUboot 概述](#1-mcuboot-概述)
2. [架构设计](#2-架构设计)
3. [镜像格式详解](#3-镜像格式详解)
4. [Flash 分区与布局](#4-flash-分区与布局)
5. [启动流程详解](#5-启动流程详解)
6. [镜像升级策略](#6-镜像升级策略)
7. [安全机制](#7-安全机制)
8. [配置体系详解](#8-配置体系详解)
9. [工具链 (imgtool)](#9-工具链-imgtool)

---
**第一篇: 裸机移植指南**
10. [裸机移植全景](#10-裸机移植全景)
11. [Flash Map 实现](#11-flash-map-实现)
12. [Flash 驱动 API 实现](#12-flash-驱动-api-实现)
13. [启动入口与跳转](#13-启动入口与跳转)
14. [日志与断言适配](#14-日志与断言适配)
15. [看门狗与空闲处理](#15-看门狗与空闲处理)
16. [完整裸机移植示例 (STM32F4)](#16-完整裸机移植示例-stm32f4)

---
**第二篇: 第三方算法移植指南**
17. [加密子系统架构](#17-加密子系统架构)
18. [自定义加密后端 (MCUBOOT_USE_CUSTOM_CRYPTO)](#18-自定义加密后端-mcuboot_use_custom_crypto)
19. [SHA-256 实现适配](#19-sha-256-实现适配)
20. [ECDSA 签名验证适配](#20-ecdsa-签名验证适配)
21. [镜像加密相关适配](#21-镜像加密相关适配)
22. [完整自定义加密后端示例 (基于硬件加速)](#22-完整自定义加密后端示例-基于硬件加速)

---
**第三篇: 裁剪移植指南**
23. [功能裁剪全景](#23-功能裁剪全景)
24. [升级策略裁剪](#24-升级策略裁剪)
25. [签名算法裁剪](#25-签名算法裁剪)
26. [加密功能裁剪](#26-加密功能裁剪)
27. [多镜像与依赖裁剪](#27-多镜像与依赖裁剪)
28. [日志/测量启动/数据共享裁剪](#28-日志测量启动数据共享裁剪)
29. [Boot Serial 裁剪](#29-boot-serial-裁剪)
30. [极端精简配置 (最小化 MCUboot)](#30-极端精简配置-最小化-mcuboot)

---
**第四篇: 串口升级移植指南 (Boot Serial)**
33. [串口升级协议与架构](#33-串口升级协议与架构)
34. [裸机串口移植必须实现的部分](#34-裸机串口移植必须实现的部分)
35. [UART 驱动适配 (boot_uart_funcs)](#35-uart-驱动适配-boot_uart_funcs)
36. [OS 抽象层适配 (os/os_malloc/hal_system)](#36-os-抽象层适配-osos_mallochal_system)
37. [ZCbor 与依赖库集成](#37-zcbor-与依赖库集成)
38. [串口恢复模式入口与跳转决策](#38-串口恢复模式入口与跳转决策)
39. [完整串口升级移植示例 (HC32L196 / Cortex-M0+)](#39-完整串口升级移植示例-hc32l196--cortex-m0)
40. [Cortex-M0+ 无 VTOR 跳转专题](#40-cortex-m0-无-vtor-跳转专题)
41. [双 Slot 链接地址与 Keil 工程管理](#41-双-slot-链接地址与-keil-工程管理)

---
**附录**
42. [常见问题 FAQ](#42-常见问题-faq)
43. [参考资源](#43-参考资源)

---

## 1. MCUboot 概述

### 1.1 什么是 MCUboot

MCUboot 是一个面向 32 位微控制器的**安全引导加载程序 (Secure Bootloader)**。它定义了一套通用的 bootloader 基础设施和 Flash 布局规范，并提供安全的固件升级能力。

核心特性:
- **不依赖特定 OS 或硬件** — 通过硬件移植层适配各种平台
- **A/B 双区镜像** — 支持主槽 (Primary Slot) 和副槽 (Secondary Slot) 的双镜像升级
- **签名验证** — 支持 RSA-2048/3072、ECDSA-P256、ED25519 等签名算法
- **镜像加密** — 支持 AES-CTR-128/256 加密的镜像升级
- **回滚保护** — 防止设备被恶意降级到有漏洞的旧版本
- **多镜像支持** — 支持同时管理多个独立固件镜像
- **串口恢复** — 通过 SMP 协议支持串口固件上传

### 1.2 已支持的平台

MCUboot 目前已在以下 OS/SoC 上有完整移植:

| 平台 | 说明 |
|------|------|
| Zephyr | 主要支持平台，最完整的集成 |
| Apache Mynewt | MCUboot 的发源地 |
| Apache NuttX | 社区移植 |
| RIOT | 仅作为启动目标 |
| Mbed OS | ARM 生态 |
| Espressif (ESP32) | WiFi SoC 平台 |
| Cypress/Infineon (PSoC 6) | 支持内部 + 外部 Flash |

### 1.3 源代码结构

```
mcuboot/
├── boot/
│   ├── bootutil/              ← 核心库 (平台无关)
│   │   ├── include/bootutil/   ← 公共头文件
│   │   └── src/                ← 实现源码
│   │       ├── loader.c        ← boot_go() 主入口
│   │       ├── image_validate.c← 镜像验证
│   │       ├── swap_*.c        ← 交换算法
│   │       └── encrypted.c     ← 加密解密
│   ├── boot_serial/           ← 串口恢复功能 (可选)
│   ├── zephyr/                ← Zephyr 移植
│   ├── mynewt/                ← Mynewt 移植
│   ├── nuttx/                 ← NuttX 移植
│   ├── mbed/                  ← Mbed OS 移植
│   ├── espressif/             ← ESP32 移植
│   └── cypress/               ← PSoC 6 移植
├── docs/                      ← 文档
├── scripts/
│   └── imgtool.py             ← 镜像签名/加密工具
├── sim/                       ← 模拟器 (Rust 实现)
└── samples/                   ← 配置模板
```

---

## 2. 架构设计

### 2.1 两层架构

MCUboot 采用**两层分离**架构：

```
┌─────────────────────────────────────┐
│     Boot Application (启动应用)      │
│  - 入口点 (main)                     │
│  - 调用 boot_go()                    │
│  - 跳转到用户固件                     │
│  - 平台初始化 (时钟/外设)            │
├─────────────────────────────────────┤
│     Bootutil Library (核心库)        │
│  - 镜像读取与解析                    │
│  - 签名验证                          │
│  - Hash 校验                         │
│  - 镜像交换 (Swap)                   │
│  - 加密/解密                         │
│  - 回滚保护                          │
└─────────────────────────────────────┘
```

* **bootutil 库 (boot/bootutil)**: 实现 bootloader 的核心功能, 可通过单元测试验证
* **Boot Application (boot/\<port\>)**: 每个移植的启动应用, 负责平台初始化和最终跳转

核心理念: **库可以被测试, 应用不能。** 因此所有逻辑尽可能放入 bootutil。

### 2.2 关键数据结构 — boot_rsp

`boot_go()` 是 bootloader 的主入口, 返回 `boot_rsp` 结构告知启动应用应当跳转到哪个镜像:

```c
struct boot_rsp {
    const struct image_header *br_hdr;  // 要执行的镜像头指针
    uint8_t  br_flash_dev_id;           // Flash 设备 ID
    uint32_t br_image_off;              // 镜像头在 Flash 中的偏移
};
```

### 2.3 启动应用的责任

每个移植的启动应用至少需要:
1. 初始化硬件 (时钟、Flash 控制器、看门狗等)
2. 调用 `boot_go(&rsp)` 执行 bootloader 逻辑
3. 根据 `boot_rsp` 中的信息跳转到用户固件入口

---

## 3. 镜像格式详解

### 3.1 镜像整体结构

```
+---------------------+  ← 镜像开头
|  image_header (32B) |  ← 固定 32 字节
+---------------------+
|                     |
|   Payload (固件)    |  ← 实际应用程序代码
|                     |
+---------------------+
|  TLV 保护区 (可选)   |  ← ih_protect_tlv_size 字节
|  (被 hash/签名覆盖)  |
+---------------------+
|  TLV 信息头         |  ← IMAGE_TLV_INFO_MAGIC (0x6907)
+---------------------+
|  SHA256 Hash TLV    |  ← 必须存在
+---------------------+
|  KeyHash TLV (可选) |  ← 用于定位验证公钥
+---------------------+
|  Signature TLV      |  ← 签名值
+---------------------+
|  Security Counter   |  ← 安全计数器 (可选)
|  TLV (可选)         |
+---------------------+
|  Boot Record TLV    |  ← 测量启动记录 (可选)
|  (可选)             |
+---------------------+
|  Image Trailer      |  ← Swap 状态 / 魔法值
|  (仅swap模式)       |
+---------------------+
```

### 3.2 image_header 结构 (32 字节, 小端)

```c
struct image_header {
    uint32_t ih_magic;              // 0x96f3b83d (固定魔数)
    uint32_t ih_load_addr;          // 加载地址 (RAM 加载模式用)
    uint16_t ih_hdr_size;           // 头大小 (通常 32)
    uint16_t ih_protect_tlv_size;   // 受保护 TLV 区域大小
    uint32_t ih_img_size;           // 有效负载大小 (不含头)
    uint32_t ih_flags;              // 标志位
    struct image_version ih_ver;    // 版本号
    uint32_t _pad1;                 // 保留
};

struct image_version {
    uint8_t  iv_major;              // 主版本号
    uint8_t  iv_minor;              // 次版本号
    uint16_t iv_revision;           // 修订号
    uint32_t iv_build_num;          // 构建号
};
```

### 3.3 镜像标志位 (ih_flags)

| 标志 | 值 | 说明 |
|------|----|------|
| IMAGE_F_PIC | 0x00000001 | 位置无关 (不支持) |
| IMAGE_F_ENCRYPTED_AES128 | 0x00000004 | AES-128 加密 |
| IMAGE_F_ENCRYPTED_AES256 | 0x00000008 | AES-256 加密 |
| IMAGE_F_NON_BOOTABLE | 0x00000010 | 非可启动 (拆分镜像) |
| IMAGE_F_RAM_LOAD | 0x00000020 | 加载到 RAM 执行 |

### 3.4 TLV (Type-Length-Value) 格式

```c
struct image_tlv_info {
    uint16_t it_magic;     // 0x6907 或 0x6908 (受保护)
    uint16_t it_tlv_tot;   // TLV 区域总大小
};

struct image_tlv {
    uint16_t it_type;      // TLV 类型
    uint16_t it_len;       // 数据长度 (不含头)
};
```

### 3.5 主要 TLV 类型

| TLV 类型 | 值 | 说明 |
|----------|------|------|
| IMAGE_TLV_KEYHASH | 0x01 | 公钥 hash |
| IMAGE_TLV_PUBKEY | 0x02 | 完整公钥 |
| IMAGE_TLV_SHA256 | 0x10 | SHA256 hash |
| IMAGE_TLV_SHA384 | 0x11 | SHA384 hash |
| IMAGE_TLV_SHA512 | 0x12 | SHA512 hash |
| IMAGE_TLV_RSA2048_PSS | 0x20 | RSA-2048 签名 |
| IMAGE_TLV_ECDSA_SIG | 0x22 | ECDSA-P256 签名 |
| IMAGE_TLV_RSA3072_PSS | 0x23 | RSA-3072 签名 |
| IMAGE_TLV_ED25519 | 0x24 | Ed25519 签名 |
| IMAGE_TLV_ENC_RSA2048 | 0x30 | RSA-OAEP 加密密钥 |
| IMAGE_TLV_ENC_KW | 0x31 | AES-KW 加密密钥 |
| IMAGE_TLV_ENC_EC256 | 0x32 | ECIES-P256 加密密钥 |
| IMAGE_TLV_ENC_X25519 | 0x33 | ECIES-X25519 加密密钥 |
| IMAGE_TLV_DEPENDENCY | 0x40 | 镜像依赖 |
| IMAGE_TLV_SEC_CNT | 0x50 | 安全计数器 |
| IMAGE_TLV_BOOT_RECORD | 0x60 | 测量启动记录 |

---

## 4. Flash 分区与布局

### 4.1 Flash 地图 (Flash Map)

Flash Map 是 MCUboot 对设备 Flash 分区的抽象。每个分区由唯一 ID 标识, 通过 `struct flash_area` 描述:

```c
struct flash_area {
    uint8_t  fa_id;          // 分区 ID
    uint8_t  fa_device_id;   // 设备 ID
    uint16_t pad16;
    uint32_t fa_off;         // 设备内偏移
    uint32_t fa_size;        // 分区大小
};
```

### 4.2 标准 Flash 区域 ID

**单镜像配置:**

| 区域 | ID | 说明 |
|------|------|------|
| FLASH_AREA_BOOTLOADER | 0 | Bootloader 自身 |
| FLASH_AREA_IMAGE_PRIMARY(0) | 1 | 主槽 (Primary Slot) |
| FLASH_AREA_IMAGE_SECONDARY(0) | 2 | 副槽 (Secondary Slot) |
| FLASH_AREA_IMAGE_SCRATCH | 3 | 暂存区 (Swap 模式) |

**双镜像配置:**

| 区域 | ID | 说明 |
|------|------|------|
| FLASH_AREA_BOOTLOADER | 0 | Bootloader |
| FLASH_AREA_IMAGE_PRIMARY(0) | 1 | 镜像 0 主槽 |
| FLASH_AREA_IMAGE_SECONDARY(0) | 2 | 镜像 0 副槽 |
| FLASH_AREA_IMAGE_SCRATCH | 3 | 暂存区 |
| FLASH_AREA_IMAGE_PRIMARY(1) | 5 | 镜像 1 主槽 |
| FLASH_AREA_IMAGE_SECONDARY(1) | 6 | 镜像 1 副槽 |

### 4.3 单镜像 Flash 布局示例

```
+--------------------------+ ← Flash 基地址 (如 0x08000000)
|                          |
|  MCUboot Bootloader      |  FLASH_AREA_BOOTLOADER (ID=0)
|  (如 48KB)               |
|                          |
+--------------------------+
|                          |
|  Primary Slot (主槽)     |  FLASH_AREA_IMAGE_PRIMARY (ID=1)
|  (如 256KB)              |  正常运行时的固件位置
|                          |
+--------------------------+
|                          |
|  Secondary Slot (副槽)   |  FLASH_AREA_IMAGE_SECONDARY (ID=2)
|  (如 256KB)              |  新固件下载位置
|                          |
+--------------------------+
|                          |
|  Scratch Area (暂存区)   |  FLASH_AREA_IMAGE_SCRATCH (ID=3)
|  (如 4KB)                |  Swap 时临时存储
|                          |
+--------------------------+
```

### 4.4 Image Trailer (镜像尾部元数据)

每个镜像槽的**末尾**有一个 trailer 区域存储 Swap 状态:

```
+---------------------------+ ← 镜像槽末尾 (最高地址)
|      Magic (16 bytes)     |  用于检测 trailer 存在
+---------------------------+
|     Image OK (1 byte)     |  0x01=确认, 0xff=未确认
+---------------------------+
|     Copy Done (1 byte)    |  0x01=完成, 0xff=未完成
+---------------------------+
|     Swap Info (1 byte)    |  低4位: swap类型, 高4位: 镜像编号
+---------------------------+
|     Swap Size (4 bytes)   |  已交换的总大小
+---------------------------+
|  Encryption Keys (可选)   |  仅加密模式
+---------------------------+
|  Swap Status              |  每个sector 2~3字节状态记录
|  (BOOT_MAX_IMG_SECTORS    |
|   × min_write_size × N)   |
+---------------------------+ ← 镜像有效数据结束位置
```

**Trailer 占用空间计算:**
- Swap Status: `BOOT_MAX_IMG_SECTORS × min_write_size × s`
  - `s = 3` (swap using scratch/move)
  - `s = 2` (swap using offset)
- 例如: 128 个 sector, 4 字节对齐, `s=3` → 1536 字节

**有效镜像最大尺寸:**
```
max_image_size = slot_size - trailer_size
```

---

## 5. 启动流程详解

### 5.1 启动流程总览 (Swap 模式)

```
上电/复位
    │
    ▼
┌─────────────────┐
│ 1. 检查 Swap 状态 │ ← 是否有中断的 Swap 需要恢复?
└────────┬────────┘
         │
    ┌────▼────┐  是
    │ 恢复Swap │────── 继续之前中断的 Swap 操作
    └────┬────┘
         │
         ▼
┌─────────────────┐
│ 2. 检查镜像 Trailer│ ← 是否需要 Swap?
└────────┬────────┘
         │
    ┌────▼────────────┐
    │ 需要 Swap?       │
    └────┬────────────┘
         │ 是
    ┌────▼──────────────────┐
    │ 3. 验证副槽镜像        │ ← Hash + 签名验证
    │    有效?               │
    └────┬──────────────────┘
         │ 是              │ 否
    ┌────▼──────┐   ┌──────▼──────┐
    │ 4. 执行Swap│   │ 擦除无效镜像 │
    │ 写 Trailer │   │ 标记失败     │
    └────┬──────┘   └──────┬──────┘
         │                 │
         ▼                 ▼
┌─────────────────┐
│ 5. 引导主槽镜像  │ ← 直接启动或 Swap 后启动
└─────────────────┘
```

### 5.2 Swap 类型决策 (非恢复场景)

MCUboot 通过检查主/副槽 trailer 中的 `magic`、`image_ok`、`copy_done` 标志来决定 swap 类型:

| 状态 | 主槽 magic | 副槽 magic | 主槽 image_ok | 副槽 image_ok | 主槽 copy_done | 结果 |
|------|-----------|-----------|---------------|---------------|----------------|------|
| State I (offset only) | Any | Good | Any | Unset | Any | REVERT |
| State II | Any | Good | Any | Unset | Any | TEST |
| State III | Any | Good | Any | 0x01 | Any | PERM |
| State IV | Good | Any | 0xff | Any | 0x01 | REVERT |
| State V | Any | Any | Any | Any | Any | NONE/FAIL/PANIC |

**Swap 类型含义:**

| 类型 | 值 | 行为 |
|------|------|------|
| BOOT_SWAP_TYPE_NONE | 1 | 不交换, 直接启动主槽 |
| BOOT_SWAP_TYPE_TEST | 2 | 测试性交换: 下次启动未确认则回滚 |
| BOOT_SWAP_TYPE_PERM | 3 | 永久交换 |
| BOOT_SWAP_TYPE_REVERT | 4 | 回滚: 上次测试未确认 |
| BOOT_SWAP_TYPE_FAIL | 5 | 交换失败: 镜像无效 |
| BOOT_SWAP_TYPE_PANIC | 0xff | 不可恢复错误 |

### 5.3 多镜像启动流程

多镜像模式下, 启动分为 4 个循环:

```
Loop 1: 逐镜像检查 Swap 状态 → 恢复中断的 Swap
Loop 2: 逐镜像检查依赖关系 → 调整 Swap 类型
Loop 3: 逐镜像执行 Swap 或 Overwrite
Loop 4: 逐镜像验证并启动
```

---

## 6. 镜像升级策略

### 6.1 升级策略对比

| 策略 | 宏定义 | Flash 需求 | 说明 |
|------|--------|-----------|------|
| Swap using Scratch (默认) | 默认 | 3 区域 + Scratch | 通过暂存区交换, 最灵活 |
| Swap using Move | MCUBOOT_SWAP_USING_MOVE | 3 区域 (Primary 多1 sector) | 无需 Scratch |
| Swap using Offset | MCUBOOT_SWAP_USING_OFFSET | 3 区域 (Secondary 多1 sector) | 推荐的无 Scratch 方案 |
| Overwrite Only | MCUBOOT_OVERWRITE_ONLY | 2 区域 (无 Scratch) | 直接覆盖, 简单但无回滚 |
| Direct-XIP | MCUBOOT_DIRECT_XIP | 2 区域 (等大) | 直接从副槽执行 |
| RAM Load | MCUBOOT_RAM_LOAD | 外部存储 + RAM | 加载到 RAM 执行 |

### 6.2 Overwrite Only (覆盖模式)

最简单的升级模式 — 直接将副槽内容覆盖到主槽:
- 无回滚能力
- 无需 Scratch 区域
- 支持 `MCUBOOT_DOWNGRADE_PREVENTION` 防降级
- 可选 `MCUBOOT_OVERWRITE_ONLY_FAST`: 仅擦写需要的 sector

### 6.3 Direct-XIP (原地执行模式)

- 两个等大的 slot, 均可直接执行
- 启动时选择版本号最新的镜像
- 支持回滚: 设置 `MCUBOOT_DIRECT_XIP_REVERT`
- **不支持镜像加密**

### 6.4 Swap 交换流程详解

以 Swap using Scratch 为例, 按 sector 逆序交换:

```
对每个 sector index (从大到小):
  a. 擦除 Scratch
  b. 复制 Secondary[index] → Scratch
  c. 写 Swap 状态 (i)
  d. 擦除 Secondary[index]
  e. 复制 Primary[index] → Secondary[index]
  f. 写 Swap 状态 (ii)
  g. 擦除 Primary[index]
  h. 复制 Scratch → Primary[index]
  i. 写 Swap 状态 (iii)
```

Swap 状态用每 sector 3 字节记录:
```
状态0: 0xff 0xff 0xff ← 初始
状态1: 0x01 0xff 0xff ← 已复制 Secondary → Scratch
状态2: 0x01 0x02 0xff ← 已复制 Primary → Secondary
状态3: 0x01 0x02 0x03 ← 已完成
```

---

## 7. 安全机制

### 7.1 签名验证

MCUboot 支持的签名算法:

| 算法 | 宏定义 | 密钥长度 | 说明 |
|------|--------|---------|------|
| RSA-2048 | MCUBOOT_SIGN_RSA | 2048-bit | 传统, 验证慢 |
| RSA-3072 | MCUBOOT_SIGN_RSA | 3072-bit | 更高安全级别 |
| ECDSA-P256 | MCUBOOT_SIGN_EC256 | 256-bit | 快速, 推荐 |
| ECDSA-P384 | MCUBOOT_SIGN_EC384 | 384-bit | 更高安全级别 |
| Ed25519 | MCUBOOT_SIGN_ED25519 | 256-bit | 现代算法 |

### 7.2 公钥管理方式

| 方式 | 宏定义 | 说明 |
|------|--------|------|
| 嵌入公钥 (默认) | (无特殊宏) | 公钥编译进 bootloader, TLV 中放 keyhash |
| 硬件密钥 | MCUBOOT_HW_KEY | 公钥 hash 存储在硬件中, TLV 中放完整公钥 |
| 内置密钥 | MCUBOOT_BUILTIN_KEY | 密钥完全由加密库管理 (仅 PSA Crypto) |

### 7.3 回滚保护

**软件防降级** (`MCUBOOT_DOWNGRADE_PREVENTION`):
- 仅 Overwrite Only 模式
- 比较镜像版本号, 新版本必须 ≥ 旧版本

**硬件防降级** (`MCUBOOT_HW_ROLLBACK_PROT`):
- 镜像 TLV 中携带安全计数器 (Security Counter)
- 与设备中存储的安全计数器比较
- 需实现 `security_cnt.h` 接口

### 7.4 FIH 故障注入加固

`MCUBOOT_FIH_PROFILE_ON` 系列宏启用故障注入加固:
- 对关键决策点进行双重/多重检查
- 可添加随机延迟 (`MCUBOOT_FIH_PROFILE_MAX`)
- 防止通过电压/时钟毛刺绕过签名验证

### 7.5 镜像加密

支持以下密钥加密方式:

| 加密方式 | 宏定义 | TLV 类型 |
|---------|--------|---------|
| RSA-OAEP | MCUBOOT_ENCRYPT_RSA | 0x30 |
| AES-KW | MCUBOOT_ENCRYPT_KW | 0x31 |
| ECIES-P256 | MCUBOOT_ENCRYPT_EC256 | 0x32 |
| ECIES-X25519 | MCUBOOT_ENCRYPT_X25519 | 0x33 |
| ECIES-X25519+SHA512 | - | 0x34 |

加密流程: 镜像 Payload 用 AES-CTR-128/256 加密, AES 密钥用上述方式加密后存储在镜像 TLV 中。

---

## 8. 配置体系详解

### 8.1 配置文件位置

MCUboot 的所有配置通过 `mcuboot_config/mcuboot_config.h` 文件进行。每个移植需要提供此文件。

默认模板: `samples/mcuboot_config/mcuboot_config.template.h`

现有移植示例:
- `boot/zephyr/include/mcuboot_config/mcuboot_config.h` (Zephyr)
- `boot/mynewt/mcuboot_config/include/mcuboot_config/mcuboot_config.h` (Mynewt)

### 8.2 配置选项分类

#### 必须选择一项的配置:

**签名类型 (三选一):**
```c
#define MCUBOOT_SIGN_RSA       // RSA 签名
#define MCUBOOT_SIGN_EC256     // ECDSA-P256 签名
#define MCUBOOT_SIGN_EC384     // ECDSA-P384 签名
#define MCUBOOT_SIGN_ED25519   // Ed25519 签名
```

**加密后端 (二选一):**
```c
#define MCUBOOT_USE_MBED_TLS   // 使用 Mbed TLS
#define MCUBOOT_USE_TINYCRYPT  // 使用 Tinycrypt
#define MCUBOOT_USE_CUSTOM_CRYPTO // 自定义加密后端
```

#### 可选配置:

**升级模式:**
```c
// #define MCUBOOT_OVERWRITE_ONLY       // 覆盖模式
// #define MCUBOOT_OVERWRITE_ONLY_FAST  // 快速覆盖 (仅擦写需要的)
// #define MCUBOOT_DIRECT_XIP           // 原地执行
// #define MCUBOOT_DIRECT_XIP_REVERT    // Direct-XIP 回滚
// #define MCUBOOT_RAM_LOAD             // RAM 加载模式
```

**Swap 算法 (默认是 swap using scratch):**
```c
// #define MCUBOOT_SWAP_USING_MOVE     // 使用 Move 算法 (无 Scratch)
// #define MCUBOOT_SWAP_USING_OFFSET   // 使用 Offset 算法 (无 Scratch)
```

**镜像加密:**
```c
// #define MCUBOOT_ENCRYPT_RSA
// #define MCUBOOT_ENCRYPT_KW
// #define MCUBOOT_ENCRYPT_EC256
// #define MCUBOOT_ENCRYPT_X25519
```

**安全相关:**
```c
#define MCUBOOT_VALIDATE_PRIMARY_SLOT   // 启动时验证主槽
// #define MCUBOOT_VALIDATE_PRIMARY_SLOT_ONCE // 验证一次后缓存
// #define MCUBOOT_HW_ROLLBACK_PROT      // 硬件回滚保护
// #define MCUBOOT_DOWNGRADE_PREVENTION  // 软件防降级
// #define MCUBOOT_HW_KEY               // 硬件密钥模式
// #define MCUBOOT_BUILTIN_KEY          // 内置密钥模式
```

**功能与优化:**
```c
#define MCUBOOT_MAX_IMG_SECTORS        128   // 每 slot 最大 sector 数
#define MCUBOOT_IMAGE_NUMBER           1     // 镜像数量
#define MCUBOOT_HAVE_LOGGING           1     // 启用日志
// #define MCUBOOT_HAVE_ASSERT_H              // 自定义断言
// #define MCUBOOT_USE_FLASH_AREA_GET_SECTORS // 使用 sector 查询 API
// #define MCUBOOT_MEASURED_BOOT              // 测量启动
// #define MCUBOOT_DATA_SHARING               // 数据共享
// #define MCUBOOT_USE_TLV_ALLOW_LIST         // TLV 白名单
```

---

## 9. 工具链 (imgtool)

### 9.1 安装

```bash
pip3 install --user -r scripts/requirements.txt
```

### 9.2 生成密钥

```bash
# ECDSA-P256 密钥
./scripts/imgtool.py keygen -k mykey.pem -t ecdsa-p256

# RSA-2048 密钥
./scripts/imgtool.py keygen -k mykey.pem -t rsa-2048

# Ed25519 密钥
./scripts/imgtool.py keygen -k mykey.pem -t ed25519
```

### 9.3 提取公钥

```bash
# 输出为 C 数据
./scripts/imgtool.py getpub -k mykey.pem

# 输出为 PEM
./scripts/imgtool.py getpub -k mykey.pem -e pem
```

### 9.4 签名镜像

```bash
./scripts/imgtool.py sign \
    -k mykey.pem \
    --align 8 \
    -v "1.0.0" \
    -H 0x200 \
    --pad \
    --slot-size 0x40000 \
    myapp.bin \
    myapp.signed.bin
```

**关键参数:**
| 参数 | 说明 |
|------|------|
| -k | 私钥文件 |
| --align | Flash 写对齐 (1/2/4/8/16/32) |
| -v | 版本号 (如 1.0.0+0) |
| -H | 镜像头偏移 (header size) |
| --pad | 填充到 slot 大小并加 trailer |
| --slot-size | Slot 大小 (字节) |
| --confirm | 标记为已确认 (永久) |
| --test | 标记为测试 |
| -E | 加密镜像 (需提供加密公钥) |
| -s | 安全计数器值 |
| -d | 依赖声明 `"(image_id, version)"` |
| --overwrite-only | 覆盖模式镜像 |
| --max-sectors | 最大 sector 数 |

### 9.5 查看镜像信息

```bash
# 验证签名并显示信息
./scripts/imgtool.py verify -k mykey.pub.pem myapp.signed.bin
```

---

---

# 第一篇: 裸机移植指南

## 10. 裸机移植全景

### 10.1 裸机移植的含义

"裸机移植"指在**无操作系统的微控制器**上运行 MCUboot。这意味着你需要自己提供:

- Flash 驱动 (读/写/擦除)
- 内存分配 (用于 Mbed TLS)
- 平台初始化 (时钟、GPIO、看门狗等)
- 启动跳转逻辑

### 10.2 裸机移植的最小文件清单

```
my_baremetal_port/
├── mcuboot_config/
│   └── mcuboot_config.h          ← MCUboot 配置文件
├── flash_map_backend/
│   └── flash_map_backend.h       ← Flash Map 后端实现
├── sysflash/
│   └── sysflash.h                ← 系统 Flash 定义
├── hal/
│   ├── flash_drv.c               ← Flash 驱动实现
│   └── flash_drv.h               ← Flash 驱动头文件
├── crypto/                        ← 加密相关 (见第二篇)
├── main.c                         ← 启动入口
├── keys.c                         ← 公钥定义
├── linkerscript.ld                ← 链接脚本
└── Makefile / CMakeLists.txt
```

### 10.3 移植的 6 个步骤

1. **创建 `mcuboot_config.h`** — 选择合适的配置选项
2. **实现 Flash Map** — 定义 Flash 分区布局
3. **实现 Flash 驱动 API** — 提供 `flash_area_*` 系列函数
4. **提供公钥** — 嵌入签名验证公钥
5. **编写启动入口** — 初始化平台, 调用 `boot_go()`, 跳转
6. **集成加密库** — 链接 Mbed TLS 或实现自定义后端

---

## 11. Flash Map 实现

### 11.1 sysflash.h — Flash 区域定义

```c
// sysflash.h
#ifndef __SYSFLASH_H__
#define __SYSFLASH_H__

#include <mcuboot_config/mcuboot_config.h>

/* Flash 设备 ID */
#define FLASH_DEVICE_INTERNAL_FLASH  0
#define FLASH_DEVICE_ID              0

/* Flash 基地址与大小 (以 STM32F407 为例) */
#define FLASH_BASE                   0x08000000
#define FLASH_SIZE                   0x00100000  // 1MB

/* 分区大小定义 */
#define BOOTLOADER_SIZE              0x00010000  // 64KB
#define SLOT_SIZE                    0x00070000  // 448KB
#define SCRATCH_SIZE                 0x00002000  // 8KB

/* Flash 区域 ID 定义 (必须与 flash_map 中的 ID 对应) */
#define FLASH_AREA_BOOTLOADER         0
#define FLASH_AREA_IMAGE_PRIMARY(i)   (1 + (i))
#define FLASH_AREA_IMAGE_SECONDARY(i) (2 + (i))
#define FLASH_AREA_IMAGE_SCRATCH      3

#endif /* __SYSFLASH_H__ */
```

### 11.2 flash_map_backend.h — Flash Map 实现

```c
// flash_map_backend.h
#ifndef __FLASH_MAP_BACKEND_H__
#define __FLASH_MAP_BACKEND_H__

#include <stdint.h>
#include <sysflash/sysflash.h>

/* flash_area 结构体 (必须按此格式定义) */
struct flash_area {
    uint8_t  fa_id;          /* 区域 ID */
    uint8_t  fa_device_id;   /* Flash 设备 ID */
    uint16_t pad16;
    uint32_t fa_off;         /* 设备内偏移 */
    uint32_t fa_size;        /* 区域大小 */
};

/* flash_sector 结构体 */
struct flash_sector {
    uint32_t fs_off;         /* sector 偏移 */
    uint32_t fs_size;        /* sector 大小 */
};

/* --- 以下为必须实现的 API --- */

/**
 * 打开指定 ID 的 Flash 区域
 * @param id  区域 ID
 * @param fa  输出: 指向 flash_area 结构体的指针
 * @return 0 成功, 非0 失败
 */
int flash_area_open(uint8_t id, const struct flash_area **fa);

/**
 * 关闭 Flash 区域
 * 注意: MCUboot 可能会多次打开同一区域, 建议实现引用计数
 */
void flash_area_close(const struct flash_area *fa);

/**
 * 读取 Flash 数据
 */
int flash_area_read(const struct flash_area *fa, uint32_t off,
                    void *dst, uint32_t len);

/**
 * 写入 Flash 数据
 */
int flash_area_write(const struct flash_area *fa, uint32_t off,
                     const void *src, uint32_t len);

/**
 * 擦除 Flash 区域
 */
int flash_area_erase(const struct flash_area *fa, uint32_t off,
                     uint32_t len);

/**
 * 获取 Flash 写对齐 (最小写入单位)
 */
uint32_t flash_area_align(const struct flash_area *fa);

/**
 * 获取擦除后 Flash 字节值 (0xff 或 0x00)
 */
uint8_t flash_area_erased_val(const struct flash_area *fa);

/**
 * 根据 image_index 和 slot 获取 Flash 区域 ID
 */
int flash_area_id_from_multi_image_slot(int image_index, int slot);

/**
 * 根据 image_index 和 area_id 获取 slot 编号
 */
int flash_area_id_to_multi_image_slot(int image_index, int area_id);

/**
 * 获取 Flash 设备基地址 (如果平台需要)
 */
int flash_device_base(uint8_t fd_id, uintptr_t *ret);

/**
 * 获取指定区域的 sector 信息 (如果启用 MCUBOOT_USE_FLASH_AREA_GET_SECTORS)
 */
#ifdef MCUBOOT_USE_FLASH_AREA_GET_SECTORS
int flash_area_get_sectors(int idx, uint32_t *cnt,
                           struct flash_sector *sectors);
#endif

#endif /* __FLASH_MAP_BACKEND_H__ */
```

---

## 12. Flash 驱动 API 实现

### 12.1 完整的 flash_map 实现示例 (STM32F4)

```c
// flash_map.c
#include <string.h>
#include <assert.h>
#include "flash_map_backend/flash_map_backend.h"
#include "sysflash/sysflash.h"
#include "mcuboot_config/mcuboot_config.h"
#include "hal/flash_drv.h"  // 你的 Flash HAL 驱动

/* ──── Flash 区域静态定义 ──── */

static const struct flash_area bootloader_area = {
    .fa_id        = FLASH_AREA_BOOTLOADER,
    .fa_device_id = FLASH_DEVICE_INTERNAL_FLASH,
    .fa_off       = FLASH_BASE,
    .fa_size      = BOOTLOADER_SIZE,
};

static const struct flash_area primary_slot = {
    .fa_id        = FLASH_AREA_IMAGE_PRIMARY(0),
    .fa_device_id = FLASH_DEVICE_INTERNAL_FLASH,
    .fa_off       = FLASH_BASE + BOOTLOADER_SIZE,
    .fa_size      = SLOT_SIZE,
};

static const struct flash_area secondary_slot = {
    .fa_id        = FLASH_AREA_IMAGE_SECONDARY(0),
    .fa_device_id = FLASH_DEVICE_INTERNAL_FLASH,
    .fa_off       = FLASH_BASE + BOOTLOADER_SIZE + SLOT_SIZE,
    .fa_size      = SLOT_SIZE,
};

#ifdef MCUBOOT_SWAP_USING_SCRATCH
static const struct flash_area scratch_area = {
    .fa_id        = FLASH_AREA_IMAGE_SCRATCH,
    .fa_device_id = FLASH_DEVICE_INTERNAL_FLASH,
    .fa_off       = FLASH_BASE + BOOTLOADER_SIZE + SLOT_SIZE * 2,
    .fa_size      = SCRATCH_SIZE,
};
#endif

/* 区域指针数组 (以 NULL 结尾) */
static const struct flash_area * const flash_areas[] = {
    &bootloader_area,
    &primary_slot,
    &secondary_slot,
#ifdef MCUBOOT_SWAP_USING_SCRATCH
    &scratch_area,
#endif
    NULL
};

/* 引用计数 (解决嵌套打开/关闭问题) */
#define MAX_FLASH_AREAS  16
static int open_count[MAX_FLASH_AREAS] = {0};

/* ──── API 实现 ──── */

int flash_area_open(uint8_t id, const struct flash_area **fa)
{
    for (int i = 0; flash_areas[i] != NULL; i++) {
        if (flash_areas[i]->fa_id == id) {
            *fa = flash_areas[i];
            open_count[i]++;
            return 0;
        }
    }
    return -1;  /* 未找到 */
}

void flash_area_close(const struct flash_area *fa)
{
    /* 如果是完全基于静态数组的实现, 可以不做任何事 */
    /* 如果动态分配, 在这里递减引用计数并释放 */
    (void)fa;
}

int flash_area_read(const struct flash_area *fa, uint32_t off,
                    void *dst, uint32_t len)
{
    uint32_t addr = fa->fa_off + off;

    /* 内部 Flash: 直接通过内存映射读取 */
    if (fa->fa_device_id == FLASH_DEVICE_INTERNAL_FLASH) {
        memcpy(dst, (const void *)addr, len);
        return 0;
    }
    /* 外部 Flash (如 SPI Flash): 调用相应驱动 */
    // if (fa->fa_device_id == FLASH_DEVICE_EXTERNAL_FLASH) {
    //     return ext_flash_read(addr, dst, len);
    // }
    return -1;
}

int flash_area_write(const struct flash_area *fa, uint32_t off,
                     const void *src, uint32_t len)
{
    uint32_t addr = fa->fa_off + off;
    const uint32_t align = flash_area_align(fa);

    if ((off % align) != 0 || (len % align) != 0) {
        return -1;  /* 未对齐 */
    }

    if (fa->fa_device_id == FLASH_DEVICE_INTERNAL_FLASH) {
        return hal_flash_program(addr, (const uint8_t *)src, len);
    }
    return -1;
}

int flash_area_erase(const struct flash_area *fa, uint32_t off,
                     uint32_t len)
{
    uint32_t addr = fa->fa_off + off;
    const uint32_t sector_size = hal_flash_get_sector_size(addr);

    if ((off % sector_size) != 0 || (len % sector_size) != 0) {
        return -1;  /* 未对齐到 sector */
    }

    if (fa->fa_device_id == FLASH_DEVICE_INTERNAL_FLASH) {
        return hal_flash_erase(addr, len);
    }
    return -1;
}

uint32_t flash_area_align(const struct flash_area *fa)
{
    (void)fa;
    return hal_flash_get_write_alignment();
}

uint8_t flash_area_erased_val(const struct flash_area *fa)
{
    (void)fa;
    return 0xff;  /* STM32 Flash 擦除后为 0xFF */
}

int flash_area_id_from_multi_image_slot(int image_index, int slot)
{
    switch (slot) {
    case 0: return FLASH_AREA_IMAGE_PRIMARY(image_index);
    case 1: return FLASH_AREA_IMAGE_SECONDARY(image_index);
    case 2: return FLASH_AREA_IMAGE_SCRATCH;
    default: return -1;
    }
}

int flash_area_id_to_multi_image_slot(int image_index, int area_id)
{
    if (area_id == FLASH_AREA_IMAGE_PRIMARY(image_index))  return 0;
    if (area_id == FLASH_AREA_IMAGE_SECONDARY(image_index)) return 1;
    return -1;
}

int flash_device_base(uint8_t fd_id, uintptr_t *ret)
{
    if (fd_id == FLASH_DEVICE_INTERNAL_FLASH) {
        *ret = FLASH_BASE;
        return 0;
    }
    return -1;
}

#ifdef MCUBOOT_USE_FLASH_AREA_GET_SECTORS
int flash_area_get_sectors(int idx, uint32_t *cnt,
                           struct flash_sector *sectors)
{
    const struct flash_area *fa = NULL;

    if (flash_area_open((uint8_t)idx, &fa) != 0) {
        return -1;
    }

    uint32_t sector_size = hal_flash_get_sector_size(fa->fa_off);
    uint32_t sector_count = fa->fa_size / sector_size;

    if (sector_count > *cnt) {
        return -1;
    }

    *cnt = sector_count;
    for (uint32_t i = 0; i < sector_count; i++) {
        sectors[i].fs_off  = fa->fa_off + i * sector_size;
        sectors[i].fs_size = sector_size;
    }
    return 0;
}
#endif
```

### 12.2 Flash HAL 驱动接口示例 (STM32 HAL)

```c
// hal/flash_drv.h
#ifndef HAL_FLASH_DRV_H__
#define HAL_FLASH_DRV_H__

#include <stdint.h>

/**
 * 初始化 Flash 外设
 */
int hal_flash_init(void);

/**
 * 写 Flash
 * @param addr  绝对地址
 * @param data  数据缓冲区
 * @param len   长度 (必须是 write_alignment 的倍数)
 */
int hal_flash_program(uint32_t addr, const uint8_t *data, uint32_t len);

/**
 * 擦除 Flash sector
 * @param addr  绝对地址
 * @param len   长度 (必须是 sector_size 的倍数)
 */
int hal_flash_erase(uint32_t addr, uint32_t len);

/**
 * 获取最小写入对齐
 */
uint32_t hal_flash_get_write_alignment(void);

/**
 * 获取 sector 大小 (给定地址所在的 sector)
 */
uint32_t hal_flash_get_sector_size(uint32_t addr);

#endif
```

```c
// hal/flash_drv.c (STM32F4 HAL 实现)
#include "stm32f4xx_hal.h"
#include "hal/flash_drv.h"

int hal_flash_init(void)
{
    /* STM32 HAL 初始化时已配置 Flash */
    return 0;
}

int hal_flash_program(uint32_t addr, const uint8_t *data, uint32_t len)
{
    HAL_FLASH_Unlock();

    /* STM32F4 Flash 按字 (32-bit) 编程 */
    uint32_t words = len / 4;
    uint32_t *src  = (uint32_t *)data;
    uint32_t dst   = addr;

    for (uint32_t i = 0; i < words; i++) {
        if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, dst, *src) != HAL_OK) {
            HAL_FLASH_Lock();
            return -1;
        }
        dst += 4;
        src += 1;
    }

    HAL_FLASH_Lock();
    return 0;
}

int hal_flash_erase(uint32_t addr, uint32_t len)
{
    HAL_FLASH_Unlock();

    FLASH_EraseInitTypeDef erase_cfg = {0};
    uint32_t sector_error = 0;

    /* 计算起始 sector 和要擦除的 sector 数量 */
    erase_cfg.TypeErase = FLASH_TYPEERASE_SECTORS;
    erase_cfg.Sector    = hal_flash_addr_to_sector(addr);
    erase_cfg.NbSectors = len / hal_flash_get_sector_size(addr);
    erase_cfg.VoltageRange = FLASH_VOLTAGE_RANGE_3;

    if (HAL_FLASHEx_Erase(&erase_cfg, &sector_error) != HAL_OK) {
        HAL_FLASH_Lock();
        return -1;
    }

    HAL_FLASH_Lock();
    return 0;
}

uint32_t hal_flash_get_write_alignment(void)
{
    return 4;  /* STM32F4: 最小写入单位是 32-bit (4 字节) */
}

uint32_t hal_flash_get_sector_size(uint32_t addr)
{
    /* STM32F4: 前 4 个 sector 16KB, 第5个 64KB, 其余 128KB */
    uint32_t offset = addr - FLASH_BASE;
    if (offset < 0x10000)      return 0x4000;   /* 16KB */
    else if (offset < 0x20000) return 0x10000;  /* 64KB */
    else                       return 0x20000;  /* 128KB */
}
```

---

## 13. 启动入口与跳转

### 13.1 公钥定义 (keys.c)

```c
// keys.c
#include <bootutil/sign_key.h>

/* 你的公钥 (由 imgtool getpub 生成) */
static const unsigned char ecdsa_pub_key[] = {
    0x30, 0x59, 0x30, 0x13, /* ... 你的 DER 格式公钥 ... */
};
static const unsigned int ecdsa_pub_key_len = sizeof(ecdsa_pub_key);

#if !defined(MCUBOOT_HW_KEY)
/* 默认模式: 公钥嵌入 bootloader */
const struct bootutil_key bootutil_keys[] = {
    [0] = {
        .key = ecdsa_pub_key,
        .len = &ecdsa_pub_key_len,
    }
};
const int bootutil_key_cnt = sizeof(bootutil_keys) / sizeof(bootutil_keys[0]);
#else
/* 硬件密钥模式: 空数组占位 */
struct bootutil_key bootutil_keys[] = { {0} };
const int bootutil_key_cnt = 1;
#endif
```

### 13.2 启动入口 (main.c)

```c
// main.c
#include <stdint.h>
#include <stddef.h>
#include "bootutil/bootutil.h"
#include "bootutil/image.h"
#include "hal/flash_drv.h"

/* MCUboot 内部使用的全局内存池 (用于 boot_go 的大局部变量) */
/* 见 bootutil_priv.h: TARGET_STATIC 机制 */
static uint8_t boot_buffer[4096] __attribute__((aligned(8)));

/**
 * 跳转到用户应用程序
 *
 * @param rsp boot_go() 返回的启动响应
 */
static void jump_to_app(const struct boot_rsp *rsp)
{
    /* 用户镜像的向量表位于 rsp->br_image_off 偏移处 */
    uint32_t app_vec_base = rsp->br_image_off;

    /* 获取栈指针 (SP) 和复位向量 (PC) */
    uint32_t *vectors = (uint32_t *)app_vec_base;
    uint32_t stack_ptr = vectors[0];   /* 初始 SP */
    uint32_t entry_point = vectors[1]; /* Reset_Handler */

    /* 关闭所有中断 */
    __disable_irq();

    /* 设置 MSP */
    __set_MSP(stack_ptr);

    /* 跳转到用户应用 */
    typedef void (*app_entry_t)(void);
    app_entry_t app_entry = (app_entry_t)entry_point;
    app_entry();

    /* 不应该到达这里 */
    while (1);
}

/**
 * 硬件初始化
 */
static void board_init(void)
{
    /* 配置系统时钟 */
    SystemClock_Config();

    /* 初始化 Flash 外设 */
    hal_flash_init();

    /* 初始化调试串口 (可选) */
    // uart_init();
}

/**
 * 主入口
 */
int main(void)
{
    struct boot_rsp rsp;

    /* 平台初始化 */
    board_init();

    /* 执行 bootloader 核心逻辑 */
    fih_ret rc = boot_go(&rsp);

    /* 根据返回值判断是否跳转 (FIH 宏在非 FIH 模式下即直接比较) */
    if (fih_eq(rc, FIH_SUCCESS)) {
        jump_to_app(&rsp);
    }

    /* boot_go 失败, 进入死循环或恢复模式 */
    while (1) {
        /* 可以在这里添加串口恢复或 LED 错误指示 */
    }
    return 0;
}
```

### 13.3 链接脚本注意事项

MCUboot 和用户应用需要**独立的链接脚本**:

**MCUboot 链接脚本要点:**
```ld
/* mcuboot.ld */
FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 64K  /* Bootloader 区域 */
RAM   (rw) : ORIGIN = 0x20000000, LENGTH = 128K

/* 确保 MCUboot 使用 RAM 的低地址区域, 与应用不冲突 */
```

**用户应用链接脚本要点:**
```ld
/* app.ld */
FLASH (rx) : ORIGIN = 0x08010000, LENGTH = 448K  /* 从 Bootloader 之后开始 */
RAM   (rw) : ORIGIN = 0x20000000, LENGTH = 128K

/* 用户应用向量表必须放在 0x08010000 */
```

**向量表偏移配置 (用户应用启动代码中):**
```c
// 在用户应用的 SystemInit() 或 main() 开头:
SCB->VTOR = FLASH_BASE | 0x10000; // 向量表重定位到 Primary Slot
```

---

## 14. 日志与断言适配

### 14.1 日志适配

如果启用 `MCUBOOT_HAVE_LOGGING`, 需要提供以下宏:

```c
// 在 mcuboot_config.h 或单独的头文件中

/* 日志模块注册 (裸机环境通常为空) */
#define MCUBOOT_LOG_MODULE_REGISTER(domain)   (void)0
#define MCUBOOT_LOG_MODULE_DECLARE(domain)    (void)0

/* 日志输出 — 映射到你的串口输出 */
#define MCUBOOT_LOG_ERR(fmt, ...)   printf("[MCUboot][ERR] " fmt "\r\n", ##__VA_ARGS__)
#define MCUBOOT_LOG_WRN(fmt, ...)   printf("[MCUboot][WRN] " fmt "\r\n", ##__VA_ARGS__)
#define MCUBOOT_LOG_INF(fmt, ...)   printf("[MCUboot][INF] " fmt "\r\n", ##__VA_ARGS__)
#define MCUBOOT_LOG_DBG(fmt, ...)   printf("[MCUboot][DBG] " fmt "\r\n", ##__VA_ARGS__)
```

如果你**不需要日志**, 直接在 `mcuboot_config.h` 中不定义 `MCUBOOT_HAVE_LOGGING` (或设为 0):

```c
// #define MCUBOOT_HAVE_LOGGING 0
```

### 14.2 断言适配

选项 1: 使用标准库 assert (默认):
```c
// 在 mcuboot_config.h 中不定义 MCUBOOT_HAVE_ASSERT_H
// MCUboot 将使用 <assert.h>
```

选项 2: 自定义断言:
```c
// 在 mcuboot_config.h 中:
#define MCUBOOT_HAVE_ASSERT_H

// 创建 mcuboot_config/mcuboot_assert.h:
#ifndef MCUBOOT_ASSERT_H__
#define MCUBOOT_ASSERT_H__

#define ASSERT(expr) do {                                   \
    if (!(expr)) {                                          \
        printf("ASSERT FAILED: %s, line %d\r\n",            \
               __FILE__, __LINE__);                         \
        while (1) { /* 停机 */ }                            \
    }                                                       \
} while (0)

#endif
```

---

## 15. 看门狗与空闲处理

### 15.1 看门狗喂狗

如果 Swap 操作时间较长, 可能触发硬件看门狗复位, 需要在 Swap 期间喂狗:

```c
// 在 mcuboot_config.h 中:
#define MCUBOOT_WATCHDOG_FEED()   \
    do {                           \
        HAL_IWDG_Refresh(&hiwdg);  \  /* 调用平台的喂狗函数 */
    } while (0)
```

### 15.2 CPU 空闲

在等待 Flash 操作完成时, MCUboot 会调用 `MCUBOOT_CPU_IDLE()`:

```c
// 裸机环境通常为空操作:
#define MCUBOOT_CPU_IDLE()  \
    do {                     \
        __WFI();             \  /* 或使用 WFI 降低功耗 */
    } while (0)
```

---

## 16. 完整裸机移植示例 (STM32F4)

### 16.1 目录结构

```
my_mcuboot_port/stm32f4/
├── CMakeLists.txt
├── mcuboot_config/
│   └── mcuboot_config.h
├── flash_map_backend/
│   └── flash_map_backend.h
├── sysflash/
│   └── sysflash.h
├── hal/
│   ├── flash_drv.h
│   └── flash_drv.c
├── src/
│   ├── main.c
│   └── keys.c
├── lib/                       ← Mbed TLS 编译产物或源码
│   ├── libmbedtls.a
│   └── include/
├── mcuboot_src/               ← 指向 mcuboot 源码的符号链接或拷贝
│   ├── bootutil/
│   └── ...
├── stm32f407vg_flash.ld       ← Linker script
└── Makefile
```

### 16.2 完整的 mcuboot_config.h (裸机, Overwrite Only + ECDSA)

```c
#ifndef __MCUBOOT_CONFIG_H__
#define __MCUBOOT_CONFIG_H__

/* ══════════════════════════════════════════════════
 * 签名类型: ECDSA-P256
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_SIGN_EC256

/* ══════════════════════════════════════════════════
 * 升级策略: 覆盖模式 (最简单)
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_OVERWRITE_ONLY

/* ══════════════════════════════════════════════════
 * 加密后端: Mbed TLS
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_USE_MBED_TLS

/* ══════════════════════════════════════════════════
 * 安全设置
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_VALIDATE_PRIMARY_SLOT   /* 每次启动验证主槽 */

/* ══════════════════════════════════════════════════
 * Flash 与内存
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_MAX_IMG_SECTORS   128
#define MCUBOOT_IMAGE_NUMBER      1

/* ══════════════════════════════════════════════════
 * 日志
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_HAVE_LOGGING  1

/* 日志函数 (裸机: 映射到 printf 或串口) */
#define MCUBOOT_LOG_MODULE_REGISTER(domain)  (void)0
#define MCUBOOT_LOG_MODULE_DECLARE(domain)   (void)0
#define MCUBOOT_LOG_ERR(_fmt, ...)  printf("[E] " _fmt "\r\n", ##__VA_ARGS__)
#define MCUBOOT_LOG_WRN(_fmt, ...)  printf("[W] " _fmt "\r\n", ##__VA_ARGS__)
#define MCUBOOT_LOG_INF(_fmt, ...)  printf("[I] " _fmt "\r\n", ##__VA_ARGS__)
#define MCUBOOT_LOG_DBG(_fmt, ...)  /* 调试日志: 关闭 */

/* 看门狗喂狗 (可选) */
// #define MCUBOOT_WATCHDOG_FEED()  HAL_IWDG_Refresh(&hiwdg)

/* CPU 空闲 */
#define MCUBOOT_CPU_IDLE()  __WFI()

/* Assert (可选: 自定义) */
#define MCUBOOT_HAVE_ASSERT_H

#endif /* __MCUBOOT_CONFIG_H__ */
```

### 16.3 构建系统示例 (CMake)

```cmake
cmake_minimum_required(VERSION 3.16)
project(mcuboot_stm32f4 C ASM)

set(CMAKE_C_STANDARD 99)

# MCUboot 核心源码
set(MCUBOOT_ROOT ${CMAKE_SOURCE_DIR}/../..)   # 指向 mcuboot 仓库根目录
set(BOOTUTIL_SRC
    ${MCUBOOT_ROOT}/boot/bootutil/src/loader.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/bootutil_misc.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/image_validate.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/image_ecdsa.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/bootutil_img_hash.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/tlv.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/caps.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/bootutil_public.c
    ${MCUBOOT_ROOT}/boot/bootutil/src/bootutil_find_key.c
    # 注意: Overwrite Only 模式不需要 swap_*.c
)

# 你的移植代码
set(PORT_SRC
    src/main.c
    src/keys.c
    hal/flash_drv.c
)

# Mbed TLS 库
set(MBEDTLS_LIB ${CMAKE_SOURCE_DIR}/lib/libmbedtls.a)

add_executable(mcuboot.elf
    ${BOOTUTIL_SRC}
    ${PORT_SRC}
)

target_include_directories(mcuboot.elf PRIVATE
    ${MCUBOOT_ROOT}/boot/bootutil/include
    ${MCUBOOT_ROOT}/boot/bootutil/src
    .
    hal
    lib/include          # Mbed TLS 头文件
)

target_link_libraries(mcuboot.elf
    ${MBEDTLS_LIB}
)

# 链接脚本
target_link_options(mcuboot.elf PRIVATE
    -T ${CMAKE_SOURCE_DIR}/stm32f407vg_flash.ld
)

# 生成 bin 文件
add_custom_command(TARGET mcuboot.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary mcuboot.elf mcuboot.bin
)

# 大小报告
add_custom_command(TARGET mcuboot.elf POST_BUILD
    COMMAND ${CMAKE_SIZE} mcuboot.elf
)
```

---

---

# 第二篇: 第三方算法移植指南

## 17. 加密子系统架构

### 17.1 抽象层架构

MCUboot 的加密子系统通过**编译时多态**实现算法替换:

```
┌──────────────────────────────────────┐
│          MCUboot 核心代码             │
│  (bootutil_img_hash, image_ecdsa...)│
│         调用抽象 API                  │
└──────────────┬───────────────────────┘
               │
       ┌───────┴───────┐
       │   加密抽象头    │
       │  (sha.h,       │
       │   ecdsa.h,     │
       │   aes_ctr.h...)│
       └───────┬───────┘
               │
   ┌───────────┼───────────┐
   ▼           ▼           ▼
┌──────┐ ┌──────┐  ┌──────────┐
│ Mbed │ │ Tiny │  │ CUSTOM   │  ← 你在这里
│ TLS  │ │crypt │  │ CRYPTO   │
└──────┘ └──────┘  └──────────┘
```

### 17.2 如何选择自定义后端

在 `mcuboot_config.h` 中定义:

```c
#define MCUBOOT_USE_CUSTOM_CRYPTO
```

**注意:** 只能定义一个加密后端宏, 不能同时定义 `MCUBOOT_USE_MBED_TLS` 或 `MCUBOOT_USE_TINYCRYPT`。

### 17.3 自定义后端的核心原则

当 `MCUBOOT_USE_CUSTOM_CRYPTO` 被定义时, MCUboot 的加密抽象头 (`sha.h`, `ecdsa.h` 等) **不会**定义任何类型和函数。你必须通过**在它们之前**包含的自定义头文件来提供所有必要的定义。

**关键: 包含顺序**
```
编译器 → 强制包含 mcuboot_custom_crypto.h  →  然后包含 MCUboot 源码
         (定义所有类型和函数)                   (使用你定义的类型和函数)
```

---

## 18. 自定义加密后端 (MCUBOOT_USE_CUSTOM_CRYPTO)

### 18.1 启用自定义后端

```c
// mcuboot_config.h
#define MCUBOOT_USE_CUSTOM_CRYPTO

// 如果你的自定义后端头文件为 mcuboot_custom_crypto.h:
// 确保编译器通过 -include 或在此处包含它
// #include "mcuboot_custom_crypto.h"  ← 必须在所有 MCUboot 源码之前包含
```

**推荐方法:** 使用 GCC 的强制包含:

```makefile
CFLAGS += -include mcuboot_custom_crypto.h
```

或在 `mcuboot_config.h` 的**第一行**包含它:

```c
// mcuboot_config.h (必须在最前面)
#include "mcuboot_custom_crypto.h"

// 然后才是其他配置...
#define MCUBOOT_SIGN_EC256
// ...
```

### 18.2 自定义后端的最小需求

根据你的 MCUboot 功能配置, 需要提供对应的一组类型和函数:

| 功能 | 需要提供的接口 |
|------|---------------|
| 镜像 Hash (总是需要) | `bootutil_sha_*` |
| 签名验证 | `bootutil_ecdsa_*` |
| 镜像加密 | `boot_enc_*` + `bootutil_hmac_sha256_*` + `bootutil_aes_ctr_*` |

---

## 19. SHA-256 实现适配

### 19.1 接口定义

无论你使用哪种 SHA 算法 (SHA-256/SHA-384/SHA-512), MCUboot 使用统一的 `bootutil_sha_*` 接口:

```c
/* 上下文类型 — 必须是完整的结构体定义 (不能是指针) */
typedef struct {
    /* 你的 SHA 库的上下文 */
    uint8_t  buffer[64];
    uint32_t state[8];
    uint64_t count;
    /* ... */
} bootutil_sha_context;

/* 初始化 */
static inline int bootutil_sha_init(bootutil_sha_context *ctx)
{
    /* 调用你的 SHA 库初始化 */
    return your_sha256_init(ctx);
}

/* 释放/清零 */
static inline int bootutil_sha_drop(bootutil_sha_context *ctx)
{
    /* 清零上下文 (安全擦除) */
    memset(ctx, 0, sizeof(*ctx));
    return 0;
}

/* 更新 hash */
static inline int bootutil_sha_update(
    bootutil_sha_context *ctx,
    const void *data,
    uint32_t data_len)
{
    return your_sha256_update(ctx, data, data_len);
}

/* 完成 hash, 输出到 output (32 字节用于 SHA-256) */
static inline int bootutil_sha_finish(
    bootutil_sha_context *ctx,
    uint8_t *output)
{
    return your_sha256_final(ctx, output);
}
```

**重要规范:**
- 所有函数必须返回 `int` (0 = 成功, 非0 = 失败)
- `bootutil_sha_init/drop` 必须总是成功 (返回 0) 或你的硬件初始化可能失败
- 函数必须是 `static inline` 或外部链接 — 推荐 `static inline` 以避免链接冲突

### 19.2 基于硬件加速 SHA-256 的完整示例

```c
// my_sha.h
#ifndef MY_SHA_H__
#define MY_SHA_H__

#include <stdint.h>
#include <string.h>

/* 假设 MCU 有硬件 SHA-256 加速器 */
#include "my_mcu_hal.h"  /* HAL_HASH_xxx() */

typedef struct {
    uint8_t  busy;        /* 硬件忙标志 */
    uint32_t reserved;    /* 保留 */
} bootutil_sha_context;

static inline int bootutil_sha_init(bootutil_sha_context *ctx)
{
    (void)ctx;
    HAL_HASH_Reset(HASH_ALG_SHA256);
    return 0;
}

static inline int bootutil_sha_drop(bootutil_sha_context *ctx)
{
    memset(ctx, 0, sizeof(*ctx));
    HAL_HASH_Reset(HASH_ALG_SHA256);
    return 0;
}

static inline int bootutil_sha_update(
    bootutil_sha_context *ctx,
    const void *data,
    uint32_t data_len)
{
    (void)ctx;
    /* 硬件 SHA 通常按块输入 */
    return HAL_HASH_Update(data, data_len);
}

static inline int bootutil_sha_finish(
    bootutil_sha_context *ctx,
    uint8_t *output)
{
    (void)ctx;
    return HAL_HASH_Final(output);  /* 输出 32 字节 SHA-256 */
}

#endif /* MY_SHA_H__ */
```

---

## 20. ECDSA 签名验证适配

### 20.1 接口定义

```c
/* ECDSA 上下文类型 */
typedef struct {
    /* 你的 ECDSA 库的上下文或硬件引擎状态 */
    uint8_t  initialized;
    /* ... */
} bootutil_ecdsa_context;

/* 初始化 */
static inline void bootutil_ecdsa_init(bootutil_ecdsa_context *ctx)
{
    memset(ctx, 0, sizeof(*ctx));
}

/* 释放 */
static inline void bootutil_ecdsa_drop(bootutil_ecdsa_context *ctx)
{
    memset(ctx, 0, sizeof(*ctx));
}

/**
 * 解析 DER 编码的公钥并加载到上下文
 *
 * @param ctx  ECDSA 上下文
 * @param cp   [输入/输出] 指向公钥数据的指针 (会被推进)
 * @param end  公钥数据结束位置
 * @return 0 成功, 非0 失败
 */
static inline int bootutil_ecdsa_parse_public_key(
    bootutil_ecdsa_context *ctx,
    uint8_t **cp,
    uint8_t *end)
{
    /* 解析 SubjectPublicKeyInfo DER 格式 */
    /* MCUboot 传入的公钥格式: DER 编码的 SubjectPublicKeyInfo */
    /* 对于 P-256: 包含 OID + 65 字节未压缩公钥 (0x04 || x || y) */

    /* 简单实现: 跳过 DER 包装, 直接获取公钥坐标 */
    /* 实际实现需要解析 ASN.1 DER 结构 */

    /* 这是需要你根据 ECDSA 库实现的解析逻辑 */
    return your_ecdsa_parse_spki(ctx, cp, end);
}

/**
 * 验证 ECDSA 签名
 *
 * @param ctx       ECDSA 上下文 (已加载公钥)
 * @param pk        公钥原始字节 (备用, 可能为 NULL)
 * @param pk_len    公钥长度
 * @param hash      hash 值
 * @param hash_len  hash 长度
 * @param sig       签名值
 * @param sig_len   签名长度
 * @return 0 验证通过, 非0 验证失败
 */
static inline int bootutil_ecdsa_verify(
    bootutil_ecdsa_context *ctx,
    uint8_t *pk,
    size_t pk_len,
    uint8_t *hash,
    size_t hash_len,
    uint8_t *sig,
    size_t sig_len)
{
    return your_ecdsa_verify(ctx, hash, hash_len, sig, sig_len);
}
```

### 20.2 基于 Mbed TLS 的 ECDSA 实现参考

```c
// my_ecdsa_mbedtls.h (如果仍使用 Mbed TLS 但要做自定义包装)
#include "mbedtls/ecdsa.h"
#include "mbedtls/asn1.h"

typedef mbedtls_ecdsa_context bootutil_ecdsa_context;

static inline void bootutil_ecdsa_init(bootutil_ecdsa_context *ctx)
{
    mbedtls_ecdsa_init(ctx);
}

static inline void bootutil_ecdsa_drop(bootutil_ecdsa_context *ctx)
{
    mbedtls_ecdsa_free(ctx);
}

static inline int bootutil_ecdsa_parse_public_key(
    bootutil_ecdsa_context *ctx,
    uint8_t **cp,
    uint8_t *end)
{
    /* 解析 DER SubjectPublicKeyInfo */
    int rc;
    size_t len;

    /* 读取 SEQUENCE */
    rc = mbedtls_asn1_get_tag(cp, end, &len,
        MBEDTLS_ASN1_CONSTRUCTED | MBEDTLS_ASN1_SEQUENCE);
    if (rc != 0) return rc;

    end = *cp + len;

    /* 读取 AlgorithmIdentifier SEQUENCE */
    rc = mbedtls_asn1_get_tag(cp, end, &len,
        MBEDTLS_ASN1_CONSTRUCTED | MBEDTLS_ASN1_SEQUENCE);
    if (rc != 0) return rc;

    *cp += len;  /* 跳过 AlgorithmIdentifier */

    /* 读取 BIT STRING (公钥位串) */
    rc = mbedtls_asn1_get_bitstring_null(cp, end, &len);
    if (rc != 0) return rc;

    /* 现在 *cp 指向原始的 65 字节未压缩公钥 (P-256) */
    /* 跳过第一个字节 (0x04 表示未压缩) 或直接解析 */
    return mbedtls_ecp_point_read_binary(
        &ctx->MBEDTLS_PRIVATE(grp),
        &ctx->MBEDTLS_PRIVATE(Q),
        *cp, len);
}

static inline int bootutil_ecdsa_verify(
    bootutil_ecdsa_context *ctx,
    uint8_t *pk, size_t pk_len,
    uint8_t *hash, size_t hash_len,
    uint8_t *sig, size_t sig_len)
{
    (void)pk;
    (void)pk_len;
    return mbedtls_ecdsa_read_signature(
        ctx, hash, hash_len, sig, sig_len);
}
```

---

## 21. 镜像加密相关适配

### 21.1 概述

如果启用了 `MCUBOOT_ENC_IMAGES` 且使用自定义加密后端, 你需要提供加密相关的所有接口, 因为 MCUboot 的 `encrypted.c` 在 `MCUBOOT_USE_CUSTOM_CRYPTO` 时会被跳过编译。

### 21.2 必须实现的加密入口函数

```c
// my_enc.h / my_enc.c

#include "bootutil/image.h"
#include "bootutil/enc_key.h"

/**
 * 解密 TLV 中的加密密钥, 获得 AES 密钥
 * @param buf    TLV 数据指针
 * @param enckey 输出: 解密后的 AES 密钥 (16 或 32 字节)
 * @return 0 成功
 */
int boot_decrypt_key(const uint8_t *buf, uint8_t *enckey);

/**
 * 初始化加密状态
 */
int boot_enc_init(struct enc_key_data *enc_state);

/**
 * 销毁加密状态 (安全擦除密钥)
 */
int boot_enc_drop(struct enc_key_data *enc_state);

/**
 * 设置 AES 密钥
 * @param enc_state  加密状态
 * @param key        AES 密钥 (16 或 32 字节)
 */
int boot_enc_set_key(struct enc_key_data *enc_state, const uint8_t *key);

/**
 * 从 Flash 加载加密密钥
 * @param state  bootloader 状态
 * @param slot   槽编号
 * @param hdr    镜像头
 * @param fap    Flash 区域
 * @param bs     Boot 状态
 */
int boot_enc_load(struct boot_loader_state *state, int slot,
                  const struct image_header *hdr,
                  const struct flash_area *fap,
                  struct boot_status *bs);

/**
 * 检查加密状态是否有效
 */
bool boot_enc_valid(const struct enc_key_data *enc_state);

/**
 * AES-CTR 加密一块数据
 */
void boot_enc_encrypt(struct enc_key_data *enc_state,
                      uint32_t off, uint32_t sz, uint32_t blk_off,
                      uint8_t *buf);

/**
 * AES-CTR 解密一块数据 (与加密相同, CTR 模式)
 */
void boot_enc_decrypt(struct enc_key_data *enc_state,
                      uint32_t off, uint32_t sz, uint32_t blk_off,
                      uint8_t *buf);

/**
 * 清零加密状态中的所有密钥
 */
void boot_enc_zeroize(struct enc_key_data *enc_state);
```

### 21.3 辅助接口 (HMAC-SHA256)

```c
typedef struct {
    /* 你的 HMAC-SHA256 上下文 */
} bootutil_hmac_sha256_context;

static inline void bootutil_hmac_sha256_init(
    bootutil_hmac_sha256_context *ctx);
static inline void bootutil_hmac_sha256_drop(
    bootutil_hmac_sha256_context *ctx);
static inline int bootutil_hmac_sha256_set_key(
    bootutil_hmac_sha256_context *ctx,
    const uint8_t *key, uint32_t key_len);
static inline int bootutil_hmac_sha256_update(
    bootutil_hmac_sha256_context *ctx,
    const void *data, uint32_t data_len);
static inline int bootutil_hmac_sha256_finish(
    bootutil_hmac_sha256_context *ctx,
    uint8_t *output, uint32_t output_len);
```

### 21.4 辅助接口 (AES-CTR)

```c
#define BOOT_ENC_BLOCK_SIZE 16  /* 必须为 16 */

typedef struct {
    /* 你的 AES-CTR 上下文 (密钥已设置) */
} bootutil_aes_ctr_context;

static inline void bootutil_aes_ctr_init(bootutil_aes_ctr_context *ctx);
static inline void bootutil_aes_ctr_drop(bootutil_aes_ctr_context *ctx);
static inline int bootutil_aes_ctr_set_key(
    bootutil_aes_ctr_context *ctx, const uint8_t *key);

/**
 * AES-CTR 加密
 * @param counter  16 字节计数器
 * @param m        明文
 * @param mlen     明文长度
 * @param blk_off  块内偏移 (用于处理非对齐)
 * @param output   密文输出
 */
static inline int bootutil_aes_ctr_encrypt(
    bootutil_aes_ctr_context *ctx, uint8_t *counter,
    const uint8_t *m, uint32_t mlen,
    size_t blk_off, uint8_t *output);

/* decrypt 通常与 encrypt 相同 (CTR 模式对称) */
static inline int bootutil_aes_ctr_decrypt(
    bootutil_aes_ctr_context *ctx, uint8_t *counter,
    const uint8_t *c, uint32_t clen,
    size_t blk_off, uint8_t *output);
```

### 21.5 辅助接口 (ECDH-P256, 用于 ECIES)

```c
typedef struct { /* ... */ } bootutil_ecdh_p256_context;
/* bootutil_key_exchange_ctx 是 bootutil_ecdh_p256_context 的别名 */
typedef bootutil_ecdh_p256_context bootutil_key_exchange_ctx;

static inline void bootutil_ecdh_p256_init(bootutil_ecdh_p256_context *ctx);
static inline void bootutil_ecdh_p256_drop(bootutil_ecdh_p256_context *ctx);

/**
 * 计算 ECDH 共享密钥
 * @param pk  对方公钥 (65 字节, 未压缩格式)
 * @param sk  己方私钥 (32 字节)
 * @param z   输出: 共享密钥 (32 字节)
 */
static inline int bootutil_ecdh_p256_shared_secret(
    bootutil_ecdh_p256_context *ctx,
    const uint8_t *pk, const uint8_t *sk, uint8_t *z);
```

---

## 22. 完整自定义加密后端示例 (基于硬件加速)

### 22.1 使用硬件 Crypto 引擎的完整示例

假设目标 MCU 有硬件 SHA-256、AES-CTR、ECDSA 加速器。

```c
// mcuboot_custom_crypto.h
#ifndef MCUBOOT_CUSTOM_CRYPTO_H__
#define MCUBOOT_CUSTOM_CRYPTO_H__

#include "hw_crypto_sha.h"       /* 硬件 SHA-256 */
#include "hw_crypto_ecdsa.h"     /* 硬件 ECDSA-P256 */
#if defined(MCUBOOT_ENC_IMAGES)
#include "hw_crypto_aes.h"       /* 硬件 AES-CTR */
#include "hw_crypto_hmac.h"      /* 硬件 HMAC-SHA256 */
#include "hw_crypto_ecdh.h"      /* 硬件 ECDH-P256 */
#endif

#endif /* MCUBOOT_CUSTOM_CRYPTO_H__ */
```

```c
// hw_crypto_sha.h — 硬件 SHA-256
#ifndef HW_CRYPTO_SHA_H__
#define HW_CRYPTO_SHA_H__

#include <stdint.h>
#include <string.h>
#include "stm32h7xx_hal.h"  /* 假设 STM32H7 HASH 外设 */

typedef struct {
    HASH_HandleTypeDef hhash;    /* STM32 HAL HASH 句柄 */
    uint8_t  started;
} bootutil_sha_context;

static inline int bootutil_sha_init(bootutil_sha_context *ctx)
{
    ctx->hhash.Instance = HASH;
    if (HAL_HASH_DeInit(&ctx->hhash) != HAL_OK) return -1;
    ctx->hhash.Init.DataType = HASH_DATATYPE_8B;
    if (HAL_HASH_Init(&ctx->hhash) != HAL_OK) return -1;
    return 0;
}

static inline int bootutil_sha_drop(bootutil_sha_context *ctx)
{
    HAL_HASH_DeInit(&ctx->hhash);
    memset(ctx, 0, sizeof(*ctx));
    return 0;
}

static inline int bootutil_sha_update(
    bootutil_sha_context *ctx,
    const void *data, uint32_t data_len)
{
    return (HAL_HASH_Accumulate(&ctx->hhash,
            (uint8_t *)data, data_len) == HAL_OK) ? 0 : -1;
}

static inline int bootutil_sha_finish(
    bootutil_sha_context *ctx, uint8_t *output)
{
    uint8_t hash[32];
    HAL_StatusTypeDef status = HAL_HASH_Accumulate(
        &ctx->hhash, NULL, 0);  /* 触发最终计算 */
    if (status != HAL_OK) return -1;
    HAL_HASH_GetHash(&ctx->hhash, hash);
    memcpy(output, hash, 32);
    return 0;
}

#endif /* HW_CRYPTO_SHA_H__ */
```

```c
// hw_crypto_ecdsa.h — 硬件 ECDSA-P256
#ifndef HW_CRYPTO_ECDSA_H__
#define HW_CRYPTO_ECDSA_H__

/* 假设使用 STM32 PKA (Public Key Accelerator) 或软件实现 */
#include "micro_ecc.h"  /* 或任何你选择的 ECDSA 库 */

typedef struct {
    uECC_Curve curve;
    uint8_t    public_key[64];   /* 32 + 32 */
    int        has_key;
} bootutil_ecdsa_context;

static inline void bootutil_ecdsa_init(bootutil_ecdsa_context *ctx)
{
    memset(ctx, 0, sizeof(*ctx));
    ctx->curve = uECC_secp256r1();
}

static inline void bootutil_ecdsa_drop(bootutil_ecdsa_context *ctx)
{
    memset(ctx, 0, sizeof(*ctx));
}

static inline int bootutil_ecdsa_parse_public_key(
    bootutil_ecdsa_context *ctx,
    uint8_t **cp, uint8_t *end)
{
    /* 解析 DER SubjectPublicKeyInfo 提取 64 字节坐标 */
    int rc = parse_der_spki_p256(cp, end, ctx->public_key);
    ctx->has_key = (rc == 0);
    return rc;
}

static inline int bootutil_ecdsa_verify(
    bootutil_ecdsa_context *ctx,
    uint8_t *pk, size_t pk_len,
    uint8_t *hash, size_t hash_len,
    uint8_t *sig, size_t sig_len)
{
    (void)pk;
    (void)pk_len;
    if (!ctx->has_key) return -1;
    return uECC_verify(ctx->public_key, hash, hash_len,
                       sig, ctx->curve) ? 0 : -1;
}

#endif /* HW_CRYPTO_ECDSA_H__ */
```

### 22.2 构建配置

```makefile
# Makefile 或 CMakeLists.txt 中:

# 1. 定义自定义加密宏
CFLAGS += -DMCUBOOT_USE_CUSTOM_CRYPTO

# 2. 强制包含自定义头 (确保在所有 MCUboot 源码之前)
CFLAGS += -include mcuboot_custom_crypto.h

# 3. 添加自定义加密实现的路径
CFLAGS += -I./crypto

# 4. 链接你的加密库
LDFLAGS += -L./crypto/lib -lmy_crypto

# 5. 如果需要加密功能, 编译自定义加密实现
ifeq ($(MCUBOOT_ENC_IMAGES),1)
    SRC += crypto/my_enc.c
endif
```

---

---

# 第三篇: 裁剪移植指南

## 23. 功能裁剪全景

MCUboot 通过编译宏实现**编译时功能裁剪**。每个 `#define MCUBOOT_*` 都会引入或移除相应的代码路径。合理的裁剪可以显著减小代码体积。

### 23.1 裁剪维度

| 维度 | 裁剪前 (典型) | 裁剪后 (最小) | 节省 |
|------|--------------|--------------|------|
| Swap 模式 → Overwrite Only | ~20KB | ~12KB | ~8KB |
| RSA → ECDSA-P256 | ~18KB | ~12KB | ~6KB |
| ECDSA → 无签名 | ~12KB | ~4KB | ~8KB |
| Mbed TLS → Tinycrypt | ~35KB | ~18KB | ~17KB |
| 有加密 → 无加密 | ~25KB | ~12KB | ~13KB |
| 有日志 → 无日志 | ~13KB | ~10KB | ~3KB |
| 多镜像 → 单镜像 | ~14KB | ~12KB | ~2KB |
| 最大 sector=128 → 32 | RAM: ~2KB | RAM: ~512B | ~1.5KB RAM |

**极端精简目标:** ~4-6KB Flash, ~1-2KB RAM (无签名验证, Overwrite Only)

---

## 24. 升级策略裁剪

### 24.1 Overwrite Only (最简模式)

这是最小代码量的升级策略:

```c
// mcuboot_config.h
#define MCUBOOT_OVERWRITE_ONLY

// 可选: 快速覆盖 (仅擦写需要的 sector, 减少 Flash 磨损)
#define MCUBOOT_OVERWRITE_ONLY_FAST
```

**裁剪效果:**
- 移除所有 swap 相关代码 (swap_scratch.c, swap_move.c, swap_offset.c)
- 移除 swap 状态管理
- 移除 scratch 分区需求
- 移除 trailer 中的 swap_status 区域
- 代码量减少 ~40%

### 24.2 Direct-XIP (无复制升级)

如果应用可以原地执行:

```c
#define MCUBOOT_DIRECT_XIP
// 可选: 支持回滚 (需要 trailer)
#define MCUBOOT_DIRECT_XIP_REVERT
```

**裁剪效果:**
- 无 swap 逻辑
- 如果不用 revert, 连 trailer 也不需要

### 24.3 升级策略对比

```c
/* ─── 选择 1: Overwrite Only (推荐最小化) ─── */
#define MCUBOOT_OVERWRITE_ONLY

/* ─── 选择 2: Direct-XIP ─── */
// #define MCUBOOT_DIRECT_XIP

/* ─── 选择 3: Swap (默认, 最大) ─── */
/* 不定义上述任何宏即为 Swap 模式 */
/* 可选具体算法: */
// #define MCUBOOT_SWAP_USING_MOVE
// #define MCUBOOT_SWAP_USING_OFFSET
```

---

## 25. 签名算法裁剪

### 25.1 签名算法选择

```c
/* ─── 仅选一个 ─── */

/* ECDSA-P256: 快速, 中等体积 */
#define MCUBOOT_SIGN_EC256

/* ECDSA-P384: 更高安全级别, 稍大 */
// #define MCUBOOT_SIGN_EC384

/* RSA-2048: 验证慢, 体积大 */
// #define MCUBOOT_SIGN_RSA

/* Ed25519: 快速, 现代, 但需要特定库支持 */
// #define MCUBOOT_SIGN_ED25519
```

### 25.2 完全无签名验证 (仅开发/封闭环境)

如果你的设备在物理上安全, 或者有其他方式保证固件完整性:

```c
// mcuboot_config.h — 没有任何 MCUBOOT_SIGN_* 宏

// 但仍需提供空的 key 数组:
// keys.c
#include <bootutil/sign_key.h>
const struct bootutil_key bootutil_keys[] = { {0} };
const int bootutil_key_cnt = 0;

// 同时需要: 不使用 MCUBOOT_VALIDATE_PRIMARY_SLOT
// (或即使定义了, 验证也会跳过签名步骤)
```

### 25.3 签名验证裁剪效果

```
┌──────────────────────────────────────────────┐
│ 配置                      │ Code  │ RAM      │
├───────────────────────────┼───────┼──────────┤
│ ECDSA-P256 + Mbed TLS    │ ~30KB │ ~2KB     │
│ ECDSA-P256 + Tinycrypt   │ ~18KB │ ~1.5KB   │
│ RSA-2048 + Mbed TLS      │ ~35KB │ ~3KB     │
│ Ed25519 + Tinycrypt      │ ~16KB │ ~1KB     │
│ 无签名                   │ ~4KB  │ ~0.5KB   │
└──────────────────────────────────────────────┘
```

---

## 26. 加密功能裁剪

### 26.1 加密后端选择

```c
/* ─── 三选一 ─── */

/* Mbed TLS: 功能完整, 体积大 */
#define MCUBOOT_USE_MBED_TLS

/* Tinycrypt: 轻量级, 仅支持 ECDSA + 基础功能 */
// #define MCUBOOT_USE_TINYCRYPT

/* 自定义: 最小体积, 需要自己实现 */
// #define MCUBOOT_USE_CUSTOM_CRYPTO
```

### 26.2 镜像加密的裁剪

```c
/* 如果你不需要镜像加密, 千万不要定义以下任何宏: */
// #define MCUBOOT_ENCRYPT_RSA       ← 不定义
// #define MCUBOOT_ENCRYPT_KW        ← 不定义
// #define MCUBOOT_ENCRYPT_EC256     ← 不定义
// #define MCUBOOT_ENCRYPT_X25519    ← 不定义

/* 上述任一定义都会自动触发:
   #define MCUBOOT_ENC_IMAGES
   从而引入大量加密相关代码 */
```

### 26.3 Tinycrypt vs Mbed TLS 体积对比

| 加密后端 | Code | RAM | 签名支持 | 加密支持 |
|---------|------|-----|---------|---------|
| Mbed TLS (完整) | ~80KB | ~8KB | RSA/ECDSA/Ed25519 | RSA-OAEP/AES-KW/ECIES |
| Mbed TLS (裁剪) | ~30KB | ~2KB | ECDSA only | 无 |
| Tinycrypt | ~15KB | ~1KB | ECDSA only | 无 |

**Mbed TLS 裁剪方法:**
```c
// mbedtls_config.h (自定义)
#define MBEDTLS_SHA256_C         // 仅 SHA-256
#define MBEDTLS_ECDSA_C          // 仅 ECDSA
#define MBEDTLS_ECP_C            // ECDSA 依赖
#define MBEDTLS_BIGNUM_C         // 大数运算
#define MBEDTLS_ASN1_PARSE_C     // ASN.1 解析
#define MBEDTLS_ASN1_WRITE_C     // ASN.1 写入
#define MBEDTLS_PLATFORM_C       // 平台抽象
// 所有其他模块全部禁用
```

---

## 27. 多镜像与依赖裁剪

### 27.1 镜像数量

```c
// mcuboot_config.h

/* 单镜像 (大多数情况) */
#define MCUBOOT_IMAGE_NUMBER 1

/* 双镜像 (如 Application + Bootloader 升级, 或双核) */
// #define MCUBOOT_IMAGE_NUMBER 2
```

**裁剪效果:**
- `MCUBOOT_IMAGE_NUMBER 1` 时, 多镜像循环和依赖检查被编译期优化掉
- 节省 ~2KB 代码和少量 RAM

### 27.2 镜像依赖

```c
/* 如果不启用多镜像, 依赖功能自动移除 */
/* 如果启用多镜像但不需要依赖检查: 签名时不加 -d 参数即可 */
```

### 27.3 Flash Sector 数量

```c
/* 默认 128 sectors — 如果你的 slot 分区少很多, 可以减小 */
#define MCUBOOT_MAX_IMG_SECTORS 32    /* 从 128 减到 32 */
```

**裁剪效果:**
- Trailer 中 swap_status 区域: `128 × 4 × 3 = 1536B` → `32 × 4 × 3 = 384B`
- RAM 中 boot_status 数组相应减小
- 如果你的 slot 只有 16 个 sector, 设为 16 即可

---

## 28. 日志/测量启动/数据共享裁剪

### 28.1 日志

```c
/* 关闭日志 — 节省 ~3KB 代码 + 串口依赖 */
// #define MCUBOOT_HAVE_LOGGING 0

/* 或只关闭调试日志: */
#define MCUBOOT_LOG_DBG(...)  /* 空 */
```

### 28.2 测量启动 (Measured Boot)

```c
/* 关闭测量启动 — 节省 TLV 处理和 CBOR 编码 */
// 不定义 MCUBOOT_MEASURED_BOOT
```

### 28.3 数据共享

```c
/* 关闭数据共享 — 节省共享内存区和 TLV 写入 */
// 不定义 MCUBOOT_DATA_SHARING
// 不定义 MCUBOOT_DATA_SHARING_BOOTINFO
```

### 28.4 验证缓存 (仅验证一次)

```c
/* 注意安全风险: 攻击者可在首次验证后修改 Flash */
// #define MCUBOOT_VALIDATE_PRIMARY_SLOT_ONCE
```

---

## 29. Boot Serial 裁剪

### 29.1 串口恢复功能

Boot Serial (boot_serial) 提供通过串口的 SMP 协议进行固件上传。如果不需要, 完全移除:

```c
/* 在构建系统中排除以下目录:
   boot/boot_serial/

   不定义任何 MCUBOOT_SERIAL_* 宏
*/

/* 如果你需要串口恢复, 可以保留并配置: */
// 支持直接上传到指定 slot
// #define MCUBOOT_SERIAL_DIRECT_IMAGE_UPLOAD
// 渐进式擦除
// #define MCUBOOT_ERASE_PROGRESSIVELY
```

**注意:** Boot Serial 需要额外的 ZCBOR 库 (`boot/zcbor/`), 会显著增加代码量 (~10-20KB), 不适合极小 Flash 设备。

### 29.2 不需要串口恢复时

确保在构建时排除:
- `boot/boot_serial/` 整个目录
- `boot/zcbor/` 整个目录
- 不定义任何 `MCUBOOT_SERIAL_*` 宏

---

## 30. 极端精简配置 (最小化 MCUboot)

### 30.1 最小化配置文件

适用于资源极端受限的 MCU (如 Cortex-M0, 32KB Flash, 4KB RAM):

```c
// mcuboot_config_minimal.h — 极端精简配置
#ifndef __MCUBOOT_CONFIG_MINIMAL_H__
#define __MCUBOOT_CONFIG_MINIMAL_H__

/* ══════════════════════════════════════════════════
 * 签名: ECDSA-P256 (最小可用的安全签名)
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_SIGN_EC256

/* ══════════════════════════════════════════════════
 * 升级策略: 覆盖模式 (移除所有 Swap 代码)
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_OVERWRITE_ONLY
#define MCUBOOT_OVERWRITE_ONLY_FAST

/* ══════════════════════════════════════════════════
 * 加密后端: Tinycrypt (最小体积)
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_USE_TINYCRYPT

/* ══════════════════════════════════════════════════
 * 安全设置: 最小
 * ══════════════════════════════════════════════════ */
/* 不验证主槽 (节省启动时间) */
/* #define MCUBOOT_VALIDATE_PRIMARY_SLOT */

/* ══════════════════════════════════════════════════
 * Flash 与内存: 精简
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_MAX_IMG_SECTORS   32    /* 从 128 减到 32 */
#define MCUBOOT_IMAGE_NUMBER      1

/* ══════════════════════════════════════════════════
 * 日志: 关闭
 * ══════════════════════════════════════════════════ */
/* #define MCUBOOT_HAVE_LOGGING */  /* 不定义即关闭 */

/* ══════════════════════════════════════════════════
 * 加密: 关闭
 * ══════════════════════════════════════════════════ */
/* 不定义任何 MCUBOOT_ENCRYPT_* 宏 */

/* ══════════════════════════════════════════════════
 * 看门狗 / CPU 空闲
 * ══════════════════════════════════════════════════ */
#define MCUBOOT_CPU_IDLE()  do { __WFI(); } while (0)

/* ══════════════════════════════════════════════════
 * 不需要的编译单元: 从构建系统中排除
 * ══════════════════════════════════════════════════
 * - boot/boot_serial/       (串口恢复)
 * - boot/zcbor/             (CBOR 库)
 * - boot/bootutil/src/swap_*.c        (Swap 算法)
 * - boot/bootutil/src/boot_record.c   (测量启动)
 * - boot/bootutil/src/encrypted*.c    (镜像加密)
 * - boot/bootutil/src/fault_injection_hardening*.c (FIH)
 */

#endif /* __MCUBOOT_CONFIG_MINIMAL_H__ */
```

### 30.2 最小构建系统中的源码列表

```makefile
# Makefile — 最小化构建
BOOTUTIL_MIN_SRC = \
    boot/bootutil/src/loader.c               \
    boot/bootutil/src/bootutil_misc.c        \
    boot/bootutil/src/bootutil_public.c      \
    boot/bootutil/src/image_validate.c       \
    boot/bootutil/src/image_ecdsa.c          \
    boot/bootutil/src/bootutil_img_hash.c    \
    boot/bootutil/src/bootutil_find_key.c    \
    boot/bootutil/src/tlv.c                  \
    boot/bootutil/src/caps.c

# 不包括:
#   swap_scratch.c, swap_move.c, swap_offset.c, swap_misc.c
#   encrypted.c, encrypted_psa.c
#   boot_record.c
#   image_rsa.c, image_ed25519.c
#   ram_load.c
#   fault_injection_hardening*.c
```

### 30.3 最小化 Flash 布局

```
+----------------------+ 0x08000000
| MCUboot (最小配置)   | ~12KB (包含 Tinycrypt)
+----------------------+ 0x08003000
| Primary Slot         | 用户固件 (无 Secondary, 无 Scratch)
|                      |
+----------------------+
```

**注意:** Overwrite Only 模式不需要 Secondary Slot 和 Scratch Area。但你需要其他方式将新固件写入 Secondary 位置 (如通过应用自身的 DFU 功能)。

---

---

# 第四篇: 串口升级移植指南 (Boot Serial)

## 33. 串口升级协议与架构

### 33.1 什么是 Boot Serial

Boot Serial 是 MCUboot 内置的**串口固件升级通道**。它使用 MCUmgr (Simple Management Protocol, SMP) 协议, 通过串口传输来实现固件上传、镜像列表查询和芯片复位等功能。对裸机移植而言, 这是最实用的"救砖"和产线烧录通道。

### 33.2 协议栈分层

```
┌──────────────────────────────────────┐
│          PC 端工具 (mcumgr CLI)       │
│   go install github.com/apache/      │
│   mynewt-mcumgr-cli/mcumgr@latest    │
└──────────────┬───────────────────────┘
               │ 串口 (UART)
┌──────────────▼───────────────────────┐
│        SMP 帧协议 (Newtmgr)          │
│  + Base64 编码 + CRC16 校验          │
│  + 分帧 (每帧 ≤127 字节 + 头尾)      │
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│        CBOR 编解码 (ZCbor)           │
│  命令: echo / reset / image list /   │
│        image upload / slot info      │
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│    MCUboot Boot Serial 核心          │
│  boot/boot_serial/src/boot_serial.c  │
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│   你的移植层 (需要实现的部分)         │
│  - UART 读写 (boot_uart_funcs)       │
│  - OS 抽象 (malloc/延时/复位)        │
│  - Base64 / CRC16                   │
│  - 复位原因检测                       │
└──────────────────────────────────────┘
```

### 33.3 裸机串口升级的启动流程

```
上电/复位
    │
    ├── 读取复位原因寄存器
    │
    ├── 是上电复位或引脚复位?
    │   ├── 是 → 检测 GPIO (如按键) 是否按下
    │   │   ├── 按下 → 进入串口恢复模式
    │   │   └── 未按下 → 正常启动流程 (boot_go)
    │   └── 否 (软件复位) → 正常启动流程
    │
    └── 正常启动流程:
        boot_go(&rsp) → jump_to_app(&rsp)

    串口恢复模式:
        boot_serial_start(&uart_funcs)  ← 阻塞等待升级命令
            ↓
        收到 image upload → 写入 Secondary Slot → 收到 reset → 复位
```

### 33.4 为什么需要复位原因判断

| 复位原因 | MCUboot 行为 |
|----------|-------------|
| **上电复位 (POR)** | 可以检测 GPIO 按键, 决定是否进串口恢复 |
| **外部引脚复位 (PIN)** | 可以检测 GPIO 按键 |
| **软件复位 (SYSRESETREQ)** | **跳过** 按键检测, 直接正常启动 —— 避免 App 热重启时误入升级模式 |
| **看门狗复位** | 通常视为异常, 直接正常启动 |

**HC32L196 读取复位原因的示例:**
```c
// 读取 RCU 复位标志寄存器
uint32_t rst_flag = RCU->RST_FLAG;

if (rst_flag & RCU_RST_FLAG_POR) {
    // 上电复位
} else if (rst_flag & RCU_RST_FLAG_PIN) {
    // 引脚复位
} else if (rst_flag & RCU_RST_FLAG_SW) {
    // 软件复位 — 跳过按键检测
}
```

### 33.5 Trailer 字段的升级状态语义

这是理解 MCUboot 升级回滚机制的核心:

| 字段 | 位置 | 取值含义 |
|------|------|---------|
| `magic` (16B) | Trailer 最末尾 | 魔数 → trailer 有效; 0xFF → 无效/不存在 |
| `image_ok` (1B) | magic 上方 | `0x01`=已确认 / `0xFF`=未确认 |
| `copy_done` (1B) | image_ok 上方 | `0x01`=Swap 已完成 / `0xFF`=未完成 |
| `swap_info` (1B) | copy_done 上方 | 低 4bit=swap 类型, 高 4bit=镜像编号 |
| `swap_status` | swap_info 上方 | 按 sector 记录交换进度 (实现断电续传) |

**升级状态机:**
```
初始状态 → 用户写入 Secondary 镜像 (带 magic, image_ok=0xFF)
        ↓
    下次启动检测到 Secondary magic=good + image_ok=0xFF
        → STATE II: TEST (测试性交换)
        ↓
    Swap 完成, 启动新固件
        ↓
    新固件调用 boot_set_confirmed() → 写 image_ok=0x01
        → STATE III: PERM (永久生效)
        ↓
    或: 新固件崩溃 (未调 boot_set_confirmed)
        → 下次启动: STATE IV: REVERT (回滚到旧固件)
```

---

## 34. 裸机串口移植必须实现的部分

### 34.1 新增文件清单

在基础裸机移植之上, 增加串口升级需要:

```
my_baremetal_port/
├── ... (原有的基础移植文件)
├── serial/
│   ├── serial_adapter.c       ← UART 适配 + boot_serial_start 调用
│   └── serial_adapter.h
├── os/
│   ├── os_malloc.c            ← malloc/free 实现 (给 boot_serial 用)
│   └── os_cputime.c           ← 延时/获取时间 (用于超时检测)
├── hal/
│   └── hal_system.c           ← hal_system_reset() 实现
├── base64/
│   └── base64.c               ← Base64 编解码 (可从 Mbed TLS 提取)
├── crc/
│   └── crc16.c                ← CRC16-CCITT 实现
└── 构建系统中增加:
    boot/boot_serial/src/boot_serial.c
    boot/boot_serial/src/zcbor_bulk.c
    boot/zcbor/src/zcbor_decode.c
    boot/zcbor/src/zcbor_encode.c
    boot/zcbor/src/zcbor_common.c
```

### 34.2 必须实现的接口全景

| 接口类别 | 函数/宏 | 来源文件 | 说明 |
|---------|---------|---------|------|
| **UART** | `boot_uart_funcs` 结构体 | `boot_serial.h` | read/write 函数指针 |
| **内存** | `os_malloc()` / `os_free()` | `os/os_malloc.h` | boot_serial 内部需要动态分配 |
| **延时** | `os_cputime_delay_usecs()` | `os/os_cputime.h` | reset 命令延时 250ms |
| **复位** | `hal_system_reset()` | `hal/hal_system.h` | 软件复位系统 |
| **Base64** | `base64_encode()` / `base64_decode()` | `base64/base64.h` | SMP 帧编解码 |
| **CRC16** | `crc16_ccitt()` | `crc/crc16.h` | 帧校验 |
| **字节序** | `ntohs()` / `htons()` | `os/endian.h` | 网络字节序转换 |

### 34.3 宏配置

在 `mcuboot_config.h` 中必须添加:

```c
/* ─── 串口升级配置 ─── */

/* 串口接收缓冲区大小 */
#define MCUBOOT_SERIAL_MAX_RECEIVE_SIZE  512

/* 可选: 支持直接上传到任意 slot */
#define MCUBOOT_SERIAL_DIRECT_IMAGE_UPLOAD

/* 可选: 渐进式擦除 (边收边擦, 减少首次等待) */
#define MCUBOOT_ERASE_PROGRESSIVELY

/* 可选: 超时自动退出串口恢复 (ms) */
/* #define MCUBOOT_SERIAL_WAIT_FOR_DFU */

/* 可选: 支持 echo 命令 */
#define MCUBOOT_BOOT_MGMT_ECHO

/* 可选: 支持 slot info 查询 */
#define MCUBOOT_SERIAL_IMG_GRP_SLOT_INFO

/* 可选: 非对齐写缓冲区大小 */
#define MCUBOOT_SERIAL_UNALIGNED_BUFFER_SIZE  32
```

---

## 35. UART 驱动适配 (boot_uart_funcs)

### 35.1 接口定义

`boot_serial.h` 中定义的 UART 函数指针接口:

```c
struct boot_uart_funcs {
    /**
     * 从 UART 读取一行
     * @param str     输出缓冲区
     * @param cnt     最多读多少个字节
     * @param newline 输出: 如果读到换行符, 设为非零值
     * @return 实际读取的字节数, ≤0 表示没读到数据
     */
    int (*read)(char *str, int cnt, int *newline);

    /**
     * 向 UART 写入数据
     * @param ptr 数据指针
     * @param cnt 字节数
     */
    void (*write)(const char *ptr, int cnt);
};
```

### 35.2 HC32L196 / 通用裸机 UART 实现

```c
// serial_adapter.c
#include <stdint.h>
#include <stdbool.h>
#include "boot_serial/boot_serial.h"
#include "hc32l196_uart.h"   /* 你的 MCU UART HAL */

/* ─── UART 硬件底层 ─── */

static int uart_poll_read(char *str, int cnt, int *newline)
{
    int bytes = 0;
    *newline = 0;

    while (bytes < cnt) {
        if (!uart_rx_ready()) {
            break;   /* 当前没有数据, 不等了 */
        }
        char c = uart_getc();
        str[bytes++] = c;
        if (c == '\n') {
            *newline = 1;
            break;
        }
    }
    return bytes;
}

static void uart_poll_write(const char *ptr, int cnt)
{
    for (int i = 0; i < cnt; i++) {
        uart_putc(ptr[i]);
    }
}

/* ─── boot_uart_funcs 实例 ─── */

static const struct boot_uart_funcs uart_funcs = {
    .read  = uart_poll_read,
    .write = uart_poll_write,
};

/* ─── 串口恢复入口 ─── */

void boot_serial_enter(void)
{
    /* 初始化 UART: 波特率 115200, 8N1 */
    uart_init(115200);

    /* 进入串口恢复模式 (阻塞, 直到收到 reset 命令或超时) */
#ifdef MCUBOOT_SERIAL_WAIT_FOR_DFU
    /* 带超时: 5 秒内收到升级命令则停留, 否则退出 */
    boot_serial_check_start(&uart_funcs, 5000);
#else
    /* 永久等待升级命令 */
    boot_serial_start(&uart_funcs);
#endif
}
```

### 35.3 关键点

1. `read()` 函数必须是**非阻塞轮询**模式。每次调用尽可能读, 没数据就返回 0。MCUboot 内部会在循环中反复调用
2. `newline` 必须正确设置 — 这是帧分隔的标志
3. 如果需要喂狗, MCUboot 内部在串口接收循环中会调用 `MCUBOOT_WATCHDOG_FEED()`
4. 如果需要低功耗, MCUboot 内部会调用 `MCUBOOT_CPU_IDLE()`

---

## 36. OS 抽象层适配 (os/os_malloc/hal_system)

### 36.1 内存分配 (os_malloc)

boot_serial 在内部分配 CBOR 编码器和接收缓冲区时需要使用 `os_malloc()`:

```c
// os/os_malloc.h
#ifndef OS_MALLOC_H__
#define OS_MALLOC_H__

#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

void *os_malloc(size_t size);
void os_free(void *ptr);

#ifdef __cplusplus
}
#endif

#endif /* OS_MALLOC_H__ */
```

```c
// os_malloc.c — 裸机静态池实现 (无需堆)
#include <string.h>

#define POOL_SIZE  4096
static uint8_t malloc_pool[POOL_SIZE];
static size_t  malloc_used = 0;

void *os_malloc(size_t size)
{
    /* 简单首次适配 (stream allocator) — 适用于 boot_serial 顺序分配 */
    size = (size + 7) & ~7;  /* 8 字节对齐 */
    if (malloc_used + size > POOL_SIZE) {
        return NULL;
    }
    void *ptr = &malloc_pool[malloc_used];
    malloc_used += size;
    return ptr;
}

void os_free(void *ptr)
{
    /* 简化实现: boot_serial 只在退出时一次性释放 */
    /* 如果 malloc_used 达到上限, 可以在这里重置 */
    (void)ptr;
}

/* 在 boot_serial 退出后重置分配器 */
void os_malloc_reset(void)
{
    malloc_used = 0;
}
```

### 36.2 系统复位 (hal_system_reset)

```c
// hal_system.c
#include "hc32l196_system.h"

void hal_system_reset(void)
{
    /* 方法 1: 使用 SCB AIRCR 软件复位 */
    SCB->AIRCR = (0x5FA << 16) | SCB_AIRCR_SYSRESETREQ_Msk;
    __DSB();
    while (1);

    /* 方法 2: 使用芯片特定的复位控制器 (如有) */
    // RCU_SoftwareReset();
}
```

### 36.3 延时与时间 (os_cputime)

```c
// os_cputime.c
#include "hc32l196_timer.h"

/* us 级延时 — 裸机通常用 SysTick 或 DWT 实现 */
void os_cputime_delay_usecs(uint32_t us)
{
    /* 使用 SysTick 实现微秒延时 */
    uint32_t start = SysTick->VAL;
    uint32_t ticks = us * (SystemCoreClock / 1000000);

    while (1) {
        uint32_t current = SysTick->VAL;
        uint32_t elapsed = (start - current) & 0xFFFFFF;
        if (elapsed >= ticks) break;
    }
}

/* 获取毫秒级时间戳 (超时检测用) */
uint32_t os_cputime_get32(void)
{
    /* 使用 SysTick 或 DWT_CYCCNT */
    return your_ms_tick_counter;
}
```

### 36.4 字节序转换 (endian.h)

```c
// os/endian.h
#ifndef OS_ENDIAN_H__
#define OS_ENDIAN_H__

/* Cortex-M 是小端, 但网络字节序是大端 */
static inline uint16_t htons(uint16_t x) {
    return (uint16_t)((x >> 8) | (x << 8));
}

static inline uint16_t ntohs(uint16_t x) {
    return htons(x);
}

#endif
```

---

## 37. ZCbor 与依赖库集成

### 37.1 ZCbor 是什么

CBOR (Concise Binary Object Representation) 是一种二进制 JSON。MCUboot 使用 ZCbor 库进行 CBOR 编解码, 用于解析和构造 SMP 协议消息。

### 37.2 集成 ZCbor

ZCbor 库以**单头文件库**的方式提供, 位于 `boot/zcbor/` 下:

**需要编译的文件:**
```
boot/zcbor/src/zcbor_decode.c
boot/zcbor/src/zcbor_encode.c
boot/zcbor/src/zcbor_common.c
boot/boot_serial/src/zcbor_bulk.c
```

**配置宏 (在 mcuboot_config.h 或构建系统中):**
```c
/* ZCbor 需要的配置 */
#define ZCBOR_ENCODE_DEBUG      0
#define ZCBOR_DECODE_DEBUG      0
#define ZCBOR_CANONICAL         0   /* 关闭规范化 (节省代码) */
#define ZCBOR_VERBOSE            0
#define ZCBOR_MAP_SMART_SEARCH  0   /* 关闭智能搜索 (节省 RAM) */
```

### 37.3 Base64 实现

可以从 Mbed TLS 中提取 `base64.c` / `base64.h`, 或自己实现:

```c
// base64.c (精简版, 仅用于 boot_serial)
#include <stdint.h>
#include <stddef.h>

static const char b64_table[] =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

int base64_encode(const void *src, size_t slen, char *dst, int null_terminate)
{
    const uint8_t *s = (const uint8_t *)src;
    size_t i;
    char *p = dst;

    for (i = 0; i + 2 < slen; i += 3) {
        *p++ = b64_table[(s[i] >> 2) & 0x3F];
        *p++ = b64_table[((s[i] << 4) | (s[i+1] >> 4)) & 0x3F];
        *p++ = b64_table[((s[i+1] << 2) | (s[i+2] >> 6)) & 0x3F];
        *p++ = b64_table[s[i+2] & 0x3F];
    }
    if (i < slen) {
        *p++ = b64_table[(s[i] >> 2) & 0x3F];
        if (i + 1 < slen) {
            *p++ = b64_table[((s[i] << 4) | (s[i+1] >> 4)) & 0x3F];
            *p++ = b64_table[(s[i+1] << 2) & 0x3F];
            *p++ = '=';
        } else {
            *p++ = b64_table[(s[i] << 4) & 0x3F];
            *p++ = '=';
            *p++ = '=';
        }
    }
    if (null_terminate) {
        *p = '\0';
    }
    return p - dst;
}
```

> **简化方案:** 如果你的 MCUboot 已链接了 Mbed TLS, 可以直接使用 `mbedtls_base64_encode()` / `mbedtls_base64_decode()` (定义 `base64_encode`/`base64_decode` 宏指向它们)。

### 37.4 CRC16 实现

```c
// crc16.c
#include <stdint.h>

/* CRC16-CCITT (x^16 + x^12 + x^5 + 1), 初始值 0 */
uint16_t crc16_ccitt(uint16_t initial, const void *buf, size_t len)
{
    const uint8_t *p = (const uint8_t *)buf;
    uint16_t crc = initial;

    while (len--) {
        crc ^= (uint16_t)(*p++) << 8;
        for (int i = 0; i < 8; i++) {
            if (crc & 0x8000) {
                crc = (crc << 1) ^ 0x1021;
            } else {
                crc <<= 1;
            }
        }
    }
    return crc;
}
```

### 37.5 构建系统中添加 Boot Serial 源码

```makefile
# Makefile 中新增
BOOT_SERIAL_SRC = \
    boot/boot_serial/src/boot_serial.c    \
    boot/boot_serial/src/zcbor_bulk.c     \
    boot/zcbor/src/zcbor_decode.c         \
    boot/zcbor/src/zcbor_encode.c         \
    boot/zcbor/src/zcbor_common.c         \

PORT_SERIAL_SRC = \
    serial/serial_adapter.c   \
    os/os_malloc.c            \
    os/os_cputime.c           \
    hal/hal_system.c          \
    base64/base64.c           \
    crc/crc16.c

# 总源码
SRC = $(BOOTUTIL_SRC) $(BOOT_SERIAL_SRC) $(PORT_SRC) $(PORT_SERIAL_SRC)

# 添加包含路径
CFLAGS += -I boot/boot_serial/include
CFLAGS += -I boot/zcbor/include
CFLAGS += -I serial
CFLAGS += -I os
```

---

## 38. 串口恢复模式入口与跳转决策

### 38.1 完整的 main.c (集成串口恢复)

```c
// main.c — 带串口恢复的完整入口
#include <stdint.h>
#include <stdbool.h>
#include "bootutil/bootutil.h"
#include "bootutil/image.h"
#include "hal/flash_drv.h"
#include "hal/hc32l196_system.h"
#include "serial/serial_adapter.h"

/* ─── 跳转到用户应用 ─── */

static void jump_to_app(uint32_t app_addr)
{
    uint32_t *vectors = (uint32_t *)app_addr;
    uint32_t stack_ptr = vectors[0];
    uint32_t entry_point = vectors[1];

    /* 关闭全局中断, 清理外设 */
    __disable_irq();

    /* 清理 NVIC 挂起和使能 */
    for (int i = 0; i < 8; i++) {
        NVIC->ICPR[i] = 0xFFFFFFFF;
        NVIC->ICER[i] = 0xFFFFFFFF;
    }

    /* 清理 SysTick */
    SysTick->CTRL = 0;

    __DSB();
    __ISB();

    __set_MSP(stack_ptr);

    typedef void (*app_t)(void);
    ((app_t)entry_point)();

    while (1);
}

/* ─── 检测是否进入串口恢复 ─── */

static bool should_enter_serial_recovery(void)
{
    /* 读取复位原因 */
    uint32_t rst_flag = RCU->RST_FLAG;

    /* 只有上电复位和引脚复位才允许检测按键 */
    if (!(rst_flag & (RCU_RST_FLAG_POR | RCU_RST_FLAG_PIN))) {
        return false;  /* 软件复位: 直接正常启动 */
    }

    /* 检测 GPIO 按键 (如 PA0 按低) */
    if (gpio_read_pin(GPIOA, 0) == 0) {
        /* 防抖延时 */
        for (volatile int i = 0; i < 100000; i++);
        if (gpio_read_pin(GPIOA, 0) == 0) {
            return true;
        }
    }

    return false;
}

/* ─── 硬件初始化 ─── */

static void board_init(void)
{
    SystemClock_Config();
    hal_flash_init();
    gpio_init();   /* GPIO 初始化 (用于按键检测) */
}

/* ─── 主入口 ─── */

int main(void)
{
    struct boot_rsp rsp;

    board_init();

    /* 决策: 串口恢复 or 正常启动 */
    if (should_enter_serial_recovery()) {
        boot_serial_enter();  /* 阻塞在串口恢复 */
        /* 从串口恢复退出后, 软件复位 */
        hal_system_reset();
        /* 不会到达这里 */
    }

    /* 正常启动流程 */
    fih_ret rc = boot_go(&rsp);

    if (fih_eq(rc, FIH_SUCCESS)) {
        jump_to_app(rsp.br_image_off);
    }

    /* boot_go 失败: 进入死循环 + 错误指示 */
    while (1) {
        led_error_blink();
        MCUBOOT_WATCHDOG_FEED();
    }
    return 0;
}
```

### 38.2 mcumgr CLI 使用流程

```bash
# 1. 安装 mcumgr
go install github.com/apache/mynewt-mcumgr-cli/mcumgr@latest

# 2. 配置连接
mcumgr conn add serial_1 type="serial" connstring="COM3,baud=115200"

# 3. 查询镜像列表
mcumgr image list -c serial_1

# 4. 上传新固件 (到 Secondary Slot)
mcumgr image upload myapp.signed.bin -c serial_1

# 5. 复位设备 (进入正常启动流程, MCUboot 会执行 Swap)
mcumgr reset -c serial_1
```

---

## 39. 完整串口升级移植示例 (HC32L196 / Cortex-M0+)

### 39.1 HC32L196 资源与分区

| 参数 | 值 |
|------|-----|
| Flash | 256KB (0x00000000 ~ 0x00040000) |
| RAM | 32KB (0x20000000 ~ 0x20008000) |
| 内核 | Cortex-M0+ (无 VTOR) |
| Sector | 4KB × 64 |

**分区方案:**
```
+----------------------+ 0x00000000
| MCUboot + BootSerial | 32KB
+----------------------+ 0x00008000
| Primary Slot (Slot0) | 96KB
+----------------------+ 0x00020000
| Secondary Slot(Slot1)| 96KB
+----------------------+ 0x00038000
| Scratch Area         | 32KB (最大sector保证)
+----------------------+ 0x00040000
```

### 39.2 HC32L196 sysflash.h

```c
// sysflash.h
#ifndef __SYSFLASH_H__
#define __SYSFLASH_H__

#define FLASH_DEVICE_INTERNAL_FLASH  0
#define FLASH_DEVICE_ID              0

#define FLASH_BASE                   0x00000000
#define FLASH_SIZE                   0x00040000   // 256KB

#define BOOTLOADER_SIZE              0x00008000   // 32KB
#define SLOT_SIZE                    0x00018000   // 96KB
#define SCRATCH_SIZE                 0x00008000   // 32KB

#define FLASH_AREA_BOOTLOADER         0
#define FLASH_AREA_IMAGE_PRIMARY(i)   (1 + (i))
#define FLASH_AREA_IMAGE_SECONDARY(i) (2 + (i))
#define FLASH_AREA_IMAGE_SCRATCH      3

/* Flash 擦除值为 0xFF (HC32L196 内部 Flash 擦除后为 0xFF) */
#define FLASH_ERASED_VALUE            0xFF

#endif
```

### 39.3 HC32L196 Flash 驱动注意点

```c
// hal/flash_drv.c — HC32L196 特定实现

/* HC32L196 Flash 操作关键步骤:
 * 1. 解锁: 写 Flash 控制寄存器解锁序列
 * 2. 擦除: 按页 (sector=4KB) 擦除
 * 3. 编程: 按字 (32-bit) 写入
 * 4. 上锁: 重新锁定 Flash 控制器
 */

int hal_flash_program(uint32_t addr, const uint8_t *data, uint32_t len)
{
    /* HC32L196 Flash 解锁序列 */
    M0P_FLASH->CR_f.SPM = 0x5A5A;
    M0P_FLASH->CR_f.SPM = 0xA5A5;

    uint32_t words = len / 4;
    uint32_t *src = (uint32_t *)data;

    for (uint32_t i = 0; i < words; i++) {
        *((volatile uint32_t *)addr) = src[i];
        /* 等待写入完成 */
        while (M0P_FLASH->SR_f.BUSY);
        addr += 4;
    }

    /* 上锁 */
    M0P_FLASH->CR_f.SPM = 0;
    return 0;
}

int hal_flash_erase(uint32_t addr, uint32_t len)
{
    M0P_FLASH->CR_f.SPM = 0x5A5A;
    M0P_FLASH->CR_f.SPM = 0xA5A5;

    uint32_t pages = len / 4096;  /* HC32L196: 每页 4KB */

    for (uint32_t i = 0; i < pages; i++) {
        M0P_FLASH->ADDR = addr + i * 4096;
        M0P_FLASH->CR_f.OP = 1;  /* 页擦除 */
        M0P_FLASH->CR_f.START = 1;
        while (M0P_FLASH->SR_f.BUSY);
    }

    M0P_FLASH->CR_f.SPM = 0;
    return 0;
}

uint32_t hal_flash_get_write_alignment(void)
{
    return 4;  /* Cortex-M0+: 字对齐写入 */
}

uint32_t hal_flash_get_sector_size(uint32_t addr)
{
    (void)addr;
    return 4096;  /* HC32L196 统一 4KB 页 */
}
```

---

## 40. Cortex-M0+ 无 VTOR 跳转专题

### 40.1 问题本质

Cortex-M3/M4/M7 有 **VTOR (Vector Table Offset Register)**, 可以直接将向量表重定位到任意地址。但 Cortex-M0/M0+ **没有 VTOR**, 向量表固定在地址 0x00000000。

这意味着: 不能让 CPU 在中断时自动从 `0x8000` 或 `0x20000` 读取向量表。

### 40.2 方案一: 硬件重映射 (推荐, 需芯片支持)

利用芯片的 **SYSCFG** 模块将 RAM 映射到 0x00000000:

**步骤:**
1. **预留 RAM 空间** — 修改链接脚本, 将 RAM 起始地址后推 256 字节
2. **复制向量表** — 跳转前将目标 APP 的向量表复制到 `0x20000000`
3. **切换映射** — 配置 SYSCFG, 使 `0x00000000` 映射到 SRAM

```c
// Cortex-M0+ RAM 重映射跳转
static void jump_to_app_m0p(uint32_t app_addr)
{
    /* 1. 关闭中断 */
    __disable_irq();

    /* 2. 清理 NVIC */
    for (int i = 0; i < 8; i++) {
        NVIC->ICPR[i] = 0xFFFFFFFF;
        NVIC->ICER[i] = 0xFFFFFFFF;
    }
    SysTick->CTRL = 0;

    /* 3. 从 APP 的 Flash 地址复制向量表到 RAM 开头 */
    /*    HC32L196 向量表: 48 个向量 = 192 字节 */
    uint32_t *app_vectors = (uint32_t *)app_addr;
    uint32_t *ram_vectors  = (uint32_t *)0x20000000;
    for (int i = 0; i < 48; i++) {
        ram_vectors[i] = app_vectors[i];
    }

    /* 4. 配置 SYSCFG 将 0x00000000 映射到 SRAM */
    /*    (具体寄存器名和位查 HC32L196 数据手册) */
    // SYSCFG->CFGR1 |= SYSCFG_CFGR1_MEM_MODE_0;  /* MEM_MODE = SRAM */

    __DSB();
    __ISB();

    /* 5. 设置 MSP 并跳转 */
    __set_MSP(app_vectors[0]);
    ((void (*)(void))app_vectors[1])();

    while (1);
}
```

**链接脚本修改 (预留 RAM 前 256 字节):**
```ld
/* HC32L196 链接脚本 — 用户应用 */
RAM (rw) : ORIGIN = 0x20000100, LENGTH = 32K - 0x100
/* 前 256 字节保留给向量表重映射使用 */
```

### 40.3 方案二: 软件中断转发 (无 SYSCFG 时备用)

Bootloader 捕获所有中断, 查询 APP 向量表并转发:

```c
// 在 Bootloader 中定义统一的中断转发入口
void HardFault_Handler(void)       { forward_irq(3); }
void SVC_Handler(void)             { forward_irq(11); }
void PendSV_Handler(void)          { forward_irq(14); }
void SysTick_Handler(void)         { forward_irq(15); }
void UART0_IRQHandler(void)        { forward_irq(16); }
// ... 为每个使用的中断定义

static uint32_t g_app_vector_base = 0;

void forward_irq(uint32_t irq_num)
{
    if (g_app_vector_base == 0) {
        while (1);  /* 还没有 APP 被加载 */
    }
    /* 从 APP 向量表中查找对应的处理函数并调用 */
    uint32_t *app_vectors = (uint32_t *)g_app_vector_base;
    void (*handler)(void) = (void (*)(void))app_vectors[irq_num];
    if (handler) {
        handler();
    }
}
```

**方案二的开销:** 每次中断多几十个时钟周期 (查表 + 间接跳转), 但对于非实时性要求极高的场景可以接受。

### 40.4 两种方案对比

| 对比项 | 方案一: RAM 重映射 | 方案二: 软件转发 |
|--------|-------------------|-----------------|
| 额外 RAM | 192~256 字节 | 几乎为 0 |
| 中断延迟 | 无额外开销 | +20~40 周期 |
| 芯片要求 | 需要 SYSCFG 支持 | 任何 Cortex-M0+ |
| 实现复杂度 | 中等 | 较高 |
| APP 改动 | 链接脚本偏移 RAM | 无 |
| 推荐度 | ⭐⭐⭐ 优先尝试 | ⭐⭐ 备用 |

---

## 41. 双 Slot 链接地址与 Keil 工程管理

### 41.1 为什么两个 Slot 不能使用相同的链接地址

在无 VTOR 的 Cortex-M0+ 上, **每个 APP 必须编译为对应的绝对地址**:

| 编译目标 | 链接地址 (Flash) | 说明 |
|---------|-----------------|------|
| app_slot0 | `0x00008000` | Slot 0 的起始地址 |
| app_slot1 | `0x00020000` | Slot 1 的起始地址 |

如果两个 APP 都用 `0x8000` 编译:
- Slot 0 运行 ✅
- Slot 1 运行 ❌ — 代码中的绝对地址引用 (如函数调用 `BL 0x8xxx`) 会跳转到错误位置

### 41.2 Keil 工程管理策略

**策略 A: 两个独立 Keil 工程 (简单, 推荐)**

```
project_slot0.uvprojx   → app_slot0.hex → 用 imgtool 签名
project_slot1.uvprojx   → app_slot1.hex → 用 imgtool 签名
```

配置差异仅在于:
- `Options → Target → IROM1 Start`: `0x8000` vs `0x20000`
- 条件编译宏: `SLOT0` vs `SLOT1`

**策略 B: 单个 Keil 工程 + 多 Target (推荐)**

在一个 `.uvprojx` 中创建两个 Target:
- Target: `Slot0_Release` → IROM1 = `0x8000`, Define: `SLOT0`
- Target: `Slot1_Release` → IROM1 = `0x20000`, Define: `SLOT1`

切换 Target 即可编译不同版本, 无需改代码。

**在代码中使用条件编译:**
```c
// app_main.c
#ifdef SLOT0
  #define APP_FLASH_BASE  0x00008000
#elif defined(SLOT1)
  #define APP_FLASH_BASE  0x00020000
#else
  #error "Must define SLOT0 or SLOT1"
#endif

/* 链接地址通过 scatter file / Target Options 设置 */
/* 此宏仅用于需要知道自身地址的代码 */
```

### 41.3 imgtool 签名命令

两套后处理命令 (可在 Keil 的 "After Build" 中配置):

```bash
# Slot 0 签名:
imgtool sign -k mykey.pem --align 4 -v "1.0.0" -H 0x20 \
    --pad --slot-size 0x18000 --max-sectors 24 \
    app_slot0.bin app_slot0.signed.bin

# Slot 1 签名:
imgtool sign -k mykey.pem --align 4 -v "1.0.0" -H 0x20 \
    --pad --slot-size 0x18000 --max-sectors 24 \
    app_slot1.bin app_slot1.signed.bin
```

### 41.4 OTA 升级时的镜像分发

| 升级场景 | 需要上传的镜像 |
|---------|--------------|
| 首次量产烧录 | `app_slot0.signed.bin` → 烧录到 Slot 0 |
| OTA 升级 | `app_slot1.signed.bin` → 通过串口上传到 Slot 1 |
| 回滚后再升级 | `app_slot1.signed.bin` → 再次上传到 Slot 1 |

> **总结:** 编译两份, 正常运行的永远是 Slot 0 版本, OTA 分发的永远是 Slot 1 版本。MCUboot Swap 机制负责在两者之间切换。

---
---

## 附录

## 42. 常见问题 FAQ

### Q1: 裸机移植 MCUboot 用 Mbed TLS 要动态内存分配吗？能用 Tinycrypt 替代吗？

**可以直接用 Tinycrypt，而且裸机更推荐它。**

Mbed TLS 在 MCUboot 的实际使用中，SHA-256 和 ECDSA-P256 路径并不触发内部的 `calloc`/`free`。`mbedtls_sha256_context` 约 224 字节纯结构体，`_init()`/`_free()` 只是清零操作，不走 `malloc`。但官方 PORTING.md 标注了"需要提供 calloc/free"，这是面向 RSA / X.509 等真正需要动态分配的模块。

Tinycrypt 从源码层面就零动态内存：

```
Tinycrypt SHA-256上下文:  struct tc_sha256_state_struct  (~108字节纯栈结构体)
Tinycrypt ECDSA上下文:    typedef uintptr_t               (占位符, uECC全栈计算)
Mbed TLS SHA-256上下文:   mbedtls_sha256_context          (~224字节纯结构体)
Mbed TLS ECDSA上下文:     mbedtls_ecdsa_context           (含ecp_group/ecp_point/mpi)
```

**对比：**

| 特性 | Mbed TLS | Tinycrypt |
|------|---------|-----------|
| 动态内存需求 | 官方说需要（实际SHA+ECDSA路径不用） | 零动态分配 |
| SHA-256 FLASH 体积 | ~8KB | ~4KB |
| ECDSA 验证 FLASH 体积 | ~12KB | ~6KB |
| 支持签名类型 | RSA / ECDSA / Ed25519(需PSA) | 仅 ECDSA |
| 支持加密 | RSA-OAEP / AES-KW / ECIES | 无（不支持镜像加密） |
| 裸机适配复杂度 | 需配置 mbedtls_config.h 裁剪 | 开箱即用 |

**裸机推荐配置：**
```c
#define MCUBOOT_USE_TINYCRYPT   // 零动态内存，体积小
#define MCUBOOT_SIGN_EC256      // ECDSA-P256
```

### Q2: 裸机移植能用 ED25519 吗？和 Tinycrypt 能搭配吗？

**可以，而且 Tinycrypt + ED25519 是裸机最优组合——但默认路径有一个隐藏的 Mbed TLS 依赖需要处理。**

#### 架构全景

ED25519 涉及两个独立子系统和两个独立依赖：

```
                     ┌──────────────────────────────┐
                     │     MCUBOOT_SIGN_ED25519      │
                     └──────────────┬───────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
   ┌─────────────────┐   ┌──────────────────────┐   ┌──────────────┐
   │ curve25519.c    │   │  image_ed25519.c     │   │   sha.h      │
   │ (椭圆曲线数学)   │   │  (签名验证胶水)       │   │ (镜像哈希)    │
   │                 │   │                      │   │              │
   │ SHA-512:        │   │ 公钥解析:             │   │ SHA-256:     │
   │  ├─ Mbed TLS ✅ │   │  默认: mbedtls ASN.1 │   │  可选任意后端 │
   │  └─ TinyCrypt ✅│   │  绕过: BYPASS_ASN    │   │              │
   │  (ext/tinycrypt │   │  替代: BUILTIN_KEY   │   │              │
   │   -sha512)      │   │                      │   │              │
   └─────────────────┘   └──────────────────────┘   └──────────────┘
```

#### 第一层：`curve25519.c`（ED25519 数学运算）— 没有问题

`ext/fiat/src/curve25519.c` 已经内置了 SHA-512 双路径：

```c
#if defined(MCUBOOT_USE_MBED_TLS)
#include <mbedtls/sha512.h>       // Mbed TLS 的 SHA-512
#else
#include <tinycrypt/sha512.h>      // ext/tinycrypt-sha512 扩展
#endif
```

选 Tinycrypt 后端时自动走 Tinycrypt SHA-512。这一层无 Mbed TLS 依赖。

#### 第二层：`image_ed25519.c`（签名验证胶水）— 这里有坑

```c
// boot/bootutil/src/image_ed25519.c，第 16-20 行
#if !defined(MCUBOOT_BUILTIN_KEY) && !defined(MCUBOOT_KEY_IMPORT_BYPASS_ASN)
/* We are not really using the MBEDTLS but need the ASN.1 parsing functions */
#define MBEDTLS_ASN1_PARSE_C
#include "mbedtls/oid.h"
#include "mbedtls/asn1.h"           // ← 解析公钥的 DER 编码
```

**默认路径无条件 include 了 Mbed TLS 的 ASN.1 头文件。** 原因是 `imgtool getpub` 输出的 ED25519 公钥是 DER SubjectPublicKeyInfo 格式，MCUboot 需要解析它来提取 32 字节原始公钥。

#### 三种路径对比

| 路径 | 需要 Mbed TLS? | 说明 |
|------|:---:|------|
| 默认 ED25519 | **部分** | 仅需 ASN.1 解析器（`mbedtls/asn1.h`、`mbedtls/oid.h`），不需加密/大数 |
| `MCUBOOT_KEY_IMPORT_BYPASS_ASN` | **否** | 裸机最优解：跳过 ASN.1，公钥取 DER 末尾 32 字节 |
| `MCUBOOT_BUILTIN_KEY` | **否** | 需 PSA Crypto 硬件支持，密钥完全由平台管理 |
| SHA-512 (curve25519) | **否** | 自动用 `ext/tinycrypt-sha512` 替代 |
| SHA-256 (镜像哈希) | **否** | 走 Tinycrypt 的 `tc_sha256_*` |

#### 完全零 Mbed TLS 的最优配置

```c
#define MCUBOOT_USE_TINYCRYPT              // SHA-256 由 Tinycrypt 提供
#define MCUBOOT_SIGN_ED25519               // 签名由 fiat-crypto 独立实现
#define MCUBOOT_KEY_IMPORT_BYPASS_ASN      // 去掉 Mbed TLS ASN.1 依赖
#define MCUBOOT_OVERWRITE_ONLY             // 裁剪 swap 代码
```

**构建系统添加：**
```makefile
SRC += ext/fiat/src/curve25519.c               # ED25519 数学 (fiat-crypto)
SRC += ext/tinycrypt-sha512/lib/source/sha512.c # SHA-512 (Tinycrypt 扩展)
SRC += boot/bootutil/src/image_ed25519.c       # 签名验证胶水
# 不需要 image_ecdsa.c, image_rsa.c
# 不需要 Mbed TLS 的任何 .a 或 .c
```

**`keys.c` 公钥格式：** 仍然用 `imgtool getpub -k mykey.pem` 生成 C 数组，但 bypass 路径会自动取末尾 32 字节作为原始公钥，无需手动处理。

**三种方案对比：**

| 方案 | SHA-256 | 签名 | Flash | RAM | 验证速度 (Cortex-M0+) | Mbed TLS 依赖 |
|------|---------|------|-------|-----|----------------------|:---:|
| Mbed TLS + ECDSA | Mbed TLS | Mbed TLS | ~30KB | ~3KB | 慢 (~1-2s) | 全部 |
| Tinycrypt + ECDSA | Tinycrypt | uECC | ~18KB | ~1.5KB | 中等 (~0.5s) | 无 |
| **Tinycrypt + ED25519 + BYPASS_ASN** | Tinycrypt | fiat-crypto | **~20KB** | **~2KB** | **快 (~0.15s)** | **无** |

**注意事项：**
1. 不定义 `MCUBOOT_KEY_IMPORT_BYPASS_ASN` 时，默认路径会拖入 Mbed TLS ASN.1 代码（~3-5KB Flash）
2. ED25519 公钥是 32 字节原始格式（非 DER），`keys.c` 中格式与 ECDSA 不同
3. 用 `imgtool keygen -k mykey.pem -t ed25519` 生成密钥
4. fiat-crypto 代码经过形式化验证，安全性不输 Mbed TLS，但可读性差
5. Tinycrypt 不支持镜像加密，如果同时需要 ED25519 和加密，需自实现 `boot_enc_*` 接口

### Q3: boot_go() 返回后如何跳转到用户应用?

```c
struct boot_rsp rsp;
boot_go(&rsp);

uint32_t *vectors = (uint32_t *)rsp.br_image_off;
uint32_t sp = vectors[0];
uint32_t pc = vectors[1];

__set_MSP(sp);
((void (*)(void))pc)();
```

### Q4: 镜像签名后太大怎么办?

- 检查 `--slot-size` 参数是否考虑了 trailer 大小
- 使用 Overwrite Only 模式减少 trailer 开销
- 减小 `MCUBOOT_MAX_IMG_SECTORS` 值
- 使用 `--pad-header` 而非在链接脚本中预留 header 空间

### Q5: 如何从旧版本升级 MCUboot?

MCUboot 不管理自身的升级。通常需要:
1. 将新 MCUboot 写入 Flash 起始地址
2. 使用芯片的 Dual-Bank 或 ROM Bootloader
3. 或使用外部编程器

### Q6: Swap 中断后如何恢复?

MCUboot 设计上支持在 Swap 过程中任意时刻断电。重新上电后:
1. 检查 Trailer 中的 swap_status 区域
2. 确定中断位置
3. 从断点继续 Swap

### Q7: 为什么我的镜像验证失败?

常见原因:
- Flash 写对齐 (`--align`) 设置不正确
- 公钥不匹配 (检查 `keys.c` 和签名用的私钥)
- Slot Size (`--slot-size`) 设置过小
- 镜像实际大小超过了 `slot_size - trailer_size`
- ECDSA 签名的 padding 问题 (尝试 `--pad-sig` 或 `--no-pad-sig`)

### Q8: 如何在裸机环境下分配 Mbed TLS 的内存?

```c
// 提供 calloc/free 给 Mbed TLS
#include <stdlib.h>  // 或你的自定义 malloc

int mbedtls_platform_set_calloc_free(
    calloc,   // 直接使用标准库的 calloc
    free      // 直接使用标准库的 free
);
```

如果堆不可用, 需要使用静态内存池实现 calloc/free。

### Q9: MCUboot 需要多少 RAM?

典型配置:
- Swap 模式: ~4-8KB (主要是 swap_status 和临时缓冲区)
- Overwrite Only: ~1-2KB
- 此外 Mbed TLS 需要 ~2-8KB 堆空间

RAM 使用可通过减小 `MCUBOOT_MAX_IMG_SECTORS` 来优化。

### Q10: 如何调试 MCUboot?

1. **模拟器**: 使用 `sim/` 下的 Rust 模拟器, 可以在 PC 上完整运行和调试
2. **日志**: 启用 `MCUBOOT_HAVE_LOGGING` 并映射到调试串口
3. **JTAG/SWD**: 设置断点在 `boot_go()`, `context_boot_go()`, 或具体的 swap 函数

---

## 43. 参考资源

### 官方资源
- [MCUboot 官网](http://mcuboot.com/)
- [MCUboot GitHub](https://github.com/mcu-tools/mcuboot)
- [开发者邮件列表](https://groups.io/g/MCUBoot)
- [Discord 频道](https://discord.com/invite/5PpXhvda5p)

### 相关文档
- `docs/design.md` — 完整设计文档
- `docs/PORTING.md` — 官方移植指南
- `docs/signed_images.md` — 镜像签名详解
- `docs/encrypted_images.md` — 镜像加密详解
- `docs/custom_crypto.md` — 自定义加密后端
- `docs/imgtool.md` — imgtool 工具用法
- `docs/serial_recovery.md` — 串口恢复功能
- `docs/ecdsa.md` — ECDSA 签名格式

### 示例移植
- `boot/zephyr/` — Zephyr RTOS
- `boot/mynewt/` — Apache Mynewt
- `boot/cypress/` — Cypress PSoC 6
- `boot/espressif/` — Espressif ESP32
- `boot/nuttx/` — Apache NuttX
- `samples/mcuboot_config/mcuboot_config.template.h` — 配置模板

---

> **文档版本:** 基于 MCUboot v2.4.0
> **最后更新:** 2025
> **许可:** Apache-2.0
