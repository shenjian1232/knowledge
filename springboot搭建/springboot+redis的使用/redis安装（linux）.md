# linux下安装redis及其中遇到的问题的解决方法
### 1.将下载好的压缩包放到/usr/local目录下
```
# tar xzf redis-3.0.2.tar.gz
# cd redis-3.0.2
# make
```

```
提示错误 make: cc: Command not found make: *** [adlist.o] Error 127
没有安装gcc环境，需要安装gcc
# yum install gcc
安装后检查是否安装成功
# rpm -qa |grep gcc
之后重新make
```
### 2.编译完成后，在Src目录下，有四个可执行文件redis-server、redis-benchmark、redis-cli和redis.conf将其拷贝到一个目录下。
```
# mkdir /usr/redis
# cp redis-server  /usr/redis
# cp redis-benchmark /usr/redis
# cp redis-cli  /usr/redis
# cp redis.conf  /usr/redis
# cd /usr/redis
```
### 3.启动服务
```
# redis-server   redis.conf
```

```
提示错误 -bash :redis-server:command not found
建立软连接
# ln -s /usr/redis/redis-server /usr/bin/redis-server
# ln -s /usr/redis/redis-cli /usr/bin/redis-cli
重新启动
# redis-server /usr/redis/redis.conf

```

## 如安装不成功请自行百度安装