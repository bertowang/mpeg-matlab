# mpeg-matlab —— 经典 MPEG 混合编码教学仿真

> 一句话定位：用最直白的 MATLAB 代码，把「I / P / B 帧 + GOP + 8×8 DCT 量化 + 块匹配运动估计与补偿」这条**经典 MPEG-1/2 式混合编码链路**完整跑一遍，并把重建结果写成 AVI 视频。

面向初学者的章节式图解教程见同目录的 **`tutorial.html`**（双击即可用浏览器打开，离线可用，内嵌 SVG 流程图）。

---

## 1. 这个工程是什么，不是什么

**它是**：约 2004 年前后流传的教学代码集（运动估计部分署名 Aroh Barjatya），用于课堂演示「为什么相邻帧可以只传差值」「为什么 DCT 后大部分系数可以丢掉」「B 帧如何借助前后参考帧」。

**它不是**：

- 不是标准兼容的 MPEG 编码器/解码器，输出**不是** `.mpg` / 码流文件；
- **没有真正的熵编码**：`decoder.m` 里硬编码了 MPEG 标准的 AC/DC Huffman 表，但这些表只被当作参数逐层传递、从未被解析；`ZigZag.m`、`dezigzag.m`、`chkcategory.m`、`makeLayers.m` 全库无调用点，`makeLayers.m` 只是恒等函数，`deRunLength.m` 仅转调 `inverseDCT`；
- **没有码率控制、没有率失真优化（RDO）**：质量由固定的 `Qtable2.txt` 量化表决定，B 帧统一用 `0.5*array`；
- 没有帧内预测、整数变换、亚像素/多参考帧、去块滤波等 H.264 及以后特性。

因此本工程的「编码结果」只是**内存中的量化 DCT 系数矩阵** `JQ`；解码端把它反量化、反变换，再加回运动补偿图像。它演示的是**原理链路**，不是压缩文件格式。

## 2. 为什么保持独立工程

工作区中另有 `../videoCodecDemo`，是 2004 年 Marc Servais 博士研究中的**率失真优化研究型 codec**（可变块划分、亚像素运动估计、OBMC、Matching Pursuits 稀疏残差、Range Coding 真实码流、λ 码控）。

两者**不是同一个系统的两个模块，而是同一学科的两个平行实现**：本工程是「经典标准链路的简化骨架」，另一个是「现代 RD 优化的完整研究实现」。它们的码流、残差表示、数据结构与运动估计接口完全异构，无法上下游拼接；且都存在对当前工作目录的强假设与旧版 MATLAB API 依赖。

**结论：刻意不合并**，各自独立运行、独立维护，通过各自教程中的「知识地图」互相参照。这样也避免 `videoCodecDemo` 中 `@uint32` / `@uint8` 类方法重载影响本工程的整数图像运算。

## 3. 文件清单

33 个 `.m` 全部平铺在根目录，无子目录、无包、无构建脚本。

