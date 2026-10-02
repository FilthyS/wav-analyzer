# Stage 1：WAV 与数字音频基础

这一阶段的目标是先建立 WAV、PCM 与 binary representation 的基本心智模型。暂时不要写 parser，也不要为了答题查找每一道诊断题的答案；不确定的地方直接写出来即可。

## 第一版项目边界

`wavinfo` 第一版应该：

- 接收一个 WAV 文件路径。
- 解析 RIFF/WAVE 的 chunk 结构，不假设固定的 44-byte header。
- 仅支持 little-endian、uncompressed PCM、16-bit、mono/stereo。
- 输出 format、channels、sample rate、bit depth、duration。
- 分别计算每个 channel 的 peak 与 RMS，并转换为 dBFS。
- 对 unsupported、malformed 或 truncated 文件给出清楚错误。

第一版暂时不做：

- MP3、FLAC、compressed WAV、floating-point PCM、24/32-bit PCM
- playback、recording、real-time audio
- FFT、frequency spectrum、LUFS、true peak
- GUI、JUCE、audio framework
- 为未来假想需求设计 inheritance、factory 或 generic abstraction layer

## Part A：起点诊断

请凭当前理解回答，不必查资料。答错或不确定都是有效信息。

### 1. Little-endian

文件中依次出现两个 byte：

```text
44 AC
```

如果它们组成一个 little-endian unsigned 16-bit integer：

- 你会怎样计算其数值？
- little-endian 描述的究竟是什么顺序？

**我的回答：**

<!-- 在这里回答 -->

不清楚little-endian的概念
十六进制的数值计算的话

$$
0x44 = 4 \times 16 + 6 \times 1
0xAC = 10 \times 16 + 12 \times 1
$$

**导师批注 — Developing：**

- 你知道 hexadecimal 使用 positional notation，这个方向正确。
- 有一个计算笔误：`0x44` 的第二个 digit 是 `4`，所以应为 $4 \times 16 + 4$，不是 $4 \times 16 + 6$。
- 目前算出的 `0x44` 和 `0xAC` 只是两个 byte 各自的值，还没有把它们组合成一个 16-bit integer。
- `endianness` 不改变单个 byte 内 bit 的书写顺序；它规定一个 multi-byte value 的各个 byte 按什么顺序存放。阅读后请回来算出完整的 `0x44 0xAC`。

### 2. Signed 16-bit integer

对 16-bit signed integer，你认为以下 bit patterns 分别表示什么十进制数？如果不确定，请直接说明。

```text
0x0000
0x7FFF
0x8000
0xFFFF
```

**我的回答：**

<!-- 在这里回答 -->

**导师批注 — Not yet covered：**

这一题留空说明 signed integer 的 bit representation 需要补齐。重点不是背四个答案，而是理解：同一个 bit pattern 的数值含义取决于把它解释为 unsigned 还是 signed two's complement，以及为什么 16-bit signed range 两端不对称。阅读后再回答本题。

### 3. Stereo PCM 的数据量

一个 stereo、44,100 Hz、16-bit PCM 文件：

- “44,100 samples per second”是整个文件每秒共 44,100 个 sample，还是每个 channel 各 44,100 个？
- 一秒的原始 audio data 大约占多少 bytes？请写出推导，并注意单位。

**我的回答：**

<!-- 在这里回答 -->

每个channel 44100
stereo： $2 \times 44100 \times 16 / 8$ byte / second

**导师批注 — Correct，补一个关键连接：**

- `44,100 Hz` 是每个 channel 每秒 44,100 个 sample，你的判断正确。
- 数据量公式正确；请再写出最终数值和单位。
- 在 stereo 中，同一个采样时刻的一组 `L`、`R` samples 叫一个 **sample frame**。因此这里每秒有 44,100 frames、88,200 samples；每个 frame 占 4 bytes。后面推导 `block align` 和 duration 时会反复使用这个关系。

### 4. 基本术语

用你目前的理解分别解释以下术语，每项一两句话即可：

- sample
- sample rate
- bit depth
- amplitude
- PCM

**我的回答：**

<!-- 在这里回答 -->

连续的模拟信号的某个时刻的值
取上述值的频率
采样的精度/使用多少memory来储存sample
响度/波形中的能量
不清楚

**导师批注：**

- `sample` — **Good intuition, terminology needs adjustment**：它不是连续信号本身在某时刻的值，而是对该值进行 measurement、再经 quantization 后得到的一个离散数值。可以暂时记作“某个离散时刻的离散 amplitude value”。
- `sample rate` — **基本正确**：更精确地说，是每个 channel 每秒取得多少个 samples，单位为 samples/s，也写作 Hz。
- `bit depth` — **Needs qualification**：它是每个 sample 使用的 bits 数，并决定可用 quantization levels 的数量。storage size 是直接后果；“精度”是可接受的直觉，但要具体理解成 amplitude quantization resolution。
- `amplitude` — **Incorrect connection**：amplitude 不是 loudness，也不是 energy。这里先把它理解为信号在纵轴上的瞬时数值或大小。loudness 是听觉感知，energy/power 通常与 amplitude 的平方有关，三者不能互换。
- `PCM` — **Not yet covered**：阅读后再定义，重点关注它如何把随时间变化的信号表示成一串 numbers。

