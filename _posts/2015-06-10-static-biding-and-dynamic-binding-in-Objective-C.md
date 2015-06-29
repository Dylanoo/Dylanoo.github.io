---
layout: post
title:  Mac OSX安装 GitLab 5.x 
date:   2015-06-09  21:00:11
category: "Git"
---



1)安装mac

2) 创建git用户和git组

4) 安装XCode

5) 安装命令行组件

6) 安装 Home brew

{% highlight Objective-C %} 
$ ruby -e 
"$(curl -fsSL 
https://raw.github.com/mxcl/homebrew/go)"

$ brew doctor
{% endhighlight %}

7) 安装一些依赖关系



$ brew install icu4c # 被charlock_holmes
gem依赖


$ brew install git


$ brew install redis


$ mkdir ~/Library/LaunchAgents # create launchAgents dir (it does
not exist in fresh system)


$ ln -sfv /usr/local/opt/redis/homebrew.mxcl.redis.plist
~/Library/LaunchAgents/homebrew.mxcl.redis.plist


$ launchctl load
~/Library/LaunchAgents/homebrew.mxcl.redis.plist
确保有python 2.5+ (gitlab 不支持 python 3.x)


$ python --version # 查看 python
版本


$ sudo ln -s /usr/bin/python /usr/bin/python2 #GitLab 需要

python2
其它依赖关系



$ sudo easy_install
pip


$ sudo pip install
pygments
8) brew install mysql #也可以从官方下载, 但是我在试验的时候官方的包有问题, 导致部署数据库的时候出错

9) 安装mysql

启动mysql服务并更改root密码


$ /usr/local/mysql/bin/mysqladmin -u root password
<密码>

确保 mysql 在 PATH 里


$ export PATH=/usr/local/mysql/bin:$PATH

登陆mysql



$ mysql -u root -p




#创建新用户 gitlab




mysql> CREATE USER 
'gitlab'@'localhost'

IDENTIFIED BY 
'密码';




#创建数据库


mysql> CREATE DATABASE IF NOT EXISTS `gitlabhq_production`
DEFAULT CHARACTER SET `utf8` COLLATE
`utf8_unicode_ci`;




# 授权


mysql> GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE,
CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO

'gitlab'@'localhost';




# 退出


mysql> \q




# 试着连接一下


sudo -u git -H mysql -u gitlab -p -D
gitlabhq_production
10) 安装 Ruby (用rbenv和rvm都可以)



$ brew install rbenv


$ brew install ruby-build





$ echo 
'export PATH="/usr/local/bin:$PATH"' 
| sudo -u git tee -a /Users/git/.profile


$ echo 
'if which rbenv > /dev/null; then eval "$(rbenv init -)";
fi' 
| sudo -u git tee -a /Users/git/.profile


$ sudo -u git cp /Users/git/.profile
/Users/git/.bashrc





# 安装ruby 1.9.3 (记得改为taobao源)


$ sudo -u git -H -i rbenv install 1.9.3-p448


$ sudo -u git -H -i 
'rbenv global 1.9.3-p448'

给自己也安装ruby 1.9.3.


$ rbenv install 1.9.3-p392


$ rbenv global 1.9.3-p392
11) 安装 rails 


$ sudo gem install rails

12) 安装Gitlab Shell



$ cd /Users/git


$ sudo -u git git clone https://github.com/gitlabhq/gitlab-shell.git


$ cd
gitlab-shell


$ sudo -u git cp config.yml.example
config.yml
打开 config.yml 文件并修改
a) 更改gitlab url 为 你的 URL (如果没有可以不改)
b) 更改 “/home” 为 “/Users” (Mac下的/home)
c) 更改 redis-cli 到 home-brew的 cli 路径 – “/usr/local/bin/redis-cli”

13) 安装GitLab -

下载Gitlab -



$ cd /Users/git


$ sudo -u git git clone https://github.com/gitlabhq/gitlabhq.git
gitlab


$ cd gitlab
配置GitLab


$ sudo -u git cp config/gitlab.yml.example
config/gitlab.yml

打开 gitlab.ym l更改 “/home” 为 “/Users” 确保更改 “gitlab.example.com” 为你的domain(如果没有可以不改)



# 确保 GitLab 能对 log/ and tmp/
进行读写


$ sudo chown -R git
log/


$ sudo chown -R git
tmp/


$ sudo chmod -R
u+rwX  log/


$ sudo chmod -R
u+rwX  tmp/




# 创建仓库目录


$ sudo -u git mkdir
/Users/git/repositories


$ sudo chmod -R
u+rwX 
/Users/git/repositories/




# 创建satellites


$ sudo -u git mkdir
/Users/git/gitlab-satellites




# 创建 pids 和
sockets


$ sudo -u git mkdir
tmp/pids/


$ sudo -u git mkdir
tmp/sockets/




$ sudo chmod -R
u+rwX  tmp/pids/


$ sudo chmod -R
u+rwX  tmp/sockets/




# 创建备份所使用的目录


$ sudo -u git mkdir 
public/uploads


$ sudo chmod -R u+rwX  
public/uploads




#
复制Puma配置文件(没有可忽略)


$ sudo -u git cp
config/puma.rb.example config/puma.rb
打开 puma.rb 修改 “/home” 为 “/Users”



# 配置 Git global
配置


$ sudo -u git -H git config --global user.name 
"GitLab"


$ sudo -u git -H git config --global user.email 
"gitlab@gitlab.example.com"
配置Gitlab的mysql信息.


$ sudo -u git cp config/database.yml.mysql
config/database.yml

修改 database.yml 里的 production 区域里的数据库账户和密码 (之前新建的那个gitlab用户)

安装 Gems





$ sudo gem install charlock_holmes --version 
'0.6.9.4'


$ sudo -u git -H bash -l -c 
'gem install bundler'


$ sudo -u git -H bash -l -c 
'bundle install --deployment --without development test
postgres'
部署 Database


$ sudo -u git -H bash -l -c 
'bundle exec rake gitlab:setup
RAILS_ENV=production'

检查 GitLab 的安装

瞅瞅环境信息


sudo -u git -H bash -l -c 
'bundle exec rake gitlab:env:info RAILS_ENV=production'

做个检查, 确保所有项目都是绿色, 如果有错误, 根据报错信息修改


sudo -u git -H bash -l -c 
'bundle exec rake gitlab:check RAILS_ENV=production'

14) 启动 gitlab


$ rails s -e production -p 12345

15) 打开 http://localhost:12345

login………admin@local.host
password……5iveL!fe

15) 创建仓库

如何创建仓库可以参考如下博文: http://blog.csdn.net/passion_wu128/article/details/8218041

原文: http://www.makebetterthings.com/git/install-gitlab-5-3-on-mac-os-x-server-10-8-4/