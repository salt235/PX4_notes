# ssh常用命令

> SSH（Secure Shell）可以理解成在另一台电脑上打开一个终端。

## 登录

```bash
ssh 用户名@IP
```

如果配置了tailscale，甚至还可以

```bash
ssh 用户名@主机名
```

## 上传文件

```bash
scp 文件 用户@IP:目录
```

例如：

```bash
scp test.txt yyx@192.168.1.100:~
```

## 上传文件夹

```bash
scp -r notes yyx@192.168.1.100:~
```

## 下载文件

```bash
scp yyx@192.168.1.100:~/test.txt .
```