# JD7601 + ESP32-S3 BLE 网页传图（初版）

## 本版本做什么

- 保留原 `EPD3.7_JD7601` 驱动与 GPIO 默认配置。
- 增加 NimBLE GATT 服务，设备名 `JD7601-EPAPER`。
- Android Chrome 网页导入 JPG/PNG，保留原图并在固定 2:3 框内裁切，可每次旋转 90°。
- 网页预览可切换调整后原图与六色发送帧，并提供六色色带测试图。
- 图片转换为 4bpp 六色帧（172800 字节），按 JD7601 的 480 条扫描线、每线 720 像素布局编码；支持 Floyd–Steinberg 误差扩散和 Bayer 有序抖动，默认使用误差扩散改善照片渐变。
- BLE 使用较大无响应写入分包并按浏览器/连接能力回退，帧收齐后调用原驱动刷新。
- 原始 LUT 文件放在 `experimental_lut_not_enabled/`，不参与默认编译，避免未经确认的波形直接驱动面板。

## 重要限制

- “48 色”目前是**六色实体墨水 + 像素抖动形成的视觉模拟**，不是保证 48 种独立实体颜色。
- BLE 传输每包最多 240 字节图像数据（加 4 字节 offset 后 ATT 值为 244 字节）；网页优先使用无响应写入并在 MTU 不足时自动尝试较小分包。低 MTU 或浏览器限制仍会降低速度；屏幕刷新另需约 30–40 秒。
- 当前项目按 ESP-IDF 5.3.1/NimBLE API 配置，和你提供的串口监视日志版本一致。若编译时出现 API/组件差异，请把从第一条 `error:` 开始的完整日志发回。
- 不能保证未经实机验证就可直接工作；原屏幕驱动的 GPIO 默认值、BUSY 极性和电源/高压外围必须与实际硬件一致。

## 目录

- `components/EPD3.7_JD7601/` 原驱动（保留）
- `main/app_main.c` BLE 接收和刷新任务
- `web/index.html` 手机网页
- `experimental_lut_not_enabled/` 未启用 LUT 原文件和安全说明

## 第一次编译与烧录

1. 安装 Espressif ESP-IDF 5.3.1（包含 Python、CMake、Ninja 和工具链）。
2. 解压本 ZIP 到一个普通英文路径，例如 `C:\esp\JD7601_BLE_Image`。
3. 打开 ESP-IDF 5.5 PowerShell / ESP-IDF Terminal，进入项目目录。
4. 执行：
   ```powershell
   idf.py set-target esp32s3
   idf.py reconfigure
   idf.py build
   ```
5. 连接开发板，确认 `main/app_main.c` 的 GPIO 与实际接线相同，再执行：
   ```powershell
   idf.py -p COMx flash monitor
   ```
   将 `COMx` 换成设备管理器中实际的串口号。
6. 串口日志应出现 `JD7601 ready; BLE image reception enabled`。


## 依据你提供的原理图更新的墨水屏引脚

| 原理图网络名 | ESP32-S3 引脚 | 固件配置字段 |
|---|---:|---|
| E_BUSY | IO1 / GPIO1 | `busy_gpio` |
| E_RES | IO2 / GPIO2 | `reset_gpio` |
| E_DC | IO3 / GPIO3 | `dc_gpio` |
| E_CS | IO4 / GPIO4 | `cs_gpio` |
| E_SCL | IO5 / GPIO5 | `sclk_gpio` |
| E_SDA | IO6 / GPIO6 | `sda_gpio`（SPI MOSI/数据输出） |

注意：图中的 `CHIP_PU` 是 ESP32-S3 的芯片使能/复位网络，`BOOT` 对应 IO0；它们不是墨水屏 SPI 信号，本程序不将它们配置成墨水屏 GPIO。图中的 NFC_IRQ=IO8、AXP_IRQ=IO9 也暂未使用。请以完整原理图和实物 PCB 网络为准，尤其确认 `E_SDA` 在此板上接的是 SPI 数据输入端（ESP32 输出）。

## 手机网页使用

1. 网页使用 Web Bluetooth，需要 Android Chrome，并且页面必须由 HTTPS 安全来源提供（例如你自己的 GitHub Pages）。直接从手机文件管理器打开 `index.html` 可能无法使用蓝牙。
2. 发布 `web/index.html` 到 HTTPS 静态网站；无需服务器后端。
3. 打开网页，点击“连接墨水屏”，选择 `JD7601-EPAPER`。
4. 选择图片，在固定比例框内调整位置/缩放/旋转，生成六色预览后发送。
5. 如需检查面板显示，可发送“六色测试图”，屏幕应出现黑、白、黄、红、蓝、绿色带。
6. 发送完成后等待墨水屏刷新，不要在刷新期间断电。
7. 若 Chrome 无法连接，确认网页是 HTTPS、系统蓝牙已打开，并检查串口日志。

## BLE 协议

- 服务 UUID：`01000001-7620-4ea1-9147-6f3a114c5d8a`
- 控制特征 UUID：`02000001-7620-4ea1-9147-6f3a114c5d8a`，写入 `01` 开始一帧。
- 数据特征 UUID：`03000001-7620-4ea1-9147-6f3a114c5d8a`，每包为 4 字节 little-endian offset + 图像数据。
- 状态特征 UUID：`04000001-7620-4ea1-9147-6f3a114c5d8a`，只读状态字符串。

## 故障时

- 若 `idf.py build` 失败，请把从第一条 `error:` 到最后的构建日志发给我。
- 若编译通过但屏幕不刷新，发串口 `monitor` 日志，并核对 GPIO、BUSY、VDD 与高压外围。
- 若显示方向/颜色错误，先不要改 LUT；先确认原驱动色带显示正常。
