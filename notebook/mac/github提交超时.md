### 配置host

---

https://zhuanlan.zhihu.com/p/107334179?ivk_sa=1024320u



https://ipaddress.com/website/github.com

https://ipaddress.com/website/github.global.ssl.fastly.net

https://ipaddress.com/website/assets-cdn.github.com

```shell
140.82.113.4 github.com
199.232.69.194 github.global.ssl.fastly.net
185.199.108.153 assets-cdn.github.com
185.199.109.153 assets-cdn.github.com
185.199.110.153 assets-cdn.github.com
185.199.111.153 assets-cdn.github.com
```

---



### 配置ssh访问

---

```
cd ~/.ssh

ssh-keygen -t rsa -C '806616508@qq.com' -f ~/.ssh/id_rsa_github 

cat ~/.ssh/id_rsa_github.pub 
copy到github
ssh-add ~/.ssh/id_rsa_github 

vim config

Host github
    AddKeysToAgent yes
    HostName github.com
    User 806616508@qq.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa_github
    
ssh -T git@github.com
```

历史项目需要修改.git/config文件

---



### 解决ssh-add每次重启都需要再执行的问题

---

1. 创建一个Automator 应用程序类型文件
2. 选择运行shell脚本，在输入框输入ssh-add命令,点击顶部未命名保存
3. 打开系统偏好设置-》用户与群组，选择登录项，选择保存的.app文件为开机启动