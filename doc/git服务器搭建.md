## GIT服务器搭建（用gitea）

很早以前就想试着自己搭一个git服务器了,这次终于付诸行动。

这里选择用gitea搭建git服务器（很方便界面也好看）

参考了一个大佬的youtube视频https://www.youtube.com/watch?v=shxiz_bos3I

大佬的github是这个https://github.com/appleboy

话不多说

### Step One

首先下载git（这里用的是ubuntu服务器）

```json
apt-get update
apt-get install git
git config --global user.name "sada"
git config --global user.email "asdasd@qq.com"
```

### Step Two

这里是具体步骤了，界面方面的操作就不细说了，很简单。

```json
wget -O gitea https://dl.gitea.io/gitea/1.5.0/gitea-1.5.0-linux-amd64
chmod +x gitea
useradd -m git
cp gitea /home/git/
su - git
./gitea web
```

这里创建gitea时最好用sqlite

### Step Three

这里用screen让进程在后台运行

```json
git config --global credential.helper store //只要输入一次账号密码
su - root
apt-get install screen

screen -S "name"//创建会话窗口
screen -ls
screen -r "name或者4790" //切回到该会话窗口
screen -d "name或者4790" //远程detach会话窗口
screen -X -S "name或者4790" quit

Ctrl+a+c(切换并创建新的窗口)
ctrl+a+d(退出当前会话到没screen状态)
```

补充：
搭建一个超级简陋朴素的git服务器(由于很简单就只记录基本的操作啦)
```json
//添加用户
groupadd git
adduser git -g git
passwd git
vi /etc/passwd
git:x:1000:1000::/home/git:/usr/bin/git-shell

//赋权限
mkdir /home/git/.ssh
cd /home/git/
chmod 700 .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
git init --bare test.git
chown -R git:git test.git

//使用
git clone git@host:/home/git/test.git
```
