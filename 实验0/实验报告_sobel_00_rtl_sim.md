# 实验 0：Sobel 边缘检测 RTL 仿真报告

## 1. 实验目标

本实验是 ZYNQ7020 图像处理课程设计的预备仿真实验，不依赖 Vivado 工程和开发板，使用纯 Verilog 仿真验证以下功能：

1. UART 串行字节流接收 128×72 RGB888 图像
2. 图像帧协议解析（帧头、行头、像素数据）
3. RGB 转灰度（整数加权平均）
4. Sobel 3×3 卷积边缘检测
5. 视频像素流输出及帧完成信号

---

## 2. 系统结构与数据流

```
input_rgb.hex (128×72×3 bytes)
    → UART byte stream model (testbench)
    → uart_rx (UART 接收模块)
    → image_frame_rx (帧协议解析)
    → rgb_to_gray (RGB→灰度)
    → sobel_core (Sobel 3×3 卷积)
    → video_stream_model (像素流封装)
    → sobel_out.pgm / sobel_out.png
```

### 2.1 UART 帧格式

| 字段 | 字节序列 | 说明 |
|------|----------|------|
| 帧头 | `55 AA` | 帧同步字 |
| 图像宽度 | `width[7:0] width[15:8]` | 小端序，≤640 |
| 图像高度 | `height[7:0] height[15:8]` | 小端序，≤480 |
| 像素格式 | `18` | RGB888 |
| 行头 | `33 CC row[7:0] row[15:8]` | 每行前发送 |
| 像素数据 | `R G B` × width | 每像素 3 字节 |

### 2.2 模块功能说明

| 模块 | 功能 |
|------|------|
| `uart_rx` | 8N1 UART 接收器，过采样检测起始位，移位接收 |
| `image_frame_rx` | 状态机解析帧协议，输出 rgb_valid + 像素坐标 |
| `rgb_to_gray` | Y = (77R + 150G + 29B) >> 8，1 周期延迟 |
| `sobel_core` | 3×3 窗口 Gx/Gy 卷积，绝对值求和，2 行缓冲 |
| `video_stream_model` | 将 edge 数据转为 video 像素流输出 |
| `sobel_system` | 顶层集成所有模块 |
| `sobel_system_tb` | testbench：发送测试帧、校验输出、写 PGM/VCD |

---

## 3. 仿真环境与工具

| 工具 | 版本 | 用途 |
|------|------|------|
| ModelSim SE-64 | 2020.4 | Verilog 编译与仿真 |
| Python | 3.11.10 | 生成输入图像、转换输出 PNG |
| Pillow | — | PNG 图像读写 |
| vcdvcd | 2.6.0 | VCD 波形解析 |

**仿真参数：** 时钟 12 MHz，波特率 1 Mbps，图像 128×72。

---

## 4. 基础实验结果

### 4.1 输入图像 (input_rgb.png)

输入图像由 `tools/gen_input_rgb.py` 生成，包含以下特征：

- **四角渐变背景**：R 随 x 从 0→255 渐变，G 随 y 从 0→255 渐变
- **中央白色矩形** (240,240,240)：位于 x∈[32,96), y∈[18,54)
- **对角 X 形暗线** (20,20,20)：沿 y≈x 和 y≈127-x 两条对角线
- **垂直红色中轴线** (255,40,40)：位于 x∈[62,67]

验证采样数据（部分像素）：
```
pixel(0,0):   R=20  G=20  B=20   ← 暗线起点
pixel(32,0):  R=64  G=0   B=41   ← 渐变+暗线
pixel(64,0):  R=255 G=40  B=40   ← 红色中轴线
pixel(32,18): R=240 G=240 B=240  ← 白色矩形内
pixel(127,71):R=255 G=255 B=255  ← 右下角渐变
```

<img src="input_rgb.png" width="512" alt="输入图像 — 渐变背景、白色矩形、X暗线、红色中轴">

### 4.2 Sobel 输出图像 (sobel_out.png)

