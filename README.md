# BLE DPV 曲线绘图器

可靠性增强版新增 `GET_SYSTEM_STATUS (0x12)`，网页可显示初始化阶段、最后错误、
队列/丢包计数、DMA错误及四个任务的最小剩余栈。原有DPV数据帧和扫描范围命令
保持兼容。

## 功能

- 连接 BLE 并实时绘制 DPV 差分电流。
- 读取和设置 `RampStartVolt`、`RampPeakVolt`。
- 启动、停止测量并等待 STM32 应答。
- 命令应答超时为 800 ms，最多自动重试 2 次。

## DPV 数据帧

STM32 保持原有固定 9 字节、小端序数据帧：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 1 | 数据帧头 `0xA5` |
| 1 | 2 | `point`，`uint16` |
| 3 | 4 | `current`，`int32`，单位 nA |
| 7 | 1 | Byte 1～6 累加和的低 8 位 |
| 8 | 1 | 帧尾 `0x5A` |

## 命令与响应帧

不定长协议帧格式：

```text
A6 + FrameType + Sequence + Command + PayloadLength + Payload
   + CRC16_L + CRC16_H + 5A
```

- 请求类型：`0x01`
- 响应类型：`0x02`
- CRC：CRC16-CCITT，多项式 `0x1021`，初始值 `0xFFFF`
- CRC范围：`FrameType`至Payload最后一个字节

| 命令 | 命令字 | 请求Payload |
|---|---:|---|
| 开始测量 | `0x01` | 无 |
| 停止测量 | `0x02` | 无 |
| 设置扫描范围 | `0x10` | 起始电位和终止电位，各为`int16`小端mV |
| 获取扫描范围 | `0x11` | 无 |

响应Payload的第一个字节是状态码，后续字节是命令返回数据。扫描范围限制为：

```text
-800 <= RampStartVolt < RampPeakVolt <= 1200 mV
```

当前阶梯步进为10 mV，因此`RampPeakVolt - RampStartVolt`必须是10 mV的整数倍。

扫描运行中设置参数会返回`BUSY`。参数只保存在RAM中，重新上电后恢复固件默认值。

## 硬件连接

- `STM32 PA9  -> BLE RXD`
- `STM32 PA10 <- BLE TXD`
- `STM32 GND  <-> BLE GND`

## 使用方法

1. 将`index.html`部署到HTTPS静态网站。
2. 使用支持Web Bluetooth的安卓Chrome打开。
3. 连接BLE后，网页自动读取STM32中的扫描范围。
4. 修改起始和终止电位，点击“应用扫描范围”。
5. 等待设置成功应答，然后点击“开始检测”。

默认服务UUID为`0000ffe0-0000-1000-8000-00805f9b34fb`，默认通知特征为
`0000ffe1-0000-1000-8000-00805f9b34fb`。控制特征留空时与通知特征共用。
