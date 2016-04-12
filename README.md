# Ganglia搭建与配置 by 李扬

Ganglia就是集群版的资源监视器，下面就以监视一个搭建在Ubuntu上的Spark集群为例，看下Ganglia是如何搭建与配置的。

Ganglia分三个模块:
> * `ganglia-monitor` 即gmond,用来监视结点的资源
> * `gmetad` Ganglia的WebUI后端，轮询接收gmond收集到的信息
> * `ganglia-webfrontend` Ganglia的WebUI前端

### 主节点Master配置
在主节点上安装gmetad, ganglia-webfrontend, ganglia-monitor, 在其它结点上，只需安装ganglia-monitor即可。

```bash
sudo apt-get install gmetad ganglia-webfrontend ganglia-monitor
```

将Ganglia的配置复制到Apache的目录下:
```bash
sudo cp /etc/ganglia-webfrontend/apache.conf /etc/apache2/sites-enabled/ganglia.conf
```

配置`gmetad.conf`

```bash
sudo vim /etc/ganglia/gmetad.conf
```

修改为以下内容:

```bash
# data_source后面是集群名"my cluster"和需要监视的结点，这里是master, slave1, slave2, 如果未指定端口号，则默认为8649
data_source "my cluster" 10 master:8649 slave1:8649 slave2:8649 
# case_sensitive_hostnames为1, 则大小写敏感，防止将hostname中大写变成小写
case_sensitive_hostnames 1 
```

配置`gmond.conf`
在每个结点都需要配置`gmond.conf`, 配置相同如下所示:

```bash
sudo vim /etc/ganglia/gmond.conf
```

修改为:

```bash
globals {                    
  daemonize = yes  
  setuid = yes             
  user = spark     #运行Ganglia的用户              
  debug_level = 0               
  max_udp_msg_len = 1472        
  mute = no             
  deaf = no             
  host_dmax = 0 /*secs */ 
  cleanup_threshold = 300 /*secs */ 
  gexec = no             
  send_metadata_interval = 5     #发送数据的时间间隔
} 

/* If a cluster attribute is specified, then all gmond hosts are wrapped inside 
 * of a <CLUSTER> tag.  If you do not specify a cluster tag, then all <HOSTS> will 
 * NOT be wrapped inside of a <CLUSTER> tag. */ 
cluster { 
  name = "my cluster"         #集群名称
  owner = "spark"               #运行Ganglia的用户
  latlong = "unspecified" 
  url = "unspecified" 
} 

/* The host section describes attributes of the host, like the location */ 
host { 
  location = "unspecified" 
} 

/* Feel free to specify as many udp_send_channels as you like.  Gmond 
   used to only support having a single channel */ 
udp_send_channel { 
  #mcast_join = 239.2.11.71     #注释掉组播
  host = master                 #发送给安装gmetad的结点
  port = 8649                   #监听端口
  ttl = 1 
} 

/* You can specify as many udp_recv_channels as you like as well. */ 
udp_recv_channel { 
  #mcast_join = 239.2.11.71     #注释掉组播
  port = 8649 
  #bind = 239.2.11.71 
} 

/* You can specify as many tcp_accept_channels as you like to share 
   an xml description of the state of the cluster */ 
tcp_accept_channel { 
  port = 8649 
} 
```

启动Ganglia
```bash
sudo service ganglia-monitor start
sudo service gmetad start
```
然后打开WebUI查看结果:
```bash
http://master/ganglia
```

### 子节点Slave配置
子节点只需安装ganglia-monitor

```bash
sudo apt-get install ganglia-monitor
```
配置`gmond.conf`
配置文件与主节点相同:

```bash
sudo vim /etc/ganglia/gmond.conf
```

修改为:

```bash
globals {                    
  daemonize = yes  
  setuid = yes             
  user = spark     #运行Ganglia的用户              
  debug_level = 0               
  max_udp_msg_len = 1472        
  mute = no             
  deaf = no             
  host_dmax = 0 /*secs */ 
  cleanup_threshold = 300 /*secs */ 
  gexec = no             
  send_metadata_interval = 5     #发送数据的时间间隔
} 

/* If a cluster attribute is specified, then all gmond hosts are wrapped inside 
 * of a <CLUSTER> tag.  If you do not specify a cluster tag, then all <HOSTS> will 
 * NOT be wrapped inside of a <CLUSTER> tag. */ 
cluster { 
  name = "my cluster"         #集群名称
  owner = "spark"               #运行Ganglia的用户
  latlong = "unspecified" 
  url = "unspecified" 
} 

/* The host section describes attributes of the host, like the location */ 
host { 
  location = "unspecified" 
} 

/* Feel free to specify as many udp_send_channels as you like.  Gmond 
   used to only support having a single channel */ 
udp_send_channel { 
  #mcast_join = 239.2.11.71     #注释掉组播
  host = master                 #发送给安装gmetad的结点
  port = 8649                   #监听端口
  ttl = 1 
} 

/* You can specify as many udp_recv_channels as you like as well. */ 
udp_recv_channel { 
  #mcast_join = 239.2.11.71     #注释掉组播
  port = 8649 
  #bind = 239.2.11.71 
} 

/* You can specify as many tcp_accept_channels as you like to share 
   an xml description of the state of the cluster */ 
tcp_accept_channel { 
  port = 8649 
} 
```

