#在CentOS 7中安装和配置OrientDB社区版

OrientDB是一种下一代多模型开源NoSQL DBMS。通过对多个数据模型的支持，OrientDB可以在一个可伸缩的高性能操作数据库中提供更多的功能和灵活性。

在本文中，笔者将演示如何在CentOS 7服务器实例上安装OrientDB社区版。

准备工作：

一个具有足够内存的Vultr CentOS 7 服务器 。推荐的内存为2GB或更多。假设它的IP地址是203.0.113.1。

您已经以sudo用户的身份登录到服务器实例。

服务器实例已经更新到最新的稳定状态。

 

##步骤1:安装OpenJDK 8包 OrientDB需要Java 1.7或更高版本。

在本文中，我选择安装OpenJDK 8包，如下所列:

sudo yum install -y java-1.8.0-openjdk-devel

安装了OpenJDK 8之后，使用下面的命令来验证结果:

java -version

如果没有出错，输出应该类似:

openjdk version "1.8.0_141"OpenJDK Runtime Environment (build 1.8.0_141-b16)OpenJDK 64-Bit Server VM (build 25.141-b16, mixed mode)

接下来，需要设置javahome环境变量:

echo "JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")" | sudo tee -a /etc/profilesource /etc/profile

##步骤2:安装OrientDB

从官方的OrientDB下载页面下载OrientDB社区版的最新稳定版本，从官方的OrientDB下载页面下载:

wget https://s3.us-east-2.amazonaws.com/orientdb3/releases/3.0.18/orientdb-3.0.18.tar.gz

将下载的存档解压到/opt目录:

sudo tar -zxvf orientdb-community-importers-2.2.37.tar.gz -C /opt

创建一个软链接，以简化日常使用和未来的更新:

sudo ln -s /opt/orientdb-community-importers-2.2.37/ /opt/orientdb

步骤3(可选):配置定向的OrientDB社区版本，以减少内存

尽管顺利运行的OrientDB社区版本要求您的机器有2GB或更多的内存，但是您仍然可以将它部署到一个具有较少内存的服务器上。

要做到这一点，请使用vi 文本编辑器打开/opt/orientdb/bin/server.sh文件:

sudo vim /opt/orientdb/bin/server.sh

找到:

ORIENTDB_OPTS_MEMORY="-Xms2G -Xmx2G"

如您所见，Xms和Xmx参数在运行OrientDB时指定了Java虚拟机的初始和最大内存分配池。为了减少对OrientDB的内存使用，您可以修改以下行:

ORIENTDB_OPTS_MEMORY="-Xms256m -Xmx512m"

注意:Xms的值不应该小于128m，否则OrientDB服务器将不会启动。

保存并退出:

:wq!

##步骤4:手动启动OrientDB服务器

通过在SSH终端窗口中执行 /opt/orientdb/bin/server.sh 脚本，您可以手动启动OrientDB服务器:

> sudo /opt/orientdb/bin/server.sh

因为这是您第一次运行OrientDB服务器，脚本会要求您为定向的root用户设置一个密码，比如这里是yourpasswordhere.。如果您将密码字段留空，该脚本将自动为OrientDBroot用户生成一个密码。这里创建的凭据将用于身份验证，当您使用二进制连接(OrientDB控制台)或web连接(OrientDB Studio)进行登录时。

 

如果正确地启动了OrientDB服务器，您将看到一条消息行:

> 2017-08-22 04:02:09:065 INFO  OrientDB Server is active v2.2.26 (build ae9fcb9c075e1d74560a336a96b57d3661234c7b). [OServer]

任何您想要退出的时候，按ctrl-c停止OrientDB服务器。

 

##步骤5:连接到OrientDB服务器

当OrientDB服务器启动并运行时，它将侦听端口2424(用于二进制连接)和端口2480(用于HTTP连接)。这意味着您不仅可以使用OrientDB的控制台，还可以使用web浏览器连接到正在运行的OrientDB服务器。

选项1:使用一个OrientDB控制台

保持服务器的SSH连接.sh脚本运行正常，然后在相同的服务器实例上建立第二个SSH连接。

在第二个SSH控制台窗口中，使用以下命令启动服务器上的OrientDB控制台:

> sudo /opt/orientdb/bin/console.sh

在控制台的shell中，连接到OrientDB服务器如下:

> orientdb> connect remote:127.0.0.1 root yourpasswordhere

如果成功连接到OrientDB服务器，您将看到下面的输出:

> Connecting to remote Server instance [remote:127.0.0.1] with user 'root'...OKorientdb {server=remote:127.0.0.1/}>

完成后，输入exit退出OrientDB控制台。

注意:您还可以使用本地console.sh (on Linux) 或者console.bat (on Windows)脚本连接OrientDB服务器。在这种情况下，您需要允许服务器2424端口上的入站通信。

> sudo firewall-cmd --zone=public --permanent --add-port=2424/tcpsudo firewall-cmd --reload

选项2:通过网络浏览器

连接OrientDB服务器的一种更直观的方法是使用web浏览器。

首先，您需要打开OrientDB服务器的2480端口，如下:

> sudo firewall-cmd --zone=public --permanent --add-port=2480/tcpsudo firewall-cmd --reload

接下来，将您最喜欢的web浏览器指向http://203.0.113.1:2480，然后您将被重定向到一个名为OrientDB Studio的页面。在这个页面上，您可以使用您之前设置的根用户凭证来登录。

在OrientDB Studio web界面中，您几乎可以完成在OrientDB控制台中所能做的所有事情。您可以自由地导航系统并测试您的查询。

步骤6:将OrientDB配置为服务器

在步骤2中，我们已经在/opt/orientdb-community-importers-2.2.37
目录中安装了OrientDB。但到目前为止，所有这些文件只是一堆脚本，这些脚本只能手动执行。为了设置操作服务器，需要将OrientDB配置为一个系统级守护进程，该守护进程将在系统引导中启动。

1)在第一个终端窗口中按ctrl-c停止OrientDB服务器。

2)创建一个专用的用户定向器，它属于OrientDB组，用于运行OrientDB服务器:

> sudo useradd -r orientdb -s /sbin/nologin

3)改变OrientDB目录的所有权:

> sudo chown -R orientdb:orientdb /opt/orientdb-community-importers-2.2.37

4)使用 vi编辑器打开/opt/orientdb/bin/orientdb.sh文件:

> sudo vim /opt/orientdb/bin/orientdb.sh

找到以下行：

ORIENTDB_DIR="YOUR_ORIENTDB_INSTALLATION_PATH"
ORIENTDB_USER="USER_YOU_WANT_ORIENTDB_RUN_WITH"

用下面的取代：

ORIENTDB_DIR="/opt/orientdb"
ORIENTDB_USER="orientdb"

保存并退出：

:wq!

5)为了防止未经授权访问OrientDB的配置，您需要修改对该配置文件的权限如下:

> sudo chmod 640 /opt/orientdb/config/orientdb-server-config.xml

6)创建一个systemd启动脚本来管理OrientDB服务:

> sudo cp /opt/orientdb/bin/orientdb.service /etc/systemd/system

使用vi编辑器打开这个文件:

> sudo vim /etc/systemd/system/orientdb.service

找到以下行:

User=ORIENTDB_USER
Group=ORIENTDB_GROUP
ExecStart=$ORIENTDB_HOME/bin/server.sh

用下面的取代：

User=orientdb
Group=orientdb
ExecStart=/opt/orientdb/bin/server.sh
保存并退出：

:wq! 启动并启用OrientDB服务:

这样，OrientDB将自动启动系统引导。
