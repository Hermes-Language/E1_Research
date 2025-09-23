# SogouE1降噪机制分析

基于IDA Pro反编译libnnse.so和模型文件分析，SogouE1的降噪主要通过NNSE (Neural Network Speech Enhancement)实现。

## 降噪技术栈

### 核心算法
- **NNSE (Neural Network Speech Enhancement)**：使用深度学习模型进行语音增强和降噪。
- **LSTM-based NNSE**：字符串中出现"lstm_nnse"，表明使用长短时记忆网络（LSTM）进行时序语音处理。
- **CRN (Causal Recurrent Network)**：另一种架构，用于实时音频降噪。
- **集成方式**：通过libnnse.so库实现，结合libnnse_nnet.so进行神经网络计算。

### 配置支持
- **nnse.conf.model.bin**：NNSE模型二进制配置文件，包含神经网络参数。
- **nnse.conf.ms.bin**：多尺度NNSE配置，可能用于不同频率带处理。
- **alg_nnse_enable**：在config.ini中启用NNSE算法（推测配置项）。

### 工作流程
1. **音频捕获**：从麦克风阵列获取原始音频数据。
2. **预处理**：应用AGC/ALC进行初步增益调节。
3. **特征提取**：转换为频域特征（如Mel频谱）。
4. **NNSE处理**：LSTM/CRN网络分析音频特征，分离语音和噪声，增强目标信号。
5. **后处理**：结合MVDR波束成形，进一步抑制方向性干扰。
6. **输出**：提供降噪后的清晰音频。

### 性能特点
- **深度学习优势**：比传统滤波器更智能，能适应复杂噪声环境。
- **实时性**：集成于音频处理管道，支持低延迟处理。
- **多尺度处理**：nnse.conf.ms.bin暗示多分辨率分析，提高降噪精度。

### 局限性
- **计算复杂度**：神经网络需要较多CPU/GPU资源。
- **模型依赖**：依赖预训练模型，适应性有限。

## 关键函数分析

### 初始化函数
- `_ZN5sogou4nnet4nnse15sogou_nnet_initEPKc`：初始化NNSE模型，加载配置文件。
- `_ZN5sogou4nnet4nnse23sogou_nnet_forward_initEPvi`：初始化前向传播。

### 处理函数
- `_ZN2MA5Cnnse12nnse_processEiPKsiPsPi`：主NNSE处理函数，输入音频数据，输出增强音频。
- `_ZN5sogou4nnet4nnse30sogou_nnet_forward_feedforwardEPvPfii`：神经网络前向传播。

### 配置函数
- `_ZN2MA14load_nnse_confEPcS0_`：加载NNSE配置文件。
- `_ZN2MA14show_nnse_confEPv`：显示配置信息。

## 详细实现流程

### 主处理流程 (nnse_process)
1. **参数验证**：检查输入输出参数有效性。
2. **状态重置**：如果是第一帧或第三帧，重置内部状态。
3. **数据准备**：调用data_prepare处理输入音频。
4. **频谱变换**：调用trans_spect进行STFT变换。
5. **NNSE增益计算**：如果启用NNSE，调用nnse_gain计算增益。
6. **AFE IMCR处理**：如果启用AFE，调用afe_imcra进行传统降噪。
7. **频谱恢复**：调用recover_spect恢复时域信号。
8. **输出处理**：调用output生成最终音频。

### 频谱变换 (trans_spect)
1. **窗口处理**：使用汉宁窗等分析窗对音频分帧。
2. **STFT变换**：进行短时傅里叶变换，得到复数频谱。
3. **功率谱计算**：计算每个频率点的功率谱密度。
4. **数据存储**：保存STFT结果和功率谱用于后续处理。

### NNSE增益计算 (nnse_gain)
1. **模型类型判断**：根据配置确定使用LSTM还是CRN模型。
2. **特征准备**：准备神经网络输入特征。
3. **前向推理**：调用sogou_nnet_forward_feedforward进行神经网络推理。
4. **增益应用**：将网络输出转换为频域增益。
5. **输出处理**：根据模型类型处理推理结果。

## 模型文件结构

