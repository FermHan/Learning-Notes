###使用beeline工具连接Hive

####第一步:修改hadoop的hdfs-site.xml文件

 在该文件中添加以下内容,开启HDFS的REST接口功能:

~~~xml
<property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
</property>
~~~

####第二步:修改hadoop的core-site.xml文件

在文件中添加以下内容,设置代理用户:

~~~xml
<property>
    <name>hadoop.proxyuser.root.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.root.groups</name>
    <value>*</value>
</property>
~~~

####第三步:重启Hadoop集群

~~~shell
scp hdfs-site.xml node02:$PWD
scp hdfs-site.xml node03:$PWD

scp core-site.xml node02:$PWD
scp core-site.xml node03:$PWD

sbin/stop-dfs.sh
sbin/stop-yarn.sh

sbin/start-dfs.sh
sbin/start-yarn.sh
~~~

####第四步:启动hiveserver2服务

`前台启动`

~~~shell
cd /export/servers/apache-hive-2.1.1-bin/
bin/hive --service hiveserver2
~~~

`后台启动`

~~~shell
nohup bin/hive --service hiveserver2  > /dev/null 2>&1 &
~~~

####第五步:使用beeline连接hiveserver2

~~~shell
bin/beeline
beeline> !connect jdbc:hive2://node03:10000
~~~

输入用户名和密码,用户名必须为root,密码任意.

![1561540565515](assets/1561540565515.png)	