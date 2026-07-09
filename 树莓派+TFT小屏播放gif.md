# 树莓派+TFT小屏播放gif

本来是想买个 ESP32 开发板连一个 TFT 小屏当个桌面 gif 播放器装饰的，结果买回来发现那个淘宝小店的开发板有问题，一直烧录不进去，无奈退货。转念一想，我可以直接用树莓派接小屏啊，于是花一会就弄出来了，还蛮简单的。

## 1. 引脚连接（右侧为派）

```
TFT GND  -> Pin 6   GND
TFT VCC  -> Pin 1   3.3V
TFT SCL  -> Pin 23  GPIO11 / SCLK
TFT SDA  -> Pin 19  GPIO10 / MOSI
TFT RES  -> Pin 13  GPIO27
TFT DC   -> Pin 22  GPIO25
TFT CS   -> Pin 24  GPIO8 / CE0
TFT BLK  -> Pin 17  3.3V
```

## 2. 屏幕配置

### 开启 SPI

完成硬件接线后，首先需要开启树莓派的 SPI 功能。

```
Ubuntu 24.04 下修改：
/boot/firmware/config.txt
添加：
dtparam=spi=on
重启后检查：
ls /dev/spidev*
```

### Python 驱动环境

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv python3-pil python3-spidev python3-libgpiod python3-lgpio

python3 -m venv --system-site-packages ~/tftenv
source ~/tftenv/bin/activate

pip install --upgrade adafruit-blinka adafruit-circuitpython-rgb-display pillow
```

### 用户权限问题

```bash
sudo usermod -aG dialout yyx
sudo reboot
```

### GPIO busy 问题

GPIO 权限解决后，又遇到：`lgpio.error: 'GPIO busy'`，原因是 ST7789 的 CS 引脚连接到了：`GPIO8 / CE0`
而 Raspberry Pi 5 默认 SPI 配置会占用 CE0，导致 Python 无法控制该 GPIO。

```
修改：
/boot/firmware/config.txt
关闭默认 CS 管理：
#dtparam=spi=on
dtoverlay=spi0-0cs
```
重启后，释放 CS 控制权，让 Python 负责控制 GPIO8。

### ST7789 显示偏移问题

成功运行测试程序后，屏幕可以显示颜色块，但是底部内容被裁剪。原因是 ST7789 控制器内部实际分辨率为：240 × 320，而实际屏幕大小为：240 × 240。

代码中需要设置显示偏移：

```py
y_offset=80
```

## 3. 加入脚本

在`/home/yyx/Pictures`放入`nuomu.gif`，在`/home/yyx/spi_test`新建 `play_gif.py`，写入：

```python
import time
from PIL import Image, ImageSequence, ImageOps

import board
import digitalio
from adafruit_rgb_display import st7789


WIDTH = 240
HEIGHT = 240

GIF_PATH = "/home/yyx/pet.gif"


# SPI
spi = board.SPI()


# 引脚
cs_pin = digitalio.DigitalInOut(board.D8)
dc_pin = digitalio.DigitalInOut(board.D25)
reset_pin = digitalio.DigitalInOut(board.D27)


# ST7789
disp = st7789.ST7789(
    spi,
    cs=cs_pin,
    dc=dc_pin,
    rst=reset_pin,
    baudrate=24000000,
    width=WIDTH,
    height=HEIGHT,
    x_offset=0,
    y_offset=80,
    rotation=180, # 上下翻转
)


# 打开 GIF
gif = Image.open(GIF_PATH)


frames = []
delays = []


# 预加载所有帧
for frame in ImageSequence.Iterator(gif):

    img = frame.convert("RGB")

    # 缩放到240x240
    img = ImageOps.fit(
        img,
        (WIDTH, HEIGHT),
        method=Image.Resampling.LANCZOS
    )

    frames.append(img)

    # 获取GIF自己的播放速度
    delays.append(
        frame.info.get("duration", 100) / 1000
    )


print(f"GIF frames: {len(frames)}")


# 无限循环播放
while True:

    for img, delay in zip(frames, delays):

        disp.image(img)

        time.sleep(delay  * 0.5) # 0.5可以让速度更快一些
```

## 4. 设置开机后台运行

```bash
sudo nano /etc/systemd/system/tft-gif.service

# 填入如下内容
[Unit]
Description=TFT GIF Player
After=network.target

[Service]
Type=simple
User=yyx
WorkingDirectory=/home/yyx/spi_test
ExecStart=/home/yyx/tftenv/bin/python3 /home/yyx/spi_test/play_gif.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

保存后执行：
```bash
sudo systemctl daemon-reload
sudo systemctl enable tft-gif.service # 开机启动
sudo systemctl start tft-gif.service # 立即启动
```

## 5. 成功

<img src="https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/5f7c97666d3758740e3a5f1804424882.png" alt="5f7c97666d3758740e3a5f1804424882" style="zoom:20%;" />

## 6. 后续计划

拓展一些展示功能，换个大一些的显示屏。同时最近感觉电子 DIY 很好玩，可能暑假想简单入个坑看看。

