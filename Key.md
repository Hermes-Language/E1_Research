# E1设备按键研究报告

## 目录
[TOC]

## 硬件配置

### 设备树配置
E1设备使用ADC按键，通过电压分压识别不同按键：

```dts
adc-keys {
    compatible = "adc-keys";
    io-channel-names = "buttons";
    poll-interval = <0x3c>;  // 60ms轮询间隔
    keyup-threshold-microvolt = <0x1b7740>;
    io-channels = <0x7d 0x01>;  // ADC通道1

    vol-up-key {
        label = "volume up";
        linux,code = <0x05>;  // KEY_4
        press-threshold-microvolt = <0x4650>;  // 18000µV
    };

    vol-down-key {
        label = "volume down";
        linux,code = <0x06>;  // KEY_5
        press-threshold-microvolt = <0x493e0>;  // 300000µV
    };

    esc-key {
        label = "micmute";
        linux,code = <0x02>;  // KEY_1
        press-threshold-microvolt = <0x112a88>;  // 1125000µV
    };

    menu-key {
        label = "play";
        linux,code = <0x03>;  // KEY_2
        press-threshold-microvolt = <0x95a88>;  // 612000µV
    };

    home-key {
        label = "mode";
        linux,code = <0x04>;  // KEY_3
        press-threshold-microvolt = <0xd90a8>;  // 890000µV
    };
}
```

### 硬件特性
- **ADC设备**: `/sys/devices/platform/ff1e0000.saradc`
- **输入设备**: `/dev/input/event3`
- **轮询间隔**: 60ms (可调节)
- **响应时间**: 60-120ms
- **ADC通道**: voltage1 (`in_voltage1_raw`)

## 按键映射

| 物理按键 | Linux按键码 | 事件码 | 功能 | ADC阈值(µV) |
|----------|-------------|--------|------|-------------|
| Volume Up | KEY_4 | 5 | 音量+ | 18,000 |
| Volume Down | KEY_5 | 6 | 音量- | 300,000 |
| Mic Mute | KEY_1 | 2 | 麦克风静音 | 1,125,000 |
| Play/Pause | KEY_2 | 3 | 播放/暂停 | 612,000 |
| Mode | KEY_3 | 4 | 模式切换 | 890,000 |

## Linux输入子系统

### 查看按键设备
```bash
# 查看所有输入设备
cat /proc/bus/input/devices

# 监控按键事件
evtest /dev/input/event3

# 查看ADC原始值
cat /sys/devices/platform/ff1e0000.saradc/iio:device*/in_voltage1_raw

# 查看轮询间隔
cat /sys/devices/platform/adc-keys/input/input3/poll
```

### 输入事件结构
```c
struct input_event {
    struct timeval time;  // 时间戳
    __u16 type;          // 事件类型 (EV_KEY = 1)
    __u16 code;          // 按键码 (2-6)
    __s32 value;         // 值 (1=按下, 0=释放)
};
```

## Rust程序读取按键

### 依赖配置 (Cargo.toml)
```toml
[dependencies]
evdev = "0.12"
tokio = { version = "1.0", features = ["full"] }
slint = "1.2"
```

### 基础按键读取
```rust
use evdev::{Device, EventType, Key};
use std::collections::HashMap;

#[derive(Debug, Clone, Copy)]
pub enum E1Key {
    VolumeUp,
    VolumeDown,
    MicMute,
    Play,
    Mode,
}

impl E1Key {
    pub fn from_code(code: u16) -> Option<Self> {
        match code {
            5 => Some(E1Key::VolumeUp),
            6 => Some(E1Key::VolumeDown),
            2 => Some(E1Key::MicMute),
            3 => Some(E1Key::Play),
            4 => Some(E1Key::Mode),
            _ => None,
        }
    }

    pub fn name(&self) -> &'static str {
        match self {
            E1Key::VolumeUp => "Volume Up",
            E1Key::VolumeDown => "Volume Down",
            E1Key::MicMute => "Mic Mute",
            E1Key::Play => "Play/Pause",
            E1Key::Mode => "Mode",
        }
    }
}

pub struct KeyReader {
    device: Device,
}

impl KeyReader {
    pub fn new() -> Result<Self, Box<dyn std::error::Error>> {
        // 打开ADC按键设备
        let device = Device::open("/dev/input/event3")?;
        Ok(KeyReader { device })
    }

    pub async fn read_events<F>(&mut self, mut callback: F) -> Result<(), Box<dyn std::error::Error>>
    where
        F: FnMut(E1Key, bool, f64), // key, pressed, timestamp
    {
        loop {
            let events = self.device.fetch_events()?;
            for event in events {
                if event.event_type() == EventType::KEY {
                    if let Some(key) = E1Key::from_code(event.code()) {
                        let pressed = event.value() == 1;
                        let timestamp = event.timestamp().as_secs_f64();
                        callback(key, pressed, timestamp);
                    }
                }
            }
            tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
        }
    }
}
```

