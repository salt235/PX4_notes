# 树莓派轻量NAS部署

最近心血来潮，海鲜市场收了个4G的树莓派5，主要想用来学习和折腾。我接了个铠侠的256G 2.5寸 SSD，把树莓派的SD卡拔了，系统装在了SSD上，装了个桌面板的Ubuntu24.04。但后来感觉用桌面还是有点卡，就把桌面关掉了，还是纯命令行舒服。

现在这个树莓派就如下图所示，躺在我的工位上，接了个网线24h运行，同时我配了一下tailscale，现在我的所有设备都在一个虚拟局域网下，这个派直接就化身为一个小服务器了。这次我试着部署了个轻量的NAS，类似于一个虚拟文件夹吧，用的一个叫Samba的项目。

![树莓派](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/%E6%A0%91%E8%8E%93%E6%B4%BE.png)

## 1. Tailscale 网络状态验证

```bash
tailscale status
```

看设备与树莓派之间的连接，可能为：

- `direct` → 点对点直连（高速）

- `relay hkg` → 香港中继（较慢）

## 2. Samba 文件服务配置（只用在树莓派端配置）

```bash
sudo apt update
sudo apt install samba -y

# 创建共享目录
mkdir -p /home/yyx/data
# 设置用户
sudo smbpasswd -a yyx
# 用nano将以下内容写入/etc/samba/smb.conf最后
[data]
   path = /home/yyx/data
   browseable = yes
   read only = no
   valid users = yyx
# 设置文件夹权限
sudo chmod -R 777 /home/yyx/data
# 启动服务
sudo systemctl restart smbd
sudo systemctl enable smbd
# 防火墙配置
sudo ufw allow samba
```

## 3. 客户端访问方式

### Windows

直接在文件管理器输入地址：

```
\\yyx-pi\data
```

然后输入账号密码就行。

### macOS

```
smb://yyx-pi/data
```

以上也适用于ios的文件app。

## 4. 实际性能测试结果

### 校园网环境（direct）

- 下载（Pi → PC）：≈ 80 MB/s
- 上传（PC → Pi）：≈ 10 MB/s
- 目前最快最流畅，拖进度条也很快

### 出租屋垃圾网络（relay hkg）

- 自动走 Tailscale 中继
- 速度下降明显，基本上看不了视频

### 手机热点环境（direct）

- 成功直连
- 速度明显提升
- 可较流畅播放视频
- 拖动进度条略有延迟

## 5. 问题与分析

### 速度不对称

- 下载快：校园网下行带宽高

- 上传慢：校园网/出租屋上行带宽低

### 手机热点拖进度条慢

看视频顺序播放时，播放器只需要持续读取后面的一小段数据；但拖进度条时，它要立刻从 NAS 的另一个位置重新请求数据，SMB + Tailscale + 远程链路会有额外延迟，所以 seek 会比本地文件慢。

### 如何有更好的体验

如果主要是看电影，可以把视频目录挂载后用播放器直接打开，而不是先在文件管理器里预览。文件管理器缩略图、属性读取有时也会额外拖慢 SMB。

