# E1设备WiFi和蓝牙研究报告

## 目录
[TOC]

## 硬件配置概览

### 芯片信息
- **WiFi/蓝牙芯片**: Realtek RTL8723DS
- **连接方式**: SDIO (MMC接口)
- **平台**: Rockchip RK3308
- **MAC地址**: c4:f1:d1:4b:6e:8a

### 设备树配置
```dts
// WiFi配置
wireless-wlan {
    compatible = "wlan-platdata";
    wifi_chip_type = "rtl8723ds";
    WIFI,host_wake_irq = <&gpio4 RK_PD0 GPIO_ACTIVE_HIGH>;
    status = "okay";
};

// 蓝牙配置
wireless-bluetooth {
    compatible = "bluetooth-platdata";
    uart_rts_gpios = <&gpio4 RK_PC7 GPIO_ACTIVE_LOW>;
    BT,power_gpio = <&gpio4 RK_PC11 GPIO_ACTIVE_HIGH>;
    BT,wake_host_irq = <&gpio4 RK_PC12 GPIO_ACTIVE_HIGH>;
    status = "okay";
};
```

### GPIO控制线路
| 功能 | GPIO | 方向 | 当前状态 | 描述 |
|------|------|------|----------|------|
| WiFi Host Wake | gpio4_PD0 | 输入 | - | WiFi唤醒主机 |
| BT RTS | gpio-135 | 输入 | 高电平 | 蓝牙UART流控 |
| BT Power | gpio-139 | 输出 | 低电平 | 蓝牙电源控制 |
| BT Wake Host | gpio-140 | 输入 | 高电平 | 蓝牙唤醒主机 |

## WiFi配置和使用

### 硬件能力
```
芯片: RTL8723DS
频段: 2.4GHz (13频道)
最大速率: 150 Mbps (HT40)
支持模式: STA, AP, IBSS, P2P
加密支持: WEP40/104, TKIP, CCMP-128
天线: 1T1R配置
发射功率: 20.0 dBm
```

### 接口状态
```bash
# 查看WiFi接口
ip link show wlan0
iwconfig wlan0

# 当前连接状态
iwgetid
cat /proc/net/wireless
```

### 扫描和连接
```bash
# 扫描可用网络
iwlist wlan0 scan | grep -E "(ESSID|Quality|Signal)"

# 查看详细的PHY信息
iw phy0 info
iw dev wlan0 info

# 信号强度监控
watch -n 1 'iwconfig wlan0 | grep -E "(Quality|Signal|Rate)"'
```

### P2P功能
E1设备支持WiFi P2P功能，通过p2p0接口：
```bash
# 查看P2P接口
iwconfig p2p0
iw dev p2p0 info
```

## 蓝牙配置和使用

### 当前状态分析
根据检测结果，蓝牙硬件存在但处于关闭状态：
- **rfkill状态**: Soft blocked: yes
- **电源GPIO**: gpio-139 = 低电平 (关闭)
- **固件**: 位于 `/lib/firmware/rtlbt/`

### 激活蓝牙
```bash
# 1. 解除软件阻塞
rfkill unblock bluetooth

# 2. 通过GPIO激活蓝牙电源
echo 1 > /sys/class/gpio/gpio139/value

# 3. 验证状态
rfkill list bluetooth
cat /sys/class/gpio/gpio139/value

# 4. 检查是否出现蓝牙设备
hciconfig -a
ls -la /sys/class/bluetooth/
```

### 蓝牙服务管理
```bash
# 启动蓝牙服务 (如果系统支持)
systemctl start bluetooth
systemctl enable bluetooth

# 或者手动加载模块
modprobe bluetooth
modprobe hci_uart
modprobe btrtl
```

### 蓝牙操作
```bash
# 基础操作
hciconfig hci0 up
hciconfig hci0 scan

# 使用bluetoothctl
bluetoothctl
> power on
> agent on
> default-agent
> scan on
> devices
> pair [MAC_ADDRESS]
```

## Rust编程接口

### 依赖配置 (Cargo.toml)
```toml
[dependencies]
# 网络和WiFi
tokio = { version = "1.0", features = ["full"] }
tokio-stream = "0.1"
nix = "0.27"

# 蓝牙
btleplug = "0.11"
bluez-async = "0.7"

# 系统交互
sysinfo = "0.29"
libc = "0.2"

# UI
slint = "1.2"
```

