 #搭建服务器的配置
  u~~~~buntu16.04
  内存4G（内存消耗很大）
  gitlab版本为最新的版本（建议gitlab不要中文化，因为中文会导致一大堆的bug出现。）
  VMware6.5
 #远程连接虚拟机（安装虚拟机和ubuntu略过）    
    1. 首先，打开VMware Workstation，在已经创建好的虚拟机“TEST 虚拟机创建”上面编辑虚拟机设置，点击“网络适配器”，弹出窗口在网络连接中选择“桥接模式（B）直接连接物理网络”
    2. 打开VMware Workstation，点击编辑菜单：选择“虚拟网络编辑器”，选择“VMnet0”桥接模式自动设置即可.
    3. 通过 ping ip 检查是否能远程连接。
    3. 如果不行就要检查需要连接的电脑是否是在同一个网关上。
 #在安装gitlab前需要对ubuntu服务器安装一些依赖包
```
sudo apt-get install curl openssh-server ca-certificates postfix
```
 #利用清华大学的镜像（https://mirror.tuna.tsinghua.edu.cn/help/gitlab-ce/）来进行主程序的安装。 
  
  首先信任 GitLab 的 GPG 公钥:
  ```
  curl https://packages.gitlab.com/gpg.key 2> /dev/null | sudo apt-key add - &>/dev/null
  ```
  利用root用户[1]（不是sudo，而是root），vi打开文件/etc/apt/sources.list.d/gitlab-ce.list，加入下面一行：
  ```
  deb https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu xenial main
  ```
  安装gitlab-ce:
  ```
  sudo apt-get update
  sudo apt-get install gitlab-ce
  ```
  配置GitLab IP地址，首先运行
  ```
  sudo -e /etc/gitlab/gitlab.rb
  ```
  在文本中修改”externval_url”之后的域名，可以是本机IP或指向本机IP的域名（注意要带有“https://”），这一行在全部文本中位于很靠上面的位置。
  但是我都是写的是externval_url="https:localhost"
  接着执行
  ```
  sudo gitlab-ctl reconfigure
  ```
#ssh和postfix检查和启动
```
service sshd status
service postfix status
service sshd start 
service postfix start 
```
#能够远程连接gotlab界面
 关闭ubuntu的防火墙
 ```
apt-get install ufw      安装防火墙
sudo ufw enable|disable|status         开启/关闭/查看防火墙状态
sudo ufw allow 22/tcp        开通22端口
sudo ufw allow 80/tcp         开通80端口
sudo ufw allow 3306/tcp        开通3306端口
sudo ufw allow from 192.168.1.100      允许此ip访问本机的所有端口
sudo ufw delete allow from 192.168.1.25       禁止这个ip访问本机
sudo ufw allow 80/tcp from 192.168.1.0/24      设置本机80端口访问的白名单：只允许192.168.1.0网段的ip访问本机80端口
sudo ufw delete allow 80/tcp      禁用本机的80端口
```
#检验远程是否能够登录gitlab服务器
   https://ip:port
   说明gitlab服务器搭建完成。
