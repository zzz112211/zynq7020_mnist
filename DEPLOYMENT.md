# ZYNQ7020 MNIST 项目 FPGA 部署完整指南

本文档提供将 MNIST 神经网络项目部署到 ZYNQ7020 FPGA 的完整步骤说明，特别详细说明了如何准备和加载数据。

## 目录
1. [前置要求](#前置要求)
2. [环境准备](#环境准备)
3. [数据准备流程](#数据准备流程)
4. [FPGA 工程准备](#fpga-工程准备)
5. [部署步骤](#部署步骤)
6. [运行和测试](#运行和测试)
7. [常见问题](#常见问题)

---

## 前置要求

### 硬件要求
- **FPGA 开发板**: ZYNQ7020 开发板（如 PYNQ-Z2 或其他基于 ZYNQ7020 的板子）
- **连接线缆**: 
  - USB 转串口线（用于串口通信）
  - USB 数据线或 JTAG 下载器
  - 网线（如果使用网络连接）
- **电源**: 开发板配套电源适配器
- **SD 卡**: 建议 8GB 以上（用于存储启动文件和系统）

### 软件要求
- **Xilinx 开发套件** (版本 2020.2):
  - Vivado 2020.2 - 用于综合和实现硬件设计
  - Vitis HLS 2020.2 - 用于高层次综合
  - Vitis 2020.2 - 用于软件开发
- **Python 环境** (建议 Python 3.10):
  - NumPy
  - SciPy
  - Matplotlib
- **串口调试工具**: 
  - Windows: PuTTY 或 Tera Term
  - Linux: Minicom 或 screen
- **SD 卡格式化工具**: SD Card Formatter

---

## 环境准备

### 1. Python 环境配置

```bash
# 创建虚拟环境
conda create -n ZyMn python=3.10

# 激活环境
conda activate ZyMn

# 安装必要的库
pip install numpy scipy matplotlib
```

### 2. Xilinx 工具安装

1. 从 Xilinx 官网下载 Vitis 2020.2 统一安装包
2. 安装时选择完整安装（包含 Vivado、Vitis HLS 和 Vitis）
3. 安装完成后，配置环境变量：

**Windows:**
```cmd
# 在系统环境变量中添加 Xilinx 工具路径
C:\Xilinx\Vitis\2020.2\bin
C:\Xilinx\Vivado\2020.2\bin
```

**Linux:**
```bash
# 在 ~/.bashrc 中添加
source /tools/Xilinx/Vitis/2020.2/settings64.sh
```

### 3. 获取项目文件

```bash
# 克隆项目仓库
git clone https://github.com/zzz112211/zynq7020_mnist.git
cd zynq7020_mnist
```

---

## 数据准备流程

这是部署的关键步骤，需要准备以下数据文件：

### 1. 获取 MNIST 数据集

数据集下载链接已提供在 `data/mnist数据集_百度网盘.txt` 文件中：
- 链接: https://pan.baidu.com/s/1HxMJ2SI7FLZ-le28kqAiyg 
- 提取码: 6gds

下载后将以下文件放到 `data/` 目录：
- `mnist_train.csv` - 训练数据集（60,000 个样本）
- `mnist_test.csv` - 测试数据集（10,000 个样本）

### 2. 训练神经网络并生成权重参数

运行训练脚本生成网络权重和偏置参数：

```bash
cd main
python neural_v02.py
```

**这个脚本会做什么？**
- 加载 MNIST 训练数据（60,000 个样本）
- 训练一个三层神经网络：
  - 输入层：784 个节点（28x28 像素图片）
  - 第一隐藏层：64 个节点
  - 第二隐藏层：32 个节点
  - 输出层：10 个节点（数字 0-9）
- 训练 15 个 epoch
- 在测试集上验证准确率
- **生成 6 个参数文件**到 `out/` 目录：
  - `mnist_wih.dat` - 输入层到第一隐藏层的权重（784×64）
  - `mnist_bih.dat` - 第一隐藏层的偏置（64×1）
  - `mnist_whh.dat` - 第一隐藏层到第二隐藏层的权重（64×32）
  - `mnist_bhh.dat` - 第二隐藏层的偏置（32×1）
  - `mnist_who.dat` - 第二隐藏层到输出层的权重（32×10）
  - `mnist_bho.dat` - 输出层的偏置（10×1）

**训练预期输出：**
```
performance =  0.9XXX  (准确率应在 90% 以上)
```

### 3. 生成测试图片数据

运行脚本生成随机测试图片数据：

```bash
python read_image_v01.py
```

**这个脚本会做什么？**
- 从 `mnist_test.csv` 中随机选择一张图片（前 10 张之一）
- 将像素值归一化（除以 255，转换为 0-1 之间的浮点数）
- 生成测试数据文件：`out/a05.dat`（包含 784 个归一化的像素值）

### 4. 数据文件说明

生成的数据文件格式说明：

**权重/偏置文件格式** (`*.dat`)：
```
0.123456,
0.234567,
0.345678,
...
```
每个值后面都有 `,\n\r` 分隔符。

**测试图片数据文件** (`a05.dat`)：
- 包含 784 个浮点数（28×28 像素）
- 每个值范围在 0.0 到 1.0 之间
- 格式同上

---

## FPGA 工程准备

### 1. 获取 Xilinx 工程文件

Xilinx 工程文件下载链接在 `xilinx/xilinx工程_百度网盘.txt`：
- 链接: https://pan.baidu.com/s/13UZWQw-jHmfFj7czWgd-iQ 
- 提取码: e2m7

下载后解压到 `xilinx/` 目录下。

### 2. 工程结构说明

典型的 ZYNQ 工程包含三个部分：

#### a) HLS 工程 (High-Level Synthesis)
- **作用**: 将 C/C++ 代码转换为硬件 IP 核
- **输入**: 神经网络推理的 C/C++ 实现
- **输出**: IP 核（.zip 文件）

#### b) Vivado 工程 (硬件设计)
- **作用**: 集成 PS（Processing System）和 PL（Programmable Logic）
- **包含组件**:
  - ZYNQ PS：ARM 处理器部分
  - 神经网络 IP 核（由 HLS 生成）
  - AXI 总线互连
  - 其他外设 IP（如 GPIO、UART 等）
- **输出**: 硬件比特流文件 (`.bit`) 和硬件描述文件 (`.xsa`)

#### c) Vitis 工程 (软件开发)
- **作用**: 开发运行在 ARM 处理器上的应用程序
- **功能**:
  - 初始化硬件
  - 将数据（权重、偏置、测试图片）加载到 FPGA
  - 触发神经网络推理
  - 读取并显示结果
- **输出**: 可执行文件 (`.elf`)

---

## 部署步骤

### 第一步：准备 HLS IP 核（如果需要修改）

如果下载的工程已包含 IP 核，可跳过此步骤。

1. 打开 Vitis HLS 2020.2
2. 打开 HLS 工程（通常在 `xilinx/hls/` 目录）
3. 在源代码中确认神经网络参数与你的训练结果一致
4. 运行 C 仿真验证功能
5. 运行综合 (Synthesis)
6. 运行 C/RTL 协同仿真（可选）
7. 导出 IP 核：Solution → Export RTL

### 第二步：配置 Vivado 硬件工程

1. **打开 Vivado 工程**
   ```
   Vivado 2020.2 → Open Project → 选择工程文件 (.xpr)
   ```

2. **更新 IP 核（如果重新生成了 HLS IP）**
   - 在 IP Integrator 中删除旧的神经网络 IP
   - 添加新生成的 IP 核
   - 重新连接 AXI 总线

3. **检查设计**
   - 验证 Block Design 中所有连接正确
   - 确认时钟和复位信号配置正确
   - 验证地址分配无冲突

4. **综合和实现**
   ```
   Flow Navigator → Synthesis → Run Synthesis
   等待完成后：
   Flow Navigator → Implementation → Run Implementation
   等待完成后：
   Flow Navigator → Generate Bitstream
   ```

5. **导出硬件平台**
   ```
   File → Export → Export Hardware
   勾选 "Include bitstream"
   保存为 .xsa 文件（例如：design_1_wrapper.xsa）
   ```

### 第三步：开发 Vitis 应用程序

1. **启动 Vitis 2020.2**
   ```
   选择或创建工作空间（workspace）
   ```

2. **创建平台项目**（如果工程未包含）
   ```
   File → New → Platform Project
   导入之前生成的 .xsa 文件
   ```

3. **创建应用项目**（或打开现有项目）
   ```
   File → New → Application Project
   选择刚创建的平台
   选择模板（通常选择 "Empty Application" 或 "Hello World"）
   ```

4. **添加应用代码**
   
   主要包含以下功能：

   **a) 初始化代码** (`main.c`)
   ```c
   #include <stdio.h>
   #include "platform.h"
   #include "xil_printf.h"
   #include "xparameters.h"
   // 包含神经网络 IP 核的头文件
   
   int main() {
       init_platform();
       
       // 1. 加载权重和偏置到 FPGA
       load_weights();
       
       // 2. 加载测试图片数据
       load_test_image();
       
       // 3. 启动神经网络推理
       start_inference();
       
       // 4. 读取结果
       int result = read_result();
       printf("识别结果: %d\n", result);
       
       cleanup_platform();
       return 0;
   }
   ```

   **b) 数据加载函数**
   
   关键是如何将 `out/` 目录下的数据文件加载到 FPGA：

   **方法 1：编译时嵌入（推荐用于小数据）**
   ```c
   // 将 .dat 文件内容转换为 C 数组
   float wih_data[] = {
       #include "../../../out/mnist_wih.dat"  // 直接包含数据文件
   };
   
   float bih_data[] = {
       #include "../../../out/mnist_bih.dat"
   };
   
   // ... 其他权重和偏置
   
   void load_weights() {
       // 将数据写入到神经网络 IP 核的内存地址
       u32 *base_addr = (u32 *)XPAR_NN_IP_0_BASEADDR;
       
       // 写入权重 wih
       for(int i = 0; i < WIH_SIZE; i++) {
           Xil_Out32(base_addr + WIH_OFFSET + i*4, 
                    *(u32*)&wih_data[i]);
       }
       
       // 类似地写入其他参数...
   }
   ```

   **方法 2：从 SD 卡读取（推荐用于大数据）**
   ```c
   #include "ff.h"  // FatFS 文件系统
   
   void load_weights_from_sd() {
       FATFS fs;
       FIL fil;
       FRESULT res;
       UINT br;
       
       // 挂载 SD 卡
       res = f_mount(&fs, "0:/", 1);
       
       // 读取权重文件
       res = f_open(&fil, "mnist_wih.dat", FA_READ);
       float buffer[784*64];
       f_read(&fil, buffer, sizeof(buffer), &br);
       f_close(&fil);
       
       // 写入到硬件
       // ...
   }
   ```

   **方法 3：通过串口传输（用于调试）**
   ```c
   void receive_data_via_uart() {
       // 通过 UART 接收 PC 发送的数据
       // 适合调试阶段
   }
   ```

5. **配置编译选项**
   - 确保包含路径正确
   - 设置优化级别（-O2 或 -O3）
   - 如果使用 SD 卡，添加 FatFS 库

6. **编译应用程序**
   ```
   Project → Build Project
   ```

### 第四步：硬件连接

1. **连接开发板**
   - 使用 USB 线连接 JTAG/UART 端口到 PC
   - 连接电源（确保拨码开关设置为正确的启动模式）
   - 如果使用 SD 卡：
     - 格式化 SD 卡为 FAT32
     - 将必要的启动文件（BOOT.bin）和数据文件拷贝到 SD 卡
     - 插入 SD 卡到开发板

2. **检查连接**
   - 在设备管理器（Windows）或 dmesg（Linux）中确认串口设备
   - 记下串口号（如 COM3 或 /dev/ttyUSB0）

### 第五步：下载和运行

#### 方式 A：使用 Vitis 调试（开发阶段推荐）

1. **配置调试设置**
   ```
   右键应用项目 → Debug As → Debug Configurations
   创建新的 "System Debugger using Debug_xxx.elf" 配置
   ```

2. **连接硬件**
   ```
   Xilinx → Program FPGA
   选择 .bit 文件
   点击 Program
   ```

3. **下载并运行程序**
   ```
   点击 Debug 按钮
   程序会自动下载到开发板并停在 main 入口
   
   使用调试控制：
   - F8 (Resume) 继续运行
   - F6 (Step Over) 单步执行
   - F5 (Step Into) 进入函数
   ```

4. **查看输出**
   - 在 Vitis 的 Console 窗口查看 printf 输出
   - 或者使用串口终端工具连接（波特率通常 115200）

#### 方式 B：生成启动文件（部署阶段推荐）

1. **创建 BOOT.bin**
   ```
   Xilinx → Create Boot Image
   
   添加以下文件（按顺序）：
   1. BootLoader (FSBL.elf) - 第一级引导加载程序
   2. Bitstream (.bit) - FPGA 配置文件
   3. Application (.elf) - 你的应用程序
   
   输出文件：BOOT.bin
   ```

2. **准备 SD 卡**
   ```
   - 格式化 SD 卡为 FAT32
   - 将 BOOT.bin 拷贝到 SD 卡根目录
   - 将数据文件（可选，如果程序需要从 SD 读取）拷贝到 SD 卡
   ```

3. **启动**
   ```
   - 将 SD 卡插入开发板
   - 设置拨码开关为 SD 卡启动模式（参考开发板手册）
   - 上电
   - 开发板会自动从 SD 卡启动并运行程序
   ```

---

## 运行和测试

### 1. 串口连接

**Windows (使用 PuTTY):**
```
- Serial line: COM3 (根据实际情况)
- Speed: 115200
- Connection type: Serial
- Flow control: None
```

**Linux (使用 minicom):**
```bash
sudo minicom -D /dev/ttyUSB0 -b 115200
```

### 2. 预期输出

程序正常运行时，串口输出应该类似：
```
MNIST Neural Network on ZYNQ7020
================================
正在加载权重参数...
加载 wih: 完成 (784x64)
加载 bih: 完成 (64)
加载 whh: 完成 (64x32)
加载 bhh: 完成 (32)
加载 who: 完成 (32x10)
加载 bho: 完成 (10)

正在加载测试图片...
图片加载完成 (784 像素)

开始推理...
推理完成！

输出结果：
[0]: 0.01
[1]: 0.02
[2]: 0.95  <-- 最大值
[3]: 0.01
...

识别结果: 2
```

### 3. 验证结果

- 检查识别结果是否与测试图片的真实标签一致
- 可以修改 `read_image_v01.py` 脚本来测试不同的图片
- 多次运行以验证稳定性

---

## 常见问题

### Q1: 如何确认数据已正确加载到 FPGA？

**A1:** 在 Vitis 应用程序中添加验证代码：
```c
void verify_weights() {
    u32 *base_addr = (u32 *)XPAR_NN_IP_0_BASEADDR;
    
    // 读回第一个权重值
    u32 read_val = Xil_In32(base_addr + WIH_OFFSET);
    float read_float = *(float*)&read_val;
    
    printf("第一个权重值: %f (期望: %f)\n", 
           read_float, wih_data[0]);
}
```

### Q2: 数据文件太大，如何优化？

**A2:** 
- 使用定点数代替浮点数（将 float 转换为 fixed-point）
- 量化权重（例如 8-bit 量化）
- 使用压缩算法
- 考虑稀疏化网络（剪枝）

### Q3: 出现内存崩溃怎么办？

**A3:** 根据 `xilinx工程_README.txt` 提到的问题：
- 检查内存分配是否超出范围
- 使用动态内存分配时确保正确释放
- 检查 AXI 总线访问地址是否正确
- 增加 DDR 内存大小配置
- 在 Vivado Block Design 中检查地址映射

### Q4: 如何修改测试的图片？

**A4:** 修改 `main/read_image_v01.py`：
```python
# 原代码：随机选择 1-10 行
random_row_index = random.randint(1, 10) - 1

# 修改为指定某一行，例如第 5 行（数字标签在第一列）
random_row_index = 4  # 固定选择第 5 行
```

### Q5: 如何查看推理耗时？

**A5:** 在应用程序中添加计时代码：
```c
#include "xtime_l.h"

XTime tStart, tEnd;

XTime_GetTime(&tStart);
start_inference();
XTime_GetTime(&tEnd);

printf("推理耗时: %llu 时钟周期\n", 
       2*(tEnd - tStart));
printf("推理耗时: %.2f ms\n", 
       1000.0 * (tEnd - tStart) / COUNTS_PER_SECOND);
```

### Q6: 如何提高识别准确率？

**A6:**
- 增加训练 epoch 数
- 调整学习率
- 增加隐藏层节点数
- 使用数据增强
- 确保数据归一化方式一致（训练和推理）

### Q7: 开发板无法启动怎么办？

**A7:** 检查清单：
- 拨码开关是否设置正确
- SD 卡是否正确格式化（FAT32）
- BOOT.bin 文件是否完整
- 电源是否稳定（建议使用原装电源）
- 尝试 JTAG 模式启动进行调试

### Q8: 数据格式转换问题

**A8:** Python 生成的 `.dat` 文件格式：
```
0.123456,
0.234567,
```

在 C 代码中直接包含时需要：
```c
float data[] = {
    #include "data.dat"
    0  // 添加一个结束符（因为最后有逗号）
};
```

或者修改 Python 脚本不在最后一个值添加逗号。

---

## 数据流程总结

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 数据准备阶段 (PC 端 - Python)                             │
├─────────────────────────────────────────────────────────────┤
│  mnist_train.csv  ──┐                                        │
│  mnist_test.csv   ──┼──> neural_v02.py                       │
│                     │         │                              │
│                     │         ├──> out/mnist_wih.dat         │
│                     │         ├──> out/mnist_bih.dat         │
│                     │         ├──> out/mnist_whh.dat         │
│                     │         ├──> out/mnist_bhh.dat         │
│                     │         ├──> out/mnist_who.dat         │
│                     │         └──> out/mnist_bho.dat         │
│                     │                                         │
│                     └──> read_image_v01.py ──> out/a05.dat   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 硬件实现阶段 (Xilinx 工具链)                              │
├─────────────────────────────────────────────────────────────┤
│  HLS (C/C++) ──> IP 核 (.zip)                                │
│                    │                                         │
│                    ▼                                         │
│  Vivado ──> 集成设计 ──> Bitstream (.bit) + 硬件描述 (.xsa)  │
│                    │                                         │
│                    ▼                                         │
│  Vitis ──> 应用程序 (.elf)                                   │
│             ├── 嵌入数据（编译时）                           │
│             │   或                                           │
│             └── 从 SD 卡读取（运行时）                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 部署阶段 (FPGA 开发板)                                    │
├─────────────────────────────────────────────────────────────┤
│  BOOT.bin (FSBL + Bitstream + Application)                  │
│      │                                                       │
│      └──> SD 卡 ──> ZYNQ7020 开发板                          │
│                          │                                   │
│                          ├──> 配置 FPGA (PL)                 │
│                          ├──> 启动 ARM (PS)                  │
│                          ├──> 加载权重/偏置到硬件            │
│                          ├──> 加载测试图片                   │
│                          ├──> 执行推理                       │
│                          └──> 输出结果                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 参考资源

1. **Xilinx 官方文档**:
   - UG902 - Vivado Design Suite User Guide: High-Level Synthesis
   - UG1144 - Vitis Unified Software Platform Documentation
   - UG585 - Zynq-7000 Technical Reference Manual

2. **项目参考**:
   - [如何从零开始将神经网络移植到FPGA(ZYNQ7020)加速](https://blog.csdn.net/u012116328/article/details/117246023?spm=1001.2014.3001.5502)
   - [mnist-nnet-hls-zynq7020-fpga](https://github.com/doveyour/mnist-nnet-hls-zynq7020-fpga)

3. **开发板文档**:
   - 查看你的具体开发板用户手册
   - 启动模式配置
   - 引脚定义

---

## 技术支持

如有问题，建议：
1. 检查本文档的常见问题部分
2. 查看项目 Issues 页面
3. 参考 Xilinx 官方论坛
4. 提交新的 Issue 描述你的问题

---

**最后更新**: 2025-10-17
**适用版本**: Xilinx 2020.2
