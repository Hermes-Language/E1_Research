# Original\sogou目录文件作用分析

## 系统文件
- **boot.bmp**：启动画面位图文件，用于设备开机显示。
- **core_helper**：核心助手可执行程序，可能处理系统初始化或后台服务。
- **system_version.info**：JSON格式系统信息文件，包含OS名称（ZhiyinOS）、构建时间（2020-01-14）、版本（1.0.2）等。
- **welcome.mp3**：欢迎音频文件，开机播放。

## 用户界面文件
- **privacy.html**：隐私政策网页，HTML格式，包含用户协议条款。
- **userprotocal.html**：用户协议网页，类似隐私政策。

## FAQ资源 (faq/)
- **index.html**：FAQ主页，HTML格式。
- **detail.html**：FAQ详情页，HTML格式。
- **css/808a0b3c.detail.min.css**：详情页压缩CSS样式文件。
- **css/808a0b3c.index.min.css**：主页压缩CSS样式文件。
- **js/detail.86a5ac.js**：详情页JavaScript文件（压缩），处理交互。
- **js/detail.js**：详情页JavaScript文件（未压缩）。
- **js/index.86a5ac.js**：主页JavaScript文件（压缩）。
- **js/index.js**：主页JavaScript文件（未压缩）。
- **js/vendor.86a5ac.js**：第三方库JavaScript文件（压缩）。
- **js/vendor.js**：第三方库JavaScript文件（未压缩）。
- **img/new/btn_go.png**：FAQ按钮图标（普通分辨率）。
- **img/new/btn_go@2x.png**：FAQ按钮图标（高分辨率，2x）。
- **img/new/icon_arrow_right.png**：右箭头图标。

## 前端配置 (front-config/)
### 全局ASR配置 (asr/)
- **agc.conf**：自动增益控制（AGC）配置文件。
- **config-level2.ini**：二级ASR配置INI文件。
- **config.ini**：主ASR配置INI文件。
- **mclp.conf**：麦克风阵列线性预测（MCLP）配置文件。
- **mvdr.conf**：最小方差无失真响应（MVDR）波束成形配置文件。

### 模型1配置 (model-1/)
#### ASR子配置 (asr/)
- **agc.conf**：模型1 ASR AGC配置。
- **config-level2.ini**：模型1二级ASR配置。
- **config.ini**：模型1主ASR配置。
- **mclp.conf**：模型1 MCLP配置。
- **mvdr.conf**：模型1 MVDR配置。

#### SOH子配置 (soh/)
- **afe_fd.conf**：前置滤波（AFE Feedforward）配置文件。
- **config.ini**：模型1 SOH主配置。
- **derever.conf**：混响消除（Dereverberation）配置文件。
- **mclp.conf**：模型1 SOH MCLP配置。
- **mvdr.conf**：模型1 SOH MVDR配置。

### 模型2配置 (model-2/)
#### ASR子配置 (asr/)
- **agc.conf**：模型2 ASR AGC配置。
- **config-level2.ini**：模型2二级ASR配置。
- **config.ini**：模型2主ASR配置。
- **mclp.conf**：模型2 MCLP配置。
- **mvdr.conf**：模型2 MVDR配置。

#### SOH子配置 (soh/)
- **afe_fd.conf**：模型2前置滤波配置。
- **config.ini**：模型2 SOH主配置。
- **derever.conf**：模型2混响消除配置。
- **mclp.conf**：模型2 SOH MCLP配置。
- **mvdr.conf**：模型2 SOH MVDR配置。

### NNSE降噪配置 (nnse/)
- **nnse.conf.model.bin**：NNSE模型二进制配置文件，包含神经网络参数。
- **nnse.conf.ms.bin**：多尺度NNSE二进制配置，可能用于不同频率带处理。

# Original\model目录文件作用分析

## 模型文件
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

## 模型文件格式说明

### 文件格式
- **格式类型**：自定义二进制格式，非标准ONNX格式
- **数据类型**：混合格式（整数配置参数 + 浮点权重数据）
- **字节序**：小端序（Little-Endian）
- **压缩**：未压缩，直接存储权重数据

### 头部结构（前80字节）
- **Offset 0-3**：魔数/版本标识符
- **Offset 4-7**：FFT大小或帧长参数
- **Offset 8-11**：隐藏层大小或其他网络参数
- **Offset 12-15**：输出大小或其他配置参数
- **Offset 16+**：浮点权重数据开始

### 使用方式
1. **配置加载**：通过nnse.conf文件指定模型类型和文件名
2. **库接口**：使用libnnse.so提供的C接口函数：
   - `load_nnse_conf()` - 加载模型配置文件
   - `init_nnse()` - 初始化NNSE处理器
   - `do_nnse()` - 执行降噪处理
3. **实时处理**：支持16kHz采样率，帧长512，帧移256的实时音频处理

### 注意事项
- 这些模型文件专为SogouE1设备优化，无法直接在其他平台使用
- 需要配套的libnnse.so和libnnse_nnet.so库文件
- 模型文件包含设备特定的优化和量化参数