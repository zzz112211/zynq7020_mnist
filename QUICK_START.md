# 快速开始指南

本文档提供 ZYNQ7020 MNIST 项目的快速部署步骤。详细说明请参考 [DEPLOYMENT.md](DEPLOYMENT.md)。

## 📋 部署流程概览

```
数据准备 → 模型训练 → 生成测试数据 → FPGA 工程 → 部署运行
   ↓           ↓            ↓              ↓           ↓
MNIST      权重/偏置      测试图片      比特流+程序   FPGA 识别
数据集      .dat 文件      .dat 文件      BOOT.bin
```

## 🚀 5 步快速部署

### 1️⃣ 准备数据集

```bash
# 下载 MNIST 数据集
# 链接: https://pan.baidu.com/s/1HxMJ2SI7FLZ-le28kqAiyg 
# 提取码: 6gds

# 将文件放到 data/ 目录
data/mnist_train.csv
data/mnist_test.csv
```

### 2️⃣ 训练模型并生成权重

```bash
# 激活环境
conda activate ZyMn

# 训练神经网络
cd main
python neural_v02.py

# 输出文件将生成在 out/ 目录：
# - mnist_wih.dat (输入层->隐藏层1 权重)
# - mnist_bih.dat (隐藏层1 偏置)
# - mnist_whh.dat (隐藏层1->隐藏层2 权重)
# - mnist_bhh.dat (隐藏层2 偏置)
# - mnist_who.dat (隐藏层2->输出层 权重)
# - mnist_bho.dat (输出层 偏置)
```

### 3️⃣ 生成测试数据

```bash
# 生成随机测试图片数据
python read_image_v01.py

# 输出文件: out/a05.dat (784 个归一化像素值)
```

### 4️⃣ 准备 FPGA 工程

```bash
# 下载 Xilinx 工程文件
# 链接: https://pan.baidu.com/s/13UZWQw-jHmfFj7czWgd-iQ 
# 提取码: e2m7

# 解压到 xilinx/ 目录
```

**使用 Vivado 2020.2:**
1. 打开工程文件 `.xpr`
2. 生成比特流: `Flow Navigator → Generate Bitstream`
3. 导出硬件: `File → Export → Export Hardware` (勾选 Include bitstream)

**使用 Vitis 2020.2:**
1. 导入 `.xsa` 硬件平台
2. 创建或打开应用项目
3. 将 `out/` 目录下的数据文件嵌入到代码中
4. 编译应用程序

### 5️⃣ 部署到 FPGA

**方法 A: 通过 Vitis 调试（推荐用于开发）**
```
1. Xilinx → Program FPGA (选择 .bit 文件)
2. Debug As → Launch on Hardware
3. 在串口终端查看输出 (115200 波特率)
```

**方法 B: SD 卡启动（推荐用于部署）**
```
1. Xilinx → Create Boot Image
   - 添加 FSBL.elf
   - 添加 .bit 文件
   - 添加应用程序 .elf
   - 生成 BOOT.bin

2. 准备 SD 卡
   - 格式化为 FAT32
   - 拷贝 BOOT.bin 到根目录

3. 启动开发板
   - 插入 SD 卡
   - 设置为 SD 卡启动模式
   - 上电启动
```

---

## 📊 预期输出

串口终端应该显示：

```
MNIST Neural Network on ZYNQ7020
================================
正在加载权重参数...
加载完成！

正在加载测试图片...
图片加载完成

开始推理...
推理完成！

识别结果: 2
```

---

## ⚠️ 常见问题快速解决

### 问题 1: 训练准确率低
```
→ 增加训练 epochs (在 neural_v02.py 中修改 epochs 变量)
→ 确认数据集文件完整
```

### 问题 2: 开发板无法启动
```
→ 检查拨码开关设置
→ 确认 SD 卡格式为 FAT32
→ 验证 BOOT.bin 文件大小合理（不为 0）
```

### 问题 3: 串口无输出
```
→ 确认波特率设置为 115200
→ 检查串口号是否正确
→ 确认 USB 驱动已安装
```

### 问题 4: 数据文件格式错误
```
→ 确认 .dat 文件不为空
→ 检查文件编码（应为 ASCII）
→ 验证每行格式为: 数值,\n\r
```

---

## 📚 更多信息

- **完整部署指南**: [DEPLOYMENT.md](DEPLOYMENT.md)
- **项目说明**: [README.md](README.md)
- **参考博客**: [如何从零开始将神经网络移植到FPGA](https://blog.csdn.net/u012116328/article/details/117246023?spm=1001.2014.3001.5502)

---

## 🛠️ 环境要求清单

- [ ] Python 3.10+ 已安装
- [ ] Conda 环境已创建 (ZyMn)
- [ ] NumPy, SciPy, Matplotlib 已安装
- [ ] Xilinx 2020.2 工具链已安装 (Vivado, Vitis HLS, Vitis)
- [ ] ZYNQ7020 开发板已准备
- [ ] USB 线缆已连接
- [ ] 串口终端工具已安装
- [ ] SD 卡已准备（8GB+, FAT32）
- [ ] MNIST 数据集已下载
- [ ] Xilinx 工程文件已下载

---

**提示**: 首次部署建议预留 2-3 小时时间。遇到问题请查看 [DEPLOYMENT.md](DEPLOYMENT.md) 中的详细说明和故障排除部分。