启动Ganglia
```bash
sudo service ganglia-monitor start
```

### 附录: 监控Hadoop集群配置
`Hadoop配置`
在所有hadoop所在的节点，均需要配置hadoop-metrics2.properties，配置如下:
```bash
#   Licensed to the Apache Software Foundation (ASF) under one or more
#   contributor license agreements.  See the NOTICE file distributed with
#   this work for additional information regarding copyright ownership.
#   The ASF licenses this file to You under the Apache License, Version 2.0
#   (the "License"); you may not use this file except in compliance with
#   the License.  You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
#

# syntax: [prefix].[source|sink].[instance].[options]
# See javadoc of package-info.java for org.apache.hadoop.metrics2 for details

#注释掉以前原有配置

#*.sink.file.class=org.apache.hadoop.metrics2.sink.FileSink
# default sampling period, in seconds
#*.period=10

# The namenode-metrics.out will contain metrics from all context
#namenode.sink.file.filename=namenode-metrics.out
# Specifying a special sampling period for namenode:
#namenode.sink.*.period=8

#datanode.sink.file.filename=datanode-metrics.out

# the following example split metrics of different
# context to different sinks (in this case files)
#jobtracker.sink.file_jvm.context=jvm
#jobtracker.sink.file_jvm.filename=jobtracker-jvm-metrics.out
#jobtracker.sink.file_mapred.context=mapred
#jobtracker.sink.file_mapred.filename=jobtracker-mapred-metrics.out

#tasktracker.sink.file.filename=tasktracker-metrics.out

#maptask.sink.file.filename=maptask-metrics.out

#reducetask.sink.file.filename=reducetask-metrics.out

*.sink.ganglia.class=org.apache.hadoop.metrics2.sink.ganglia.GangliaSink31  
*.sink.ganglia.period=10

*.sink.ganglia.slope=jvm.metrics.gcCount=zero,jvm.metrics.memHeapUsedM=both  
*.sink.ganglia.dmax=jvm.metrics.threadsBlocked=70,jvm.metrics.memHeapUsedM=40  

namenode.sink.ganglia.servers=master:8649  
resourcemanager.sink.ganglia.servers=master:8649  

datanode.sink.ganglia.servers=master:8649    
nodemanager.sink.ganglia.servers=master:8649    


maptask.sink.ganglia.servers=master:8649    
reducetask.sink.ganglia.servers=master:8649
```

`Hbase配置`
在所有的hbase节点中均配置hadoop-metrics2-hbase.properties,配置如下:
```bash
# syntax: [prefix].[source|sink].[instance].[options]
# See javadoc of package-info.java for org.apache.hadoop.metrics2 for details

#*.sink.file*.class=org.apache.hadoop.metrics2.sink.FileSink
# default sampling period
#*.period=10

# Below are some examples of sinks that could be used
# to monitor different hbase daemons.

# hbase.sink.file-all.class=org.apache.hadoop.metrics2.sink.FileSink
# hbase.sink.file-all.filename=all.metrics

# hbase.sink.file0.class=org.apache.hadoop.metrics2.sink.FileSink
# hbase.sink.file0.context=hmaster
# hbase.sink.file0.filename=master.metrics

# hbase.sink.file1.class=org.apache.hadoop.metrics2.sink.FileSink
# hbase.sink.file1.context=thrift-one
# hbase.sink.file1.filename=thrift-one.metrics

# hbase.sink.file2.class=org.apache.hadoop.metrics2.sink.FileSink
# hbase.sink.file2.context=thrift-two
# hbase.sink.file2.filename=thrift-one.metrics

# hbase.sink.file3.class=org.apache.hadoop.metrics2.sink.FileSink
# hbase.sink.file3.context=rest
# hbase.sink.file3.filename=rest.metrics


*.sink.ganglia.class=org.apache.hadoop.metrics2.sink.ganglia.GangliaSink31  
*.sink.ganglia.period=10  

hbase.sink.ganglia.period=10  
hbase.sink.ganglia.servers=master:8649
```
启动hadoop、hbase集群:
```bash
start-dfs.sh
start-yarn.sh
start-hbase.sh
```