### WiFi管理器
```rust
use std::process::{Command, Stdio};
use std::io::{BufRead, BufReader};
use tokio::process::Command as AsyncCommand;

#[derive(Debug, Clone)]
pub struct WiFiNetwork {
    pub essid: String,
    pub signal_strength: i32,
    pub quality: u8,
    pub encryption: String,
    pub frequency: f64,
}

#[derive(Debug, Clone)]
pub struct WiFiStatus {
    pub connected: bool,
    pub essid: Option<String>,
    pub signal_strength: i32,
    pub link_quality: u8,
    pub bit_rate: f64,
    pub tx_power: i32,
}

pub struct WiFiManager {
    interface: String,
}

impl WiFiManager {
    pub fn new(interface: &str) -> Self {
        Self {
            interface: interface.to_string(),
        }
    }

    pub async fn get_status(&self) -> Result<WiFiStatus, Box<dyn std::error::Error>> {
        let output = AsyncCommand::new("iwconfig")
            .arg(&self.interface)
            .output()
            .await?;

        let output_str = String::from_utf8(output.stdout)?;
        
        // 解析iwconfig输出
        let connected = output_str.contains("Access Point:") && !output_str.contains("Not-Associated");
        let essid = self.extract_essid(&output_str);
        let signal_strength = self.extract_signal_strength(&output_str);
        let link_quality = self.extract_link_quality(&output_str);
        let bit_rate = self.extract_bit_rate(&output_str);
        
        Ok(WiFiStatus {
            connected,
            essid,
            signal_strength,
            link_quality,
            bit_rate,
            tx_power: 20, // RTL8723DS固定发射功率
        })
    }

    pub async fn scan_networks(&self) -> Result<Vec<WiFiNetwork>, Box<dyn std::error::Error>> {
        let output = AsyncCommand::new("iwlist")
            .arg(&self.interface)
            .arg("scan")
            .output()
            .await?;

        let output_str = String::from_utf8(output.stdout)?;
        Ok(self.parse_scan_results(&output_str))
    }

    pub async fn connect_network(&self, essid: &str, password: Option<&str>) -> Result<(), Box<dyn std::error::Error>> {
        // 使用wpa_supplicant连接网络
        if let Some(pass) = password {
            let mut child = AsyncCommand::new("wpa_passphrase")
                .arg(essid)
                .arg(pass)
                .stdout(Stdio::piped())
                .spawn()?;

            let output = child.wait_with_output().await?;
            let config = String::from_utf8(output.stdout)?;
            
            // 写入临时配置文件
            tokio::fs::write("/tmp/wpa_temp.conf", config).await?;
            
            // 连接网络
            AsyncCommand::new("wpa_supplicant")
                .args(&["-i", &self.interface, "-c", "/tmp/wpa_temp.conf", "-B"])
                .output()
                .await?;
        }

        Ok(())
    }

    fn extract_essid(&self, output: &str) -> Option<String> {
        if let Some(start) = output.find("ESSID:\"") {
            let start = start + 7;
            if let Some(end) = output[start..].find("\"") {
                return Some(output[start..start + end].to_string());
            }
        }
        None
    }

    fn extract_signal_strength(&self, output: &str) -> i32 {
        if let Some(start) = output.find("Signal level=") {
            let start = start + 13;
            if let Some(end) = output[start..].find(" dBm") {
                return output[start..start + end].parse().unwrap_or(0);
            }
        }
        0
    }

    fn extract_link_quality(&self, output: &str) -> u8 {
        if let Some(start) = output.find("Link Quality=") {
            let start = start + 13;
            if let Some(end) = output[start..].find("/") {
                return output[start..start + end].parse().unwrap_or(0);
            }
        }
        0
    }

    fn extract_bit_rate(&self, output: &str) -> f64 {
        if let Some(start) = output.find("Bit Rate:") {
            let start = start + 9;
            if let Some(end) = output[start..].find(" Mb/s") {
                return output[start..start + end].parse().unwrap_or(0.0);
            }
        }
        0.0
    }

    fn parse_scan_results(&self, output: &str) -> Vec<WiFiNetwork> {
        let mut networks = Vec::new();
        let mut current_network = None;
        
        for line in output.lines() {
            let line = line.trim();
            
            if line.contains("Cell") && line.contains("Address:") {
                if let Some(network) = current_network.take() {
                    networks.push(network);
                }
                current_network = Some(WiFiNetwork {
                    essid: String::new(),
                    signal_strength: 0,
                    quality: 0,
                    encryption: String::new(),
                    frequency: 0.0,
                });
            }
            
            if let Some(ref mut network) = current_network {
                if line.contains("ESSID:") {
                    if let Some(start) = line.find("\"") {
                        if let Some(end) = line[start + 1..].find("\"") {
                            network.essid = line[start + 1..start + 1 + end].to_string();
                        }
                    }
                } else if line.contains("Quality=") {
                    // 解析信号质量和强度
                    if let Some(quality_str) = line.split("Quality=").nth(1) {
                        if let Some(quality_part) = quality_str.split_whitespace().next() {
                            if let Some(numerator) = quality_part.split('/').next() {
                                network.quality = numerator.parse().unwrap_or(0);
                            }
                        }
                    }
                    if let Some(signal_str) = line.split("Signal level=").nth(1) {
                        if let Some(signal_part) = signal_str.split_whitespace().next() {
                            network.signal_strength = signal_part.parse().unwrap_or(0);
                        }
                    }
                }
            }
        }
        
        if let Some(network) = current_network {
            networks.push(network);
        }
        
        networks
    }
}

// GPIO控制
pub struct WiFiGPIO {
    power_gpio: u32,
    wake_gpio: u32,
}

impl WiFiGPIO {
    pub fn new() -> Self {
        Self {
            power_gpio: 139,  // BT power GPIO (也控制WiFi电源域)
            wake_gpio: 140,   // BT/WiFi wake host
        }
    }

    pub async fn set_power(&self, enabled: bool) -> Result<(), Box<dyn std::error::Error>> {
        let value = if enabled { "1" } else { "0" };
        tokio::fs::write(format!("/sys/class/gpio/gpio{}/value", self.power_gpio), value).await?;
        
        // 等待电源稳定
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        Ok(())
    }

    pub async fn get_wake_status(&self) -> Result<bool, Box<dyn std::error::Error>> {
        let content = tokio::fs::read_to_string(format!("/sys/class/gpio/gpio{}/value", self.wake_gpio)).await?;
        Ok(content.trim() == "1")
    }
}
```