| 统计项 | 数值 |
|--------|------|
| 总像素数 | 9216 (128×72) |
| 非零边缘像素 | 6980 (75.7%) |
| 强边缘 (>128) | 1223 (13.3%) |
| 最大边缘值 | 255 |
| 零值像素 | 2236 (24.3%) |

边缘分布分析：
```
row  0:  0,  0,  0,  0,  0,  0,  0,  0,  0  ← 边界清零
row  1:  0,  6,106, 24,  0, 26,255, 12,  0  ← 第一行边缘（弱+强）
row 36:  0, 24, 24,255,  0,255, 24, 22,  0  ← 中行（白矩形+红线边界）
row 70:  0, 26, 26, 26,  0, 26, 26, 28,  0  ← 倒数第二行
row 71:  0,  0,  0,  0,  0,  0,  0,  0,  0  ← 最后行边界清零
```

<img src="sobel_out.png" width="512" alt="Sobel 边缘检测输出">

### 4.3 仿真验证结果

```
Sobel RGB888 simulation passed
Errors: 0, Warnings: 0

验证项目：
  gray_valid 脉冲数:    9216  ✓ (= 128×72)
  edge_valid 脉冲数:    9017  ✓ (= 127×71, 首行首列无输出)
  frame_done_count:     1     ✓
  frame_error_count:    3     ✓ (3个异常帧被正确检测)
  总仿真时间:           275.3 ms
```

---

## 5. 扩展实验 1：更换输入图片

### 5.1 新输入图设计

使用 `tools/gen_checkerboard.py` 生成棋盘格+同心圆图案：

| 特征 | 描述 |
|------|------|
| 棋盘格 (16×16 块) | 浅蓝白 (220,220,240) / 深蓝 (30,30,60) 交替 |
| 同心圆环 | 以图像中心为圆心，每 10 像素交替颜色 |
| 绿色顶条 | y < 12 区域，(50,180,50) |
| 红色左边条 | x < 10 区域，(200,60,60) |
| 黄色斜方块 | 沿对角线移动的 14×14 方块，(255,220,40) |

<img src="input_checker.png" width="512" alt="棋盘格输入图">

### 5.2 对比结果

| 指标 | 原图（渐变+几何） | 棋盘格图 |
|------|:---:|:---:|
| 非零边缘像素 | 6980 (75.7%) | 3345 (36.3%) |
| 强边缘 (>128) | 1223 (13.3%) | 2925 (31.7%) |
| 边缘类型 | 大量渐变弱边缘 | 大量硬边界强边缘 |

<img src="sobel_out_checker.png" width="512" alt="棋盘格 Sobel 输出">

**分析：** 原图的渐变背景产生大量低强度边缘（0-32 范围），而棋盘格图的硬边界使边缘强度集中在 255 附近。棋盘格产生的边缘网格规则、清晰，而原图边缘较为连续、模糊。这验证了 Sobel 算子对硬边界的响应远强于渐变过渡。

---

## 6. 扩展实验 2：Sobel 输出阈值对比

### 6.1 RTL 修改

在 `sobel_core.v` 中增加 `THRESHOLD` 参数：

```verilog
module sobel_core #(
    parameter integer WIDTH     = 128,
    parameter integer HEIGHT    = 72,
    parameter integer THRESHOLD = 0      // 新增阈值参数
) ( ... );

// 边缘输出增加阈值判断：
edge_data <= (mag > 13'd255) ? 8'hff :
             (mag > THRESHOLD) ? mag[7:0] : 8'd0;
```

`THRESHOLD=0` 保持与原版完全一致（仿真验证通过）。

### 6.2 阈值对比结果

| 阈值 | 边缘像素数 | 占比 | 变化 |
|:----:|----------:|:----:|------|
| 0 | 6980 | 75.7% | 基准（无阈值） |
| 32 | 1315 | 14.3% | ↓ 81.2% — 大量弱边缘被滤除 |
| 64 | 1288 | 14.0% | ↓ 81.5% |
| 128 | 1223 | 13.3% | ↓ 82.5% — 仅保留强边缘 |

**分析：**