| 知识领域 | 文件 | 作用 |
|---|---|---|
| 演示入口 | `main44.m` | **唯一顶层脚本**：清屏、读视频、循环 6 个 GOP、逐帧编解码、显示并写出 `2.avi` |
| 序列 I/O | `GOP.m` | 按 `i+1 … i+9` 从 AVI 读出 9 帧，缩放到 128×128 并转 `double` |
| 帧级编码 | `compensatedFrame.m` / `compensatedFrameB.m` | P 帧 / B 帧的运动补偿 + 残差 + DCT |
| 帧级编码 | `bFrameProc.m` / `bFrameCal.m` / `pFrameCal.m` | B 帧参考选择与薄封装（`pFrameCal.m` 未被调用且参数个数不匹配） |
| 变换量化 | `forwardDCT.m` / `inverseDCT.m` | `Im-128` → 8×8 DCT → 除以量化表取 `round`；逆过程 +128 |
| 熵编码（骨架） | `deRunLength.m` / `makeLayers.m` / `ZigZag.m` / `dezigzag.m` / `chkcategory.m` | 未接入主流程的占位/备用实现 |
| 运动估计 | `motionEstTSS.m`（主流程唯一使用）、`motionEstES.m`、`motionEstDS.m`、`motionEstARPS.m` | P 帧四族快速块匹配：三步搜索 / 全搜索 / 菱形搜索 / 自适应十字搜索 |
| 运动估计 | `BmotionEstTSS.m`（主流程唯一使用）、`BmotionEstES.m`、`BmotionEstDS.m`、`BmotionEstARPS.m`、`PmotionEstES.m` | B 帧对应实现，额外返回零运动矢量个数 |
| 运动补偿 | `motionComp.m`（主流程唯一使用）、`MC.m` | 按 MV 从参考帧搬运 8×8 块 |
| 代价与工具 | `costFuncMAD.m` / `minCost.m` / `zeroMatrixCount.m` | MAD 匹配准则、最小代价、零 MV 统计 |
| 解码 | `decoder.m` / `Idecoder.m` / `pdecoder.m` / `bdecoder.m` | 按 `identifier`（1…10）和 `flag`（0/1）分派帧类型与参考帧 |
| 质量评价 | `imgPSNR.m` | PSNR 计算（未接入主流程，可用于实验） |
| 数据 | `news.avi`（唯一输入）、`Qtable2.txt`（8×8 标准亮度量化表）、`2.avi`（运行时生成的输出） | — |
| 说明 | `read me.txt`（原始 16 行说明）、`main44.asv`（MATLAB 自动备份）、本文件、`tutorial.html` | — |

## 4. 运行方法

### 环境要求

- MATLAB + **Image Processing Toolbox**（用到 `dctmtx`、`blkproc`、`imresize`、`frame2im`、`imsubtract`、`imadd`、`imshow`）；
- **旧版 MATLAB（约 R2010b 之前）**：代码使用 `avifile` / `addframe` / `aviread` / `aviinfo` / `blkproc`，这些 API 在现代版本已被移除或更名；
- 无 MEX、无 `.exe`、无外部库、无 `addpath` 调用。

```matlab
cd('…/mpeg-matlab')   % 必须把当前目录设为本工程根目录
main44                % 运行（脚本开头有 clear all，会清空工作区）
```

运行后会弹出图像窗口逐帧显示重建帧，并在当前目录生成 `2.avi`（25 fps，无压缩）。

### 现代 MATLAB 迁移提示

| 旧 API | 位置 | 替换 |
|---|---|---|
| `aviread` / `aviinfo` / `frame2im` | `main44.m`、`GOP.m` | `VideoReader` + `readFrame` |
| `avifile` / `addframe` / `close(aviobj)` | `main44.m:7,142,225` | `VideoWriter` + `writeVideo` |
| `blkproc` | `forwardDCT.m`、`inverseDCT.m` | `blockproc` |
| `bitcmp(val,category)` 旧语法 | `chkcategory.m` | 按新版语法改写 |

## 5. 关键参数在哪改

全部集中在 `main44.m`：

| 参数 | 位置 | 当前值 | 含义 |
|---|---|---|---|
| 帧尺寸 | `main44.m:32`、`GOP.m` 共 9 处 | `[128 128]` | 输入帧统一缩放尺寸，必须是 8 的倍数 |
| `mbSize` | `main44.m:25` | `8` | 宏块边长（像素） |
| `p` | `main44.m:27` | `7` | 运动估计搜索半径，TSS 步长由此推出 |
| GOP 循环 | `main44.m:44` | `for i=1:10:60` | 固定处理 6 个 GOP（最大读到第 55 帧） |
| 量化表 | `main44.m:10-12`、`decoder.m:169-170` | `Qtable2.txt` | 8×8 亮度量化表；改大 → 更模糊、系数更小 |
| B 帧量化 | `main44.m:67,75,91,99,115,123` | `0.5*array` | B 帧用更细的量化 |
| 运动估计算法 | `compensatedFrame.m:3`、`bFrameProc.m:4,7` | `motionEstTSS` / `BmotionEstTSS` | 可替换为 ES / DS / ARPS 做对比实验 |

## 6. 处理链路速览

