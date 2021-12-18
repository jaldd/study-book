1. 安装ssh-copy-id

```
curl -L https://raw.githubusercontent.com/beautifulcode/ssh-copy-id-for-OSX/master/install.sh | sh

```

2. 生成本机的密钥

```
ssh-keygen -t rsa

```

这时会在当前用户的根目录下生成.ssh文件夹，也就是~/.ssh，如果是首次执行会在.ssh文件中生成id_rsa以及id_rsa.pub文件。


3. 将本地密钥copy到目标服务器中

```
ssh-copy-id -i ~/.ssh/id_rsa.pub root@*.*.*.*

```