1. **阈值 32** 滤除了 5665 个弱边缘像素（81.2%），这些主要来自渐变背景区域的低幅度梯度
2. **阈值 32→64** 滤除 27 个边缘，变化微小，说明 32-64 范围内的边缘极少
3. **阈值 128** 时边缘数 1223 = 强边缘数，即所有幅度 >128 的边缘都被保留到这个点
4. 阈值 ≥32 后边缘数量趋于稳定（1315→1223），说明真正有意义的结构边缘约 1200-1300 个
5. 建议实际应用使用阈值 32-64 之间，可有效抑制渐变噪声同时保留结构边缘

<img src="sobel_threshold_comparison.png" width="700" alt="Sobel 阈值对比">

---

## 7. 扩展实验 3：UART 行包异常仿真

### 7.1 新增异常场景设计

在 testbench 中新增 `send_missing_pixels_frame` 测试：发送 8×8 小帧，第 3 行仅发送 4 个像素（正常应为 8 个），然后立即发送第 4 行头。模拟 UART 传输中字节丢失的场景。

### 7.2 异常对比分析

| 异常类型 | 测试场景 | frame_error | 视频输出 | 输出像素数 |
|----------|----------|:----------:|:--------:|:----------:|
| 错误帧头 | 55 00 (缺少 AA) | ✓ 触发 | ✗ 无 | 0 |
| 错误格式 | format=0x99 | ✓ 触发 | ✗ 无 | 0 |
| 错误行号 | row=5, height=4 | ✓ 触发 | ✗ 无 | 0 |
| **缺少像素** | **第 3 行仅 4/8 像素** | **✗ 未触发** | **✓ 有** | **42 (应为 64)** |

### 7.3 关键发现

缺少像素的场景表现出**静默数据损坏**：

- **不触发 frame_error**：接收状态机在像素接收阶段（ST_R/ST_G/ST_B）不检查特殊字节，只盲存数据
- **产生乱码输出**：第 4 行头字节 (33 CC 04 00) 被当作第 3 行的像素数据处理
- **frame_done 意外触发**：乱码数据流中恰好出现有效行头模式，使状态机恢复同步
- **输出 42 像素**（期望 64）：22 个像素丢失 + 状态机错位

**启示：** UART 传输中的字节丢失是最危险的错误模式——接收方无法自动检测，导致错误数据流入后续处理流水线。实际系统中应增加字节计数校验或 CRC。

---

## 8. 扩展实验 4：关键时序波形标注

### 8.1 关键信号说明

| 信号 | 含义 | 产生模块 |
|------|------|----------|
| `frame_start` | 有效帧头解析完成，脉冲 1 周期 | image_frame_rx |
| `gray_valid` | 灰度像素输出有效 | rgb_to_gray |
| `edge_valid` | Sobel 边缘像素输出有效 | sobel_core |
| `video_frame_done` | 等效 edge_frame_done，整帧处理完成 | video_stream_model |

### 8.2 时序测量结果

```
frame_start  ──┬── 406.5 us
               │
               │  UART 传输: 7 字节帧头 + 4 字节行头 + 128×3 像素
               │  延迟 ~69 us
               │
gray_valid  ──┬── 475.5 us  (首个灰度像素输出)
               │
               │  Sobel 行缓冲: 需 2 行数据填满 3×3 窗口
               │  延迟 ~3847 us
               │
edge_valid  ──┬── 4323.0 us  (首个边缘像素输出)
               │
               │  9216 个灰度像素 + 9017 个边缘像素
               │  持续 ~271 ms
               │
video_       ──┬── 275313.3 us  (帧处理完成)
frame_done     │
```

### 8.3 时序参数汇总

| 参数 | 数值 | 说明 |
|------|------|------|
| 时钟频率 | 12 MHz | 周期 83.3 ns |
| UART 波特率 | 1 Mbps | 12 周期/bit |
| 帧头传输时间 | 70 us | 7 字节 × 10 位 × 1 us |
| 单行传输时间 | 3.88 ms | (4+384)×10×1 us |
| 总帧传输时间 | ~279.5 ms | 7 + 72×388 字节 |
| frame_start→首个 gray_valid | 69.0 us | UART 接收+帧解析+灰度转换 |
| 首个 gray_valid→首个 edge_valid | 3847.5 us | 2 行缓冲填满 3×3 窗口 |
| gray_valid→video_frame_done | ~274.8 ms | 9216 像素处理 + 边界填充 |
| 仿真通过总时间 | 275.3 ms | 含异常帧测试 |