### 高级按键处理
```rust
use std::time::{Duration, Instant};
use tokio::sync::mpsc;

#[derive(Debug, Clone)]
pub enum KeyEvent {
    Press(E1Key),
    Release(E1Key),
    LongPress(E1Key),
    DoubleClick(E1Key),
}

pub struct KeyProcessor {
    sender: mpsc::UnboundedSender<KeyEvent>,
    last_press: HashMap<E1Key, Instant>,
    press_start: HashMap<E1Key, Instant>,
}

impl KeyProcessor {
    pub fn new() -> (Self, mpsc::UnboundedReceiver<KeyEvent>) {
        let (sender, receiver) = mpsc::unbounded_channel();
        let processor = KeyProcessor {
            sender,
            last_press: HashMap::new(),
            press_start: HashMap::new(),
        };
        (processor, receiver)
    }

    pub fn process_key(&mut self, key: E1Key, pressed: bool, _timestamp: f64) {
        let now = Instant::now();

        if pressed {
            // 检查双击
            if let Some(last_time) = self.last_press.get(&key) {
                if now.duration_since(*last_time) < Duration::from_millis(500) {
                    let _ = self.sender.send(KeyEvent::DoubleClick(key));
                    self.last_press.remove(&key);
                    return;
                }
            }

            self.press_start.insert(key, now);
            let _ = self.sender.send(KeyEvent::Press(key));
            self.last_press.insert(key, now);

        } else {
            // 按键释放
            if let Some(start_time) = self.press_start.remove(&key) {
                let duration = now.duration_since(start_time);
                
                if duration > Duration::from_millis(1000) {
                    let _ = self.sender.send(KeyEvent::LongPress(key));
                } else {
                    let _ = self.sender.send(KeyEvent::Release(key));
                }
            }
        }
    }
}
```

## 性能优化

### 调整轮询间隔
```bash
# 查看当前间隔
cat /sys/devices/platform/adc-keys/input/input3/poll

# 设置更短的间隔（需要root权限）
echo 30 | sudo tee /sys/devices/platform/adc-keys/input/input3/poll

# 最小建议值: 10-20ms
echo 20 | sudo tee /sys/devices/platform/adc-keys/input/input3/poll
```

### Rust性能优化
```rust
// 使用更高效的事件读取
use evdev::Device;
use std::os::unix::io::AsRawFd;

impl KeyReader {
    pub async fn read_events_async<F>(&mut self, mut callback: F) -> Result<(), Box<dyn std::error::Error>>
    where
        F: FnMut(E1Key, bool, f64),
    {
        let fd = self.device.as_raw_fd();
        let mut poll = mio::Poll::new()?;
        let mut events = mio::Events::with_capacity(128);
        
        poll.registry().register(
            &mut mio::unix::SourceFd(&fd),
            mio::Token(0),
            mio::Interest::READABLE,
        )?;

        loop {
            poll.poll(&mut events, None)?;
            
            for event in events.iter() {
                if event.token() == mio::Token(0) {
                    let input_events = self.device.fetch_events()?;
                    for input_event in input_events {
                        if input_event.event_type() == EventType::KEY {
                            if let Some(key) = E1Key::from_code(input_event.code()) {
                                let pressed = input_event.value() == 1;
                                let timestamp = input_event.timestamp().as_secs_f64();
                                callback(key, pressed, timestamp);
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## 故障排除

### 常见问题

1. **权限问题**
```bash
# 检查设备权限
ls -la /dev/input/event3

# 临时解决方案
sudo chmod 666 /dev/input/event3

# 永久解决方案：添加udev规则
echo 'KERNEL=="event*", SUBSYSTEM=="input", MODE="0666"' | sudo tee /etc/udev/rules.d/99-input.rules
sudo udevadm control --reload-rules
```

2. **设备不存在**
```bash
# 检查ADC按键设备
cat /proc/bus/input/devices | grep -A5 "adc-keys"

# 检查内核模块
lsmod | grep adc
dmesg | grep -i adc
```

3. **响应延迟**
```bash
# 检查轮询间隔
cat /sys/devices/platform/adc-keys/input/input3/poll

# 检查系统负载
top
cat /proc/loadavg
```

4. **调试工具**
```bash
# 实时监控按键
evtest /dev/input/event3

# 监控ADC值
watch -n 0.1 'cat /sys/devices/platform/ff1e0000.saradc/iio:device*/in_voltage1_raw'

# 查看输入子系统状态
cat /proc/bus/input/handlers
```

### 性能基准
- **标准轮询间隔**: 60ms
- **推荐轮询间隔**: 20-30ms
- **最小响应时间**: ~1个轮询周期
- **典型响应时间**: 20-60ms
- **长按检测**: >1000ms
- **双击间隔**: <500ms