### 蓝牙管理器
```rust
use std::process::Command;
use tokio::process::Command as AsyncCommand;

#[derive(Debug, Clone)]
pub struct BluetoothDevice {
    pub address: String,
    pub name: String,
    pub rssi: i16,
    pub connected: bool,
    pub paired: bool,
}

pub struct BluetoothManager {
    hci_device: String,
}

impl BluetoothManager {
    pub fn new() -> Self {
        Self {
            hci_device: "hci0".to_string(),
        }
    }

    pub async fn initialize(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 1. 激活蓝牙电源
        self.power_on().await?;
        
        // 2. 解除rfkill阻塞
        AsyncCommand::new("rfkill")
            .args(&["unblock", "bluetooth"])
            .output()
            .await?;

        // 3. 启动HCI设备
        tokio::time::sleep(tokio::time::Duration::from_millis(500)).await;
        AsyncCommand::new("hciconfig")
            .args(&[&self.hci_device, "up"])
            .output()
            .await?;

        Ok(())
    }

    pub async fn power_on(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 通过GPIO激活蓝牙
        tokio::fs::write("/sys/class/gpio/gpio139/value", "1").await?;
        tokio::time::sleep(tokio::time::Duration::from_millis(200)).await;
        Ok(())
    }

    pub async fn power_off(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 关闭HCI设备
        AsyncCommand::new("hciconfig")
            .args(&[&self.hci_device, "down"])
            .output()
            .await?;

        // 关闭GPIO电源
        tokio::fs::write("/sys/class/gpio/gpio139/value", "0").await?;
        Ok(())
    }

    pub async fn is_powered(&self) -> Result<bool, Box<dyn std::error::Error>> {
        let output = AsyncCommand::new("hciconfig")
            .arg(&self.hci_device)
            .output()
            .await?;

        let output_str = String::from_utf8(output.stdout)?;
        Ok(output_str.contains("UP RUNNING"))
    }

    pub async fn scan_devices(&self, duration_secs: u64) -> Result<Vec<BluetoothDevice>, Box<dyn std::error::Error>> {
        // 开始扫描
        AsyncCommand::new("hciconfig")
            .args(&[&self.hci_device, "leadv", "3"])
            .output()
            .await?;

        AsyncCommand::new("hciconfig")
            .args(&[&self.hci_device, "lescan"])
            .output()
            .await?;

        // 等待扫描
        tokio::time::sleep(tokio::time::Duration::from_secs(duration_secs)).await;

        // 停止扫描
        AsyncCommand::new("hciconfig")
            .args(&[&self.hci_device, "noleadv"])
            .output()
            .await?;

        // 解析扫描结果
        self.get_discovered_devices().await
    }

    async fn get_discovered_devices(&self) -> Result<Vec<BluetoothDevice>, Box<dyn std::error::Error>> {
        let output = AsyncCommand::new("hcitool")
            .args(&["lescan"])
            .output()
            .await?;

        let output_str = String::from_utf8(output.stdout)?;
        let mut devices = Vec::new();

        for line in output_str.lines() {
            if let Some((address, name)) = self.parse_device_line(line) {
                devices.push(BluetoothDevice {
                    address,
                    name,
                    rssi: 0, // 需要额外查询
                    connected: false,
                    paired: false,
                });
            }
        }

        Ok(devices)
    }

    fn parse_device_line(&self, line: &str) -> Option<(String, String)> {
        let parts: Vec<&str> = line.split_whitespace().collect();
        if parts.len() >= 2 {
            let address = parts[0].to_string();
            let name = parts[1..].join(" ");
            Some((address, name))
        } else {
            None
        }
    }

    pub async fn pair_device(&self, address: &str) -> Result<(), Box<dyn std::error::Error>> {
        AsyncCommand::new("bluetoothctl")
            .args(&["pair", address])
            .output()
            .await?;
        Ok(())
    }

    pub async fn connect_device(&self, address: &str) -> Result<(), Box<dyn std::error::Error>> {
        AsyncCommand::new("bluetoothctl")
            .args(&["connect", address])
            .output()
            .await?;
        Ok(())
    }
}
```

