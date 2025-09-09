# 
本文记录zynqImage启动文件组成部分

## 启动镜像的组成

 <div align="center">
<img src="image/startImage.png " width="70%">
<p>图片4.SIMD&FP寄存器</p>
</div>

启动文件有以下几个重要组成部分：
1. 启动报头
2. 镜像报头
3. 分区报头

### 启动报头

### 表9: Zynq UltraScale+ MPSoC器件启动报头

| 地址偏移 | 参数 | 描述 |
|:---:|:---:|:---:|
| 0x00-0x1F | Arm® vector table | XIP ELF 矢量表: <br>0xEAEFFFFE: 对应于 Cortex®-R5F 和 Cortex A53 (32 位)<br>0x14000000: 对应于 Cortex A53 (64 位) |
| 0x20 | Width Detection Word | 该字段用于 QSPI 宽度检测。`0xaa995566` (小端字节序格式)。|
| 0x24 | Header Signature | 包含4字节的 "X"、"N"、"L"、"X" (按字节顺序)，按小端字节序格式为 `0x584c4e58`。|
| 0x28 | Key Source | `0x00000000` (未加密)<br>`0xa5c3c5a5` (eFUSE 中存储的黑密钥)<br>`0xa5c3c5a7` (eFUSE 中存储的模糊密钥)<br>`0xa5a3c5c5a` (BBRAM 中存储的红密钥)<br>`0xa5c3c5a3` (eFUSE 中存储的 eFUSE 红密钥)<br>`0xa5c3c7ca5` (启动报头中存储的模糊密钥)<br>`0xa5a3c3c5` (启动报头中存储的用户密钥)<br>`0xa5c3c7c53` (启动报头中存储的黑密钥)|
| 0x2C | FSBL Execution address (RAM) | OCM 或 XIP 基址中的 FSBL 执行地址。|
| 0x30 | Source Offset | 如无 PMUFW，则这是 FSBL 的起始偏移。如有 PMUFW，则为 PMUFW 的起始偏移。|
| 0x34 | PMU Image Length | PMUFW 镜像长度 (以字节为单位)。 (0-128 KB)。|
| 0x38 | Total PMU FW Length | PMUFW 镜像总长 (以字节为单位)。(PMUFW 长度 + 加密开销)|
| 0x3C | FSBL Image Length | 原始 FSBL 镜像长度 (以字节为单位)。(0-250 KB)。如为0，则使用 XIP 启动镜像。|
| 0x40 | Total FSBL Length | FSBL 镜像长度 + FSBL 镜像的加密开销 + 身份验证。证书 + 64字节对齐 + 散列大小 (完整性检查)。|
| 0x44 | FSBL Image Attributes | 请参阅**属性**。|
| 0x48 | Boot Header Checksum | 按标准算法，从偏移 `0x20` 到 `0x44` (含) 计算所得的字数总和的反码。这些字假定按小端字节序。|
| 0x4C-0x6C | Obfuscated/Black Key Storage | 存储模糊密钥或黑密钥。|
| 0x6C | Shutter Value | 32 位 PUF_SHUT 寄存器值，用于配置 PUF 的快门偏移时间和快门打开时间。|

---



| 地址偏移 | 参数 | 描述 |
|:---:|:---:|:---:|
| 0x70-0x94 | User-Defined Fields (UDF) | 40 字节。|
| 0x98 | Image Header Table Offset | 指向镜像报头表的指针。|
| 0x9C | Partition Header Table Offset | 指向分区报头表的指针。|
| 0xA0-0xA8 | Secure Header IV | 用于启动加载程序分区的安全报头的 IV。|
| 0xAC-0xB4 | Obfuscated/Black Key IV | 用于模糊密钥或黑密钥的 IV。|