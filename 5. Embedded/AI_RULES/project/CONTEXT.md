# 业务背景与工程全景 (CONTEXT)

规范编号：PROJECT-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 产品概述与设计目标

本项目是基于 **C / FreeRTOS** 与 **ESP-IDF v5.x** 构建的高可靠、低功耗物联网智能硬件嵌入式固件系统。

固件的核心应用场景涵盖智能照明设备（户外太阳能智能庭院灯、RGB/CCT氛围灯具系列）以及物联网智能硬件设备（如 YeahBox 系列音箱与硬件终端），与移动端 App（Flutter）、Web 运营管理后台（React）以及云端 IoT Core（FastAPI / MQTT Broker）构成完整端云一体化闭环。

核心设计目标包括：
1. **极致的硬件可靠性与防变砖能力**：硬件一旦交付出厂，OTA 是唯一的固件迭代维保通道。必须构建 A/B 双分区无缝容灾、ECDSA-SHA256 验签与防降级回滚闭环。
2. **高频调光无频闪与丝滑视觉体验**：采用硬件定时器 LEDC 输出 $\ge 20\text{kHz}$ 高频 PWM，消除人耳啸叫与摄影机条纹频闪，搭配 Gamma 2.2 非线性感知校正。
3. **低功耗运行与超长待机保护**：支持自动动态调频 (DFS)、Light Sleep 与 Deep Sleep，待机功耗控制在 0.5W 以下，集成锂电池放电曲线滤波与欠压阶梯降额保护。

---

## 2. 硬件芯片与核心组件全景

所有固件组件统一组织在 `components/` 目录下，实现高度内聚与模块自治：

```
firmware/
├── main/                           # 系统总装入口与顶层状态机 (FSM)
│   ├── main.c                      # app_main 入口：外设上电时序、组件总装启动
│   └── app_state_machine.c         # 系统生命周期、配网、连云、升级状态机
└── components/
    ├── bsp/                        # 板级支持包：芯片引脚定义 (Pinout)、时钟源与电源域配置
    ├── drivers/                    # 硬件抽象驱动：LEDC 高频 PWM、ADC 电量采样、GPIO/PIR 中断、按键
    ├── lighting_engine/            # 照明核心算法：Gamma 2.2 查表校正、CCT 双色温插值、HSV 色彩空间转换
    ├── protocols/                  # 通信协议栈：
    │   ├── ble/                    # 对齐《BLE API 规范》：31B 主广播构造 (Flags + 0xFF55 UUID + 0xFF 厂商数据)、GATT Server
    │   ├── mqtt/                   # 对齐《MQTT 规范》：TLS 加密传输、小驼峰 JSON 物模型、60s 心跳、遗嘱 LWT
    │   └── wifi_prov/              # Wi-Fi 配网管理器：支持 BLE Protocomm 配网与 SoftAP 配网
    ├── ota_manager/                # OTA 固件升级：YBOT 头部信封解析、ECDSA 离线验签、A/B 双分区刷写与自检
    ├── storage/                    # 持久化存储：NVS 参数存取、掉电保护与 2 秒状态变更防抖定时器
    └── power_mgmt/                 # 电源管理：太阳能电池过充过放保护、低电量降额、休眠唤醒调度
```

---

## 3. 跨系统契约对齐与依赖关系

1. **移动端 (Flutter App)**：
   - 固件 BLE 主广播严格采用 31 字节格式（3B Flags + 4B `0xFF55` Service UUID + 24B `0xFF` 厂商数据，含 `AdvVer`、`BID`、`PID`、`ModelID`、`CAP`、`STATE`、`StateSeq` 与 `DeviceID_Short`）。
   - App 发送的控制指令通过 Control Service (`0xFF55`) 下的自定义 128-bit 特征交互（Write: `A83B0002-7D4E-4F9A-9C12-8E4B01000001` / Notify: `A83B0003-7D4E-4F9A-9C12-8E4B01000001`），所有应用层多字节字段均遵循**大端序 (Big-Endian)**。
2. **云端中枢 (FastAPI & MQTT Broker)**：
   - 设备建连时向 `devices/{product_id}/{device_sn}/status` 注册遗嘱消息 (LWT)。
   - 属性上报与下发 Payload 字段严格遵循小驼峰命名（如 `deviceId`、`batteryLevel`、`firmwareVersion`）。
3. **OTA 分发系统**：
   - 固件升级包严格遵循《OTA 固件升级与设备管理规范》定义的 `YBOT` 二进制信封格式，包含 Magic `0x59424F54`、硬件兼容掩码及 ECDSA-SHA256 签名。