## 网络管理

### 系统集成
```bash
# 创建网络管理服务
cat > /etc/systemd/system/e1-wireless.service << 'EOF'
[Unit]
Description=E1 Wireless Manager
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/e1-wireless-manager
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

# 启用服务
systemctl enable e1-wireless.service
systemctl start e1-wireless.service
```

### 网络配置脚本
```bash
#!/bin/bash
# E1 WiFi快速配置脚本

INTERFACE="wlan0"
CONFIG_FILE="/etc/wpa_supplicant/wpa_supplicant.conf"

connect_wifi() {
    local ssid="$1"
    local password="$2"
    
    echo "连接到 $ssid..."
    
    # 生成配置
    wpa_passphrase "$ssid" "$password" > /tmp/wifi_config.conf
    
    # 停止现有连接
    killall wpa_supplicant 2>/dev/null
    
    # 启动新连接
    wpa_supplicant -i $INTERFACE -c /tmp/wifi_config.conf -B
    
    # 获取IP
    sleep 3
    dhcpcd $INTERFACE
    
    echo "WiFi连接完成"
}

# 使用示例
# connect_wifi "MyNetwork" "MyPassword"
```

## 故障排除

### 常见问题

#### 1. WiFi连接问题
```bash
# 检查WiFi状态
iwconfig wlan0
ip addr show wlan0

# 重启WiFi接口
ip link set wlan0 down
ip link set wlan0 up

# 检查驱动
dmesg | grep -i rtl8723
lsmod | grep 8723
```

#### 2. 蓝牙激活失败
```bash
# 检查GPIO状态
cat /sys/class/gpio/gpio139/value

# 手动激活蓝牙
echo 1 > /sys/class/gpio/gpio139/value
rfkill unblock bluetooth
hciconfig hci0 up

# 检查固件
ls -la /lib/firmware/rtlbt/
dmesg | grep -i bluetooth
```

#### 3. 信号质量差
```bash
# 检查天线连接
iwlist wlan0 scan | grep -A5 -B5 "Quality"

# 调整发射功率
iwconfig wlan0 txpower 20

# 监控信号质量
watch -n 1 'cat /proc/net/wireless'
```

### 性能优化

#### 1. WiFi性能调优
```bash
# 禁用电源管理
iwconfig wlan0 power off

# 设置最大发射功率
iwconfig wlan0 txpower 20

# 优化TCP参数
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf
```

#### 2. 蓝牙性能调优
```bash
# 设置HCI参数
hciconfig hci0 sspmode 1
hciconfig hci0 class 0x200404

# 优化扫描参数
hciconfig hci0 inqmode 2
hciconfig hci0 inqtype 0
```

### 调试工具

```bash
# 创建无线调试脚本
cat > /usr/local/bin/wireless-debug.sh << 'EOF'
#!/bin/bash
echo "=== E1无线调试信息 ==="
echo "时间: $(date)"
echo ""

echo "--- WiFi状态 ---"
iwconfig wlan0
echo ""

echo "--- 蓝牙状态 ---"
hciconfig -a
rfkill list
echo ""

echo "--- GPIO状态 ---"
echo "BT Power (GPIO139): $(cat /sys/class/gpio/gpio139/value 2>/dev/null || echo 'N/A')"
echo "BT Wake (GPIO140): $(cat /sys/class/gpio/gpio140/value 2>/dev/null || echo 'N/A')"
echo ""

echo "--- 网络统计 ---"
cat /proc/net/wireless
echo ""

echo "--- 内核消息 ---"
dmesg | grep -E "(wifi|bt|rtl)" | tail -10
EOF

chmod +x /usr/local/bin/wireless-debug.sh
```

