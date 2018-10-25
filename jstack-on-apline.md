### jstack on alpine：Unable to get pid of LinuxThreads manager thread 
>在docker 中使用jstack 报错：
```bash
bash-4.3# jps
112 Jps
7 jar
bash-4.3# jstack 7
7: Unable to get pid of LinuxThreads manager thread
```

>解决方法：尝试把你的dockerfile 中启动java的方式改为以下方法：
```docker
ENTRYPOINT ["/bin/bash", "-c", "set -e && java -Xmx100m -jar /demo.jar"]
```
>以下是相关信息
>系统版本：
```bash
bash-4.3# cat /etc/issue
Welcome to Alpine Linux 3.5
Kernel \r on an \m (\l)
```
>java 版本：
```bash
bash-4.3# java -version
openjdk version "1.8.0_121"
OpenJDK Runtime Environment (IcedTea 3.3.0) (Alpine 8.121.13-r0)
OpenJDK 64-Bit Server VM (build 25.121-b13, mixed mode)
```

>glibc信息：
```bash
bash-4.3# apk info | grep glibc
bash-4.3# 
```