```
news.avi ──aviread→ 128×128 double 帧
   │
   ├ I 帧 : Im-128 → 8×8 DCT → /量化表 → JQ（内存中，即"编码结果"）
   │              └→ ×量化表 → IDCT → +128 → buffer（本地重建帧，作为后续参考）
   │
   ├ P 帧 : 对参考 buffer 做 TSS 块匹配 → MV → motionComp → 预测图
   │        残差 = 当前帧 − 预测图 → DCT/量化 → JQ
   │        解码：IDCT(JQ) + 预测图 → 新 buffer
   │
   ├ B 帧 : 分别对前、后参考帧做 TSS，比较"零 MV 个数"
   │        → 选零 MV 更多的那个作为参考（flag=0 前 / flag=1 后）
   │        → 残差额外 −128（解码端 +128 补回）
   │
   └ 解码总控 decoder.m 用 identifier=1…10 + flag 分派 → global *d → imshow/getframe → 2.avi
```

GOP 显示顺序：`I B B P B B P B B I`（10 帧）。注意本实现的 B 帧是**前向/后向择一单向参考**，不是标准的双向平均插值。

关键教学点：编码器内部包含一份「解码器」（`inverseDCT` + 运动补偿重建 `buffer*`），因为后续帧必须参照**解码端看到的、已失真的重建帧**做预测，否则误差会累积漂移。

## 7. 已知问题与代码债（保持原样，供学习辨析）

1. **文件名与内部函数名大小写不一致**：`pdecoder.m`→`pDecoder`、`bdecoder.m`→`bDecoder`、`dezigzag.m`→`deZigZag`、`ZigZag.m`→`zigzag`。Windows 侥幸可跑，Linux 会断。
2. **重复函数定义**：`PmotionEstES.m:15` 声明 `motionEstES`（与 `motionEstES.m` 重名）；`motionEstARPS.m:20` 声明 `BmotionEstARPS`（与 `BmotionEstARPS.m` 重名）；`zeroMatrixCount.m:1` 声明 `BmotionEstES` 且**文件无函数体**。
3. `pFrameCal.m` 向 `compensatedFrame.m` 传 6 个参数而函数只收 5 个，属死代码。
4. `clear all`（`main44.m:2`）与 10 个 `global *d`（`main44.m:29`、`decoder.m:167`）使解码结果不经返回值而经全局工作区传递，不便复用。
5. `Qtable2.txt` 被 `main44.m` 与 `decoder.m` 各 `fopen` 一次，两处都没有 `fclose`。
6. P 残差不减 128、B 残差再 −128 并在解码端 +128（`compensatedFrameB.m:6-7`、`bdecoder.m:6`），是不对称的数值 hack，不是标准做法。
7. `BmotionEstES.m` 中存在恒不成立的判断分支；`motionVect=zeros(2,256)` 写死 128×128 / 8×8 = 256 个块。
8. `main44.asv` 是编辑器自动备份，不要加入 MATLAB 路径。

## 8. 建议的学习路线

`tutorial.html` 按以下章节组织，建议顺序阅读：

1. 先修概念：时间冗余、空间冗余、I/P/B 帧、GOP
2. 全局鸟瞰：一张图看懂 `main44` 流水线
3. 视频读取与 GOP 时间轴
4. I 帧与 8×8 DCT／量化／ZigZag
5. 块匹配运动估计与三步搜索（TSS）
6. 运动补偿与残差
7. B 帧的参考选择
8. 解码器分派与「编码器内嵌解码器」的误差漂移问题
9. 熵编码的空座位：哪里可以自己补全
10. 知识地图、自测题与动手运行

## 9. 来源与授权

- 运动估计系列文件注释含 `Written by Aroh Barjatya`，算法引用 Shan Zhu & Ma (2000, DS)、Yao Nie & Ma (2002, ARPS)，属于网络上广泛流传的教学代码。
- 工程**未附带任何 LICENSE / 授权声明**，也无版本与作者总说明；原始说明仅 `read me.txt`。
- 请仅用于教学与自学，勿假定可自由再分发；如需公开使用，请先确认原始代码授权。
- 本 `README.md` 与 `tutorial.html` 为新增的说明性文档，未改动任何原有代码与数据。