## Part B：最小阅读任务

请按下面的顺序阅读。前三项用于补 Part A 的缺口，后两项再进入 RIFF/WAV。范围已经刻意缩小，不需要顺着链接读完整章。

### 补充阅读 1：Endianness

阅读 University of Maryland 的 [A Simple Model of Memory](https://www.cs.umd.edu/class/sum2003/cmsc311/Notes/CompOrg/simpMem.html) 中：

- `Multiple-byte quantities`
- `Endianness`

阅读目的：理解 endianness 排列的是一个 multi-byte value 中的 **bytes**，以及 least significant byte 是什么。

读完后应能回答：数值 `0xAC44` 写入文件后，little-endian 的两个 bytes 为什么依次是 `44 AC`？

### 补充阅读 2：Two's complement

阅读 Cornell 的短讲义 [Two’s complement notation](https://www.cs.cornell.edu/courses/JavaAndDS/files/twosComplement.pdf)。

阅读目的：理解 signed 16-bit PCM sample 的整数表示，而不只是记忆正负号规则。

读完后应能回答：

- n-bit two's complement 的 range 是什么？
- 为什么 `0xFFFF` 不是 signed integer 的最大正数？
- 为什么 16-bit PCM 会出现 `-32768` 与 `+32767` 的不对称？

### 补充阅读 3：Sampling、quantization 与 PCM 基础

阅读 Digital Sound & Music 的 [5.1.2 Digitization](https://digitalsoundandmusic.com/5-1-2-digitization/) 中：

- `5.1.2.1 Two Steps: Sampling and Quantization`
- `5.1.2.3 Bit Depth and Quantization Error`，读到 16-bit 有 65,536 个 discrete levels 的说明即可

暂时跳过 `Sampling and Aliasing`、`Dynamic Range`、dBFS 与 dithering；这些会在对应阶段单独学习。

阅读目的：把 sample、sample rate、bit depth、amplitude、quantization 和 PCM 串成一个模型。

读完后应能回答：

- digital audio 为什么同时是 discrete in time 和 discrete in amplitude？
- sampling 与 quantization 分别离散化了什么？
- bit depth 增加时，增加的是采样频率还是 amplitude levels？

### 阅读 1：RIFF overview

阅读 Microsoft 的 [Resource Interchange File Format (RIFF)](https://learn.microsoft.com/en-us/windows/win32/xaudio2/resource-interchange-file-format--riff-)。

阅读目的：

- 理解 RIFF 是一种 container structure，而不是 audio encoding。
- 理解一个普通 chunk 由 `chunk ID + chunk size + chunk data` 构成。
- 初步区分 `RIFF`、`WAVE`、`fmt ` 和 `data`。

### 阅读 2：WAVE specification summary

阅读 McGill 的 [WAVE File Specifications](https://www-mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/WAVE.html) 中以下部分：

- `Wave File Format`
- `fmt Chunk`
- `PCM Format`
- `data Chunk`
- 关于不能假设 header 固定为 44 bytes 的 note

阅读目的：

- 看清 `fmt ` 中每个核心 field 的含义和宽度。
- 理解 `data` 只有结合 `fmt ` 才能被正确解释。
- 理解 WAV 中可能存在额外 chunks，parser 必须逐个查找。

可选参考：Microsoft 的 [原始 RIFF/WAVE 文档副本](https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/Docs/riffmci.pdf)。目前不要求通读。

## Part C：阅读后检查

### 1. PCM 与 WAV

PCM 和 WAV 为什么不是同一个概念？

**我的回答：**

<!-- 在这里回答 -->

### 2. `fmt ` 与 `data` 的关系

`fmt ` 中的信息为什么是解释 `data` bytes 所必需的？

**我的回答：**

<!-- 在这里回答 -->

### 3. 固定 offset 的问题

为什么寻找 `data` chunk 时不能简单地跳到 byte offset 44？

**我的回答：**

<!-- 在这里回答 -->

### 4. Chunk size

普通 chunk 的 size field 是否包含 chunk ID 和 size field 本身？

**我的回答：**

<!-- 在这里回答 -->

### 5. `fmt ` 的空格

`fmt ` 为什么有一个容易忽略的尾随空格？

**我的回答：**

<!-- 在这里回答 -->

## 学习状态（待回答后更新）

### Solid

- CS 与 C++ 基础（目前依据自述，之后通过代码确认）
- 能正确推导 stereo PCM 每秒的 byte 数
- 理解 sample rate 是 per-channel rate

### Developing

- hexadecimal positional notation（有一次 digit 计算笔误）
- sample、sample rate 与 bit depth 的基本直觉
- byte count、sample count 与 frame count 的关系

### Not yet covered

- endianness
- signed two's complement representation
- amplitude 与 loudness/energy 的区别
- PCM 与 quantization
- WAV/RIFF 结构
- duration、peak、RMS 与 dBFS
- DSP 基础概念
