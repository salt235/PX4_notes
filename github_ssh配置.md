# github_ssh配置

## 检查是否存在ssh key

```powershell
ls ~/.ssh
```

如果能看到：

```
id_ed25519 私钥，不能泄露
id_ed25519.pub 公钥，可以添加到 GitHub
```

那就说明已经存在了。

## 生成ssh key

```powershell
ssh-keygen -t ed25519 -C "your_email@example.com"
```

然后连按三次回车，之后可以用下面的命令查看。

```powershell
cat ~/.ssh/id_ed25519.pub
```

得到的就是公钥，复制这一行。

## 在github添加

打开`https://github.com/settings/keys`，点击新建，输入名字和秘钥。

```powershell
ssh -T git@github.com
```

以上命令可以测试秘钥，之后会问你：

```powershell
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入：yes

```powershell
Hi salt235! You've successfully authenticated, but GitHub does not provide shell access.
```

看到这个就OK了。



