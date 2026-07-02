# wsl部署px4环境

## 1. BIOS中开启虚拟化支持，并安装wsl

```powershell
wsl --install
```

## 2. 安装Ubuntu 22.04

```powershell
wsl --install -d Ubuntu-22.04
```

## 3. 迁移到非系统盘

### 导出当前发行版

```powershell
wsl --export Ubuntu-22.04 D:\WSL\ubuntu.tar
```

### 注销原发行版

```powershell
wsl --unregister Ubuntu-22.04
```

### 导入目标路径

```powershell
wsl --import Ubuntu-22.04 D:\WSL\Ubuntu D:\WSL\ubuntu.tar --version 2
```

### 验证结果

```powershell
wsl --list --verbose
```

至此全部迁移到d盘的wsl文件夹中。

## 4. 初始化ubuntu

```bash
# 系统基础更新
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y

# 配置镜像
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo nano /etc/apt/sources.list
# 按Ctrl + -,然后狂按Ctrl + k，删除全部内容

# 替换为如下：
	# 默认阿里云镜像源
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
	# 源码仓库（可选）
deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
# 保存退出：按Ctrl+X → 输入Y → 回车确认
# 修改后执行以下命令使更改生效：
sudo apt update
# sudo apt --fix-broken install 若出现依赖问题，可运行修复命令
```

## 5. 安装PX4工具链

### 拉取PX4源码

```bash
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
```

### 运行**ubuntu.sh**安装脚本，并根据脚本运行过程中的任何提示进行确认

```bash
bash ./PX4-Autopilot/Tools/setup/ubuntu.sh
```

### 重启

```bash
exit
wsl --shutdown
wsl
```

### 进入PX4存储库，然后构建并测试：

```bash
cd ~/PX4-Autopilot
make px4_sitl
```

## 6. 安装Win版本的QGC并与wsl连接

1. 先去官网下载win版本的QGC，运行以下命令检查 WSL 虚拟机的 IP 地址`ip addr | grep eth0`：

```bash
ip addr | grep eth0

6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 172.18.46.131/20 brd 172.18.47.255 scope global eth0
```

`eth0`将接口地址的第一部分复制`inet`到剪贴板。在本例中：`172.18.46.131`。

2. 在 QGC 中，转到**Q > 应用程序设置 > 通信链接**
3. 添加一个名为“WSL”的UDP链路，port填14550，server填wsl的IP地址加:18570。
4. 保存并连接。
5. 每次 WSL 重启后，您都需要在 QGC 中**更新** WSL 通信链路（因为它会获得**动态 IP 地址**）。

## 7. 配置clash代理（可选）

参考：https://github.com/nelvko/clash-for-linux-install

配置好之后：clashctl help查看帮助