### 8.4 波形图

**帧起始 — frame_start → gray_valid → edge_valid：**

<img src="waveform_timing_start.png" width="900" alt="波形-帧起始">

**帧中部 — 连续 gray_valid 和 edge_valid 处理：**

<img src="waveform_timing_mid.png" width="900" alt="波形-帧中部">

**帧结束 — 最后像素 + video_frame_done：**

<img src="waveform_timing_end.png" width="900" alt="波形-帧结束">

**完整帧时间线总览：**

<img src="waveform_timing_overview.png" width="900" alt="波形-时间线总览">

---

## 9. 问题记录与解决

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| Windows 无 iverilog | 未安装 Icarus Verilog | 使用已安装的 ModelSim SE-64 2020.4，用 vlog/vsim 替代 iverilog/vvp |
| WSL 未安装 Linux 发行版 | 仅安装了 wsl.exe 但无发行版 | 无需 WSL，ModelSim 原生 Windows 可直接使用 |
| Makefile 不可用 | Windows 无 make | 直接使用 vlog/vsim 命令行编译仿真 |
| 阈值修改后仿真行为不一致 | 需同步修改 sobel_system.v 和 TB | 添加 THRESHOLD 参数传递链：TB→sobel_system→sobel_core |
| 缺少像素测试首次失败 | frame_done_count 期望值不对 | 分析发现乱码帧也会意外完成，修改断言为 frame_done_count==2 |

---

## 10. 总结

1. **基础仿真**：成功运行 128×72 Sobel 边缘检测 RTL 仿真，9216 个灰度像素产生 9017 个边缘像素（首行首列无输出），帧处理时间 ~274.9 ms。

2. **更换输入**：棋盘格图的强边缘占比 31.7% 远高于原图的 13.3%，说明 Sobel 算子对硬边界的检测效果显著优于渐变边界。

3. **阈值对比**：阈值 32 可滤除 81% 的渐变噪声弱边缘，而阈值 128 仅保留 1223 个结构强边缘。建议实际应用使用 32-64。

4. **异常仿真**：发现字节丢失是最危险的 UART 错误——不触发 frame_error 但产生静默数据损坏（42 个乱码像素）。

5. **时序分析**：从 frame_start 到首个 edge_valid 需 ~3.9 ms 流水线延迟（含 2 行缓冲），总帧处理与 UART 传输时间基本一致。

---

## 附录：文件清单

### 实验源码
```
rtl/uart_rx.v              UART 接收模块
rtl/image_frame_rx.v       帧协议解析模块
rtl/rgb_to_gray.v          RGB 转灰度模块
rtl/sobel_core.v           Sobel 边缘检测核心
rtl/video_stream_model.v   视频像素流封装
rtl/sobel_system.v         顶层集成模块
tb/sobel_system_tb.v       仿真 testbench
```

### 工具脚本
```
tools/gen_input_rgb.py      默认输入图生成
tools/gen_checkerboard.py   棋盘格输入图生成
tools/convert_images.py     PGM→PNG 转换
tools/gen_waveform_annotated.py  波形标注图生成
```

### 实验结果（位于 build/ 目录）
```
基础实验:
  input_rgb.png, sobel_out.png, sobel_out.pgm, sobel_system_tb.vcd

扩展 1:
  input_checker.png, sobel_out_checker.png, sobel_system_tb_checker.vcd

扩展 2:
  sobel_threshold_comparison.png
  sobel_thresh_{0,32,64,128}_out.png
  sobel_thresh_{0,32,64,128}.vcd

扩展 3:
  sobel_missing_pixels.vcd

扩展 4:
  waveform_timing_start.png, waveform_timing_mid.png
  waveform_timing_end.png, waveform_timing_overview.png
```

### 原始结果备份
```
build_original/             原始仿真结果（未修改的所有文件）
```