### CRN NNSE模型 (crn_nnse/)
- **nnse.conf**：CRN（Causal Recurrent Network）NNSE配置文件，指定模型类型为CRN (NNSE_TYPE=1)
- **nnse.conf.crn.nnet.scale.bin**：CRN神经网络模型二进制文件，自定义格式，包含网络权重和架构参数
- **nnse.conf.ms.bin**：多尺度NNSE配置二进制文件，包含采样率(16000)、FFT大小(1024)等参数

### LSTM NNSE模型 (lstm_nnse/)
- **nnse.conf**：LSTM（Long Short-Term Memory）NNSE配置文件，指定模型类型为LSTM (NNSE_TYPE=0)
- **nnse.conf.model.bin**：LSTM神经网络模型二进制文件，自定义格式，包含网络权重和架构参数
- **nnse.conf.ms.bin**：多尺度NNSE配置二进制文件，与CRN版本相同

### 语音质量检测模型 (voice_quality_detection/)
- **glitch_filter.conf**：故障滤波配置文件，用于检测和过滤音频故障
- **mean_var_1100h_afe.npy**：NumPy数组文件，包含1100Hz前置滤波的均值和方差统计，用于特征归一化
- **model.gru.20200501.2**：GRU（Gated Recurrent Unit）模型文件，自定义二进制格式，版本20200501.2，用于语音质量评估

## 模型文件格式分析

### 二进制格式特点
- **格式类型**：Sogou自定义二进制格式，非标准ONNX格式
- **数据结构**：头部配置参数 + 浮点权重数据
- **字节序**：小端序（Little-Endian）
- **数据类型**：混合使用int32和float32

### 头部结构分析
通过十六进制分析发现的头部结构：
- **魔数/版本**：CRN文件以77开头，LSTM文件以7开头
- **网络参数**：包含FFT大小(1024)、帧长(512)等配置
- **权重数据**：从offset 80开始的float32权重数组

### 文件熵值分析
- CRN模型文件熵值：4.71（介于纯文本和随机数据之间）
- 表明数据有结构但不是高度压缩的格式

### 使用方式
1. **配置文件读取**：nnse.conf指定NNSE_TYPE和模型文件名
2. **库函数调用**：
   - `load_nnse_conf()` - 加载模型配置
   - `init_nnse()` - 初始化神经网络
   - `do_nnse()` - 执行降噪推理
3. **实时处理**：支持16kHz音频，帧长512样本

### 技术特点
- **专用格式**：专为嵌入式设备优化，无法直接转换为ONNX
- **量化优化**：可能包含设备特定的权重量化
- **内存布局**：针对RK3308 DSP优化的数据排列

## 配置参数

从字符串分析，NNSE支持大量参数：
- NNSE_TYPE：模型类型（LSTM/CRN）
- NNSE_EPS：最小值阈值
- NNSE_MIN_SNR：最小信噪比
- NNSE_ALFA_SP_1ST/2ND：谱平滑因子
- NNSE_ALFA_SNR_1ST/2ND：SNR平滑因子
- NNSE_ETA_MIN_DB：最小增益（dB）
- NNSE_LEN_SM_WIN：平滑窗口长度
- 等等

## 技术实现细节

### 音频处理流程
1. **时域到频域**：使用STFT（短时傅里叶变换）转换到频域。
2. **特征提取**：计算Mel频谱特征。
3. **神经网络推理**：通过LSTM/CRN模型预测增强掩码或直接特征。
4. **频域增强**：应用掩码增强目标信号。
5. **逆变换**：ISTFT回到时域。

### 集成架构
- **libnnse.so**：主NNSE库，处理配置和接口。
- **libnnse_nnet.so**：神经网络推理库，实际计算。
- **依赖库**：使用FFFT库进行FFT运算。

### 优化技术
- **SIMD优化**：使用NEON指令集进行向量运算加速。
- **内存复用**：高效的内存管理，减少分配开销。
- **流水线处理**：并行处理多个音频帧。

## 性能评估

- **降噪效果**：能有效降低背景噪声，提高语音可懂度。
- **延迟**：支持实时处理，帧长512，帧移256。
- **资源占用**：LSTM模型较大（471KB ONNX），CRN较小（26KB）。

该降噪系统结合传统信号处理和深度学习，提供强大的语音增强能力。