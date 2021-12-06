1. ### yum安装

   ```
   yum search jdk
   yum install java-1.8.0-openjdk-devel.x86_64 （其他包会没有jre）
   ```

   yum安装的默认路径为：/usr/lib/jvm

   ```
   vi /etc/profile
   JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-2.el8_5.x86_64
   PATH=$PATH:$JAVA_HOME/bin
   CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
   export JAVA_HOME CLASSPATH PATH
   
   
   ```

   保存退出 `source /etc/profile`

2. 