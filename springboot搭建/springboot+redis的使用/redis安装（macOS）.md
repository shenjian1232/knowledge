# Mac环境下安装Redis
### 一、首先是官网下载redis
```
自行下载
```
### 二、安装与编译
```
解压：tar zxvf redis-4.0.10.tar.gz
移动到： mv redis-4.0.10 /usr/local/
切换到：cd /usr/local/redis-4.0.10/
编译测试 sudo make test
编译安装 sudo make install
```
### 三、启动redis
```
输入redis-server启动redis
```