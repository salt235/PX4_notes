# 基于树莓派的 DIY NAS

参考项目：https://the-diy-life.com/building-a-4-bay-3-5-nas-with-a-raspberry-pi-5-and-3d-printed-enclosure/

开源3D打印壳：https://makerworld.com/zh/models/1605027-raspberry-pi-5-based-4-bay-nas#profileId-1697964

参考资料：https://docs.radxa.com/en/accessories/storage/penta-sata-hat

## 高颜值实物图



## 配件清单及花费

| 序号 | 项目                   | 金额（元）  | 备注                             |
| ---- | ---------------------- | ----------- | -------------------------------- |
| 1    | 树莓派本体及相关配件   | 850.00      | 闲鱼二手收的                     |
| 2    | Penta SATA HAT         | 230.00      | 闲鱼全新                         |
| 3    | 铠侠二手 SSD           | 100.00      | 收的朋友的                       |
| 4    | 3D 打印外壳            | 141.50      | 找闲鱼哥们定制了颜色，打印了很久 |
| 5    | 小风扇                 | 3.50        | 买错接口，浪费了                 |
| 6    | DC 延长线              | 2.90        |                                  |
| 7    | 热熔螺母               | 1.30        |                                  |
| 8    | 各种螺丝螺母           | 5.11        |                                  |
| 9    | 12V 6A 电源            | 21.80       | 72W                              |
| 10   | 树莓派主动散热器       | 10.87       | 自带的和hat不兼容，我重买了一个  |
| 11   | 新风扇                 | 9.90        | 大4pin接口（母）                 |
| 12   | SATA 延长线两根        | 8.44        |                                  |
| 13   | 绿联硬盘盒             | 22.61       |                                  |
| 14   | 3.5寸西数紫盘 * 2      | 0           | 白嫖的全新，可惜不是红盘         |
|      | **总计**               | **1407.93** |                         

1400出头，还行，主要白嫖了两个硬盘没花钱，里面的所有配件都是参考的[该开源项目博客](https://the-diy-life.com/building-a-4-bay-3-5-nas-with-a-raspberry-pi-5-and-3d-printed-enclosure/)。

其中只有一个风扇和它不一样，原文是用的**小3pin母口的风扇**加一个**小3pin公口转大4pin母口**的转接线连到SATA HAT上。而我花了大量精力在国内各个平台找了好久，完全找不到**小3pin公口转大4pin母口**的转接线，最后偶然发现根本不需要转接线，直接买个大4pin母口的风扇就行。

## 组装过程

## 基础配置

### 禁用开机的电流检查

用上12V 72W的电源后，树莓派是通过SATA HAT的40Pin供电的，所以和树莓派的官方5V 27W的电源有区别。我的树莓派从 USB 硬盘启动后，检测到的整机供电能力只有 **3000 mA**，因此每次开机都停下来要求按钮确认。

编辑 `/boot/firmware/config.txt`，把 `usb_max_current_enable=1` 添加到文件末尾，保存后重启。

之后从 USB 硬盘启动时，就不会再停在这个确认界面了。该参数会重新允许 USB 启动，并把树莓派 5 的 USB 总电流限制从约 **600 mA 提高到 1600 mA**。

### 开启PCIe

编辑 `/boot/firmware/config.txt`，把 `dtparam=pciex1` 添加到文件末尾，保存后重启。

### 查看硬盘情况

```bash
lsblk
```
我目前是这样的结构：sda和sdb是两个3.5寸HDD，通过SATA HAT利用PCIe连接；另外sdc是一个2.5寸的SSD，通过usb连接树莓派作为一个系统盘。

```bash
sda # 3.5寸数据盘1
sdb # 3.5寸数据盘2
sdc # 2.5寸系统盘
├─sdc1 vfat FAT32 system-boot
└─sdc2 ext4 writable /
```

### 清理并格式化硬盘

```bash
# 清理
sudo wipefs -a /dev/sda
sudo wipefs -a /dev/sdb
# 创建 GPT
sudo parted /dev/sda mklabel gpt
sudo parted /dev/sdb mklabel gpt
# 创建分区
sudo parted -a optimal /dev/sda mkpart primary ext4 0% 100%
sudo parted -a optimal /dev/sdb mkpart primary ext4 0% 100%
# 格式化
sudo mkfs.ext4 /dev/sda1
sudo mkfs.ext4 /dev/sdb1
```

