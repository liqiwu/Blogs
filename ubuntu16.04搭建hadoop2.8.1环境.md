# ubuntu16.04搭建hadoop2.8.1环境详解 #
###参考博文：
[http://www.cnblogs.com/lighten/p/6106891.html]()     
   
[http://www.cnblogs.com/lighten/p/6105463.html]()

[http://blog.csdn.net/a925907195/article/details/52624935]()


最近在帮师姐搭建hadoop平台，顺便记录一下这个学习过程，参考博文如上，结合自己的试验过程整理总结一下，欢迎吐槽，批评，指正！

## 一、	JDK安装 ##
###简单安装
安装JDK的最简单方法应该就是使用apt-get来安装了，安装ubuntu后一般都具有源OpenJDK


1.使用ctrl+alt+t打开终端，你可以添加一个含有OpenJDK源的仓库，一般是不需要，因为一般都有。

备份原始源文件：

	cp /etc/apt/sources.list /etc/apt/sources.list.bak.1

	vim /etc/apt/sources.list

修改里面的源就行了。


2.更新系统安装包缓存，并且安装OpenJDK8
	sudo apt-get update

	sudo apt-get install openjdk-8-jdk


3.如果你系统中存在多个版本的JDK，使用下列命令设置一个默认的JDK

	sudo update-alternatives --config java

	sudo update-alternatives --config java

输入选择的java版本的编号


4.最后检查当前的java版本查看是否编译成功

`java -version`

###手动安装oracle JDK

![](https://i.imgur.com/YIA7QyE.png)

1.去oracle官网下载。[点此下载](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

2.解压 

`tar -zxvf jdk-8u144-linux-x64.tar.gz`

3.移动到自己想放的位置：

	mkdir /usr/local/java

	mv jdk1.8.0_144  /usr/local/java    //解压在该文件夹下后并改文件夹名为jdk

4.设置环境变量：

方案一：修改全局配置文件，作用于所有用户：

	vim /etc/profile 

	export JAVA_HOME=/usr/local/java/jdk

	export JRE_HOME=${JAVA_HOME}/jre

	export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
	
	export PATH=.:${JAVA_HOME}/bin:$PATH

方案二：修改当前用户配置文件，只作用于当前用户：

`vim ~/.bashrc` 

5.使修改的配置立刻生效：

`source /etc/profile` 或者 `source ~/.bashrc`

6.检查是否安装成功：`java -version`

##2.配置SSH免密登录

hadoop需要使用SSH的方式登陆，linux下需要安装SSH。客户端已经安装好了，只需要安装服务端就可以了：

`sudo apt-get install openssh-server`

测试登陆本机 ssh localhost 输入yes就应该可以登录了。但是每次输入比较繁琐，如果是集群那就是灾难了，所以要配置成免密码登陆的方式。

一共有三步：

1.生成公钥私钥 `ssh -keygen -t rsa`，将在`~/.ssh`文件夹下生成文件`id_rsa`：私钥，`id_rsa.pub`：公钥

2.导入公钥到认证文件，更改权限：

1）导入本机：`cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`

2）导入服务器：

首先将公钥复制到服务器：

`scp ~/.ssh/id_rsa.pub xxx@host:/home/xxx/id_rsa.pub` 
 
然后，将公钥导入到认证文件，这一步的操作在服务器上进行：

`cat ~/id_rsa.pub >> ~/.ssh/authorized_keys` 

最后在服务器上更改权限：

	chmod 700 ~/.ssh

	chmod 600 ~/.ssh/authorized_keys
  
3）测试：ssh localhost 第一次需要输入yes，之后就不需要了。

**注意**：这只是允许当前用户可免密登录localhost，在切换到root用户下，再用ssh登录localhost是行不通的，还是需要输入密码。

ubuntu默认的root密码似乎是随机的，需要修改root密码的话，输入

`$sudo passwd root`

但是修改了之后，对于`root@localhost's password`：还是没有办法。

此时修改ssh配置文件

`$sudo gedit /etc/ssh/sshd_config`

注释掉

`PermitRootLogin prohibit-password`

改成

`PermitRootLogin yes`

然后重启ssh服务

`$service sshd restart`

尝试用root连接localhost

![](https://i.imgur.com/oVu1WIs.png)

即可连接成功

##3.HADOOP安装

1.下载hadoop安装包，[下载地址](http://hadoop.apache.org/releases.html)

2.解压、移动到你想要放置的文件夹

	tar -zvxf hadoop-2.8.1.tar.gz

	mv ./hadoop-2.8.1.tar.gz   /urs/local/hadoop

3.创建hadoop用户和组，并授予执行权限

	sudo addgroup hadoop

	sudo usermod -a -G hadoop xxx 　　//将当前用户加入到hadoop组

	sudo gedit etc/sudoers　　//将hadoop组加入到sudoer

在`root ALL=(ALL：ALL) ALL`后` hadoop ALL=(ALL:ALL) ALL`

	sudo chmod -R 755 /urs/local/hadoop

	sudo chown -R xxx:hadoop /usr/local/hadoop //否则ssh会拒绝访问 

**注意**：由于hadoop文件夹下的目录访问是需要root权限的，但是我们没有root权限怎么办，如果一直在root权限下操作也未尝不可，我们之前ssh也设置好了root可登陆localhost，但需要每次输入密码。但是一般配置的都是当前用户ssh免密登录localhost，所以我们只需要将hadoop文件夹下的目录设置为当前用户可操作访问即可，不用每次都用sudo行驶root权限啦。
其实，一切都很简单：
　
`sudo chown -R xxx:hadoop /usr/local/hadoop`

即是将/usr/local/hadoop及其所有内容的拥有者改为普通用户xxx即可（前提当前用户在hadoop用户组下），否则用

`sudo chown -R ××× /usr/local/hadoop`

4.修改配置文件，和JDK的安装一样，可以选择修改哪个文件。这里可以修改/etc/profile

	export HADOOP_HOME=/usr/local/hadoop

	export PATH=.:${JAVA_HOME}/bin:${HADOOP_HOME}/bin:$PATH

	source /etc/hadoop

当然，也可修改当前用户配置文件：`sudo gedit ~/.bashrc`

在文件末尾添加：
		
    #HADOOP VARIABLES START  
      
    export JAVA_HOME=/usr/local/java/jdk       
    export HADOOP_HOME=/usr/local/hadoop       
    export PATH=$PATH:$HADOOP_HOME/bin  
    export PATH=$PATH:$JAVA_HOME/bin   
    export PATH=$PATH:$HADOOP_HOME/sbin  
    export HADOOP_MAPRED_HOME=$HADOOP_HOME  
    export HADOOP_COMMON_HOME=$HADOOP_HOME  
    export HADOOP_HDFS_HOME=$HADOOP_HOME  
    export YARN_HOME=$HADOOP_HOME  
    export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native  
    export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib"  
    #HADOOP VARIABLES END  

然后运行：`source ~/.bashrc`使改动生效

5.测试是否配置成功

`hadoop version`

**测试**

通过执行hadoop自带实例WordCount验证是否安装成功

在/usr/local/hadoop路径下创建input文件夹

	mkdir input
	
	cp README.txt input

在hadoop目录下执行WordCount：

    bin/hadoop jar share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.8.1-sources.jar org.apache.hadoop.examples.WordCount input output

6.hadoop伪分布配置

该[博文](http://blog.csdn.net/ggz631047367/article/details/42426391#t8)对伪分布配置有详细介绍，包括可能出现的问题以及解决办法。

##4.总结

本人从安装Ubuntu开始到配置好Hadoop单机环境足足花了一周左右，由于使用的是自己笔记本安装的win7 64位+ubuntu16.04双系统，配置过程中掉进不少坑，参考的文章也不少，也算是从小白进阶到菜鸟吧，可能菜鸟都算不上。特此在基于各路大神的一再总结下，本人再次记录这个学习过程，有问题，请指教！