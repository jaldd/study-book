1. ### 下载安装item2

2. ### 配置主题

3. ### lrzsz配置

   [lrzsz安装](../linux/服务器文件上传下载.md)完成后我们需要在 iTerm2 中使用的话，还需要一些配置

   进入到 /usr/local/bin 目录下，下载两个脚本文件

   ```javascript
   cd /usr/local/bin 
   sudo wget https://gist.githubusercontent.com/sy-records/1b3010b566af42f57fa6fa38138dd22a/raw/2bfe590665d3b0e6c8223623922474361058920c/iterm2-send-zmodem.sh 
   sudo wget https://gist.githubusercontent.com/sy-records/40f4ba22e3fbdeedf58463b067798962/raw/b32d2f7ac3fa54acca81be3664797cebb724690f/iterm2-recv-zmodem.sh
   sudo chmod 777 /usr/local/bin/iterm2-* 
   ```

   下载好之后我们进行 iTerm2 的配置

   点击 iTerm2 的设置界面 Perference -> Profiles -> Default -> Advanced -> Triggers 的 Edit 按钮，点击+号，添加如下的参数

   ```javascript
   Regular expression: rz waiting to receive.\*\*B0100
               Action: Run Silent Coprocess
           Parameters: /usr/local/bin/iterm2-send-zmodem.sh
              Instant: checked
   
   Regular expression: \*\*B00000000000000
               Action: Run Silent Coprocess
           Parameters: /usr/local/bin/iterm2-recv-zmodem.sh
              Instant: checked
   ```

   