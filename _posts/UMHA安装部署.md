---
layout: post
title: Openstack计算节点高可用
---

UMHA(UMCloud High Availability)--计算节点高可用处理程序，其目的是在某计算节点发生故障时，能够
自动将在该计算节点上的虚机疏散到其他可用计算节点，进而保证生产环境业务的正常运行。另外，真正运行在
环境下的程序名称则是umha-processing。

## 1. 环境准备
umha是基于ceilometer的扩展来实现的，目前阶段只增加storage_connectivity(存储网连通性，后续
可根据实际需求再拓展其他meter项)一个meter项，所以我们首先需要安装、配置及部署ceilomter。

### 安装
安装分两块：计算节点和控制节点安装。
#### 计算节点
计算节点安装的ceilometer程序用来采集数据（这里指的是上面提到的存储网的连通性），然后通过publisher把数据上报，注意此次ceilometer扩展采用的publisher是udp。
##### 安装
需要安装的包：`python-ceilometer,python-ceilometerclient,ceilometer-common,ceilometer-polling`,如果是直接通过dpkg安装，因为包间的依赖关系注意它们的安装顺序，当然也可以制作apt源通过apt-get安装。安装特别是首次安装时，往往会有ceilometer配置文件的交互界面的出现，直接跳过或者一路按enter即可，因为总归安装完之后我们仍要编辑ceilometer配置文件，关于这一点下边会讲到。
##### 配置
涉及到两个文件：`/etc/ceilometer/ceilometer.conf和/etc/ceilometer/pipeline_ha.yaml`，下面给出这两个配置文件的sample。

`/etc/ceilometer/ceilometer.conf`:
```sh
[DEAFULT]
# Polling namespace(s) to be used while resource polling (list value)
# Allowed values: compute, central, ipmi
polling_namespaces = compute

# List of metadata keys reserved for metering use. And these keys are additional to the ones included in the namespace. (list value)
reserved_metadata_keys = cluster

# Configuration file for pipeline definition. (string value)
pipeline_cfg_file = pipeline_ha.yaml

# If set to true, the logging level will be set to DEBUG instead of the default INFO level. (boolean value)
debug = false

# If set to false, the logging level will be set to WARNING instead of the default INFO level. (boolean value)
# This option is deprecated for removal.
# Its value may be silently ignored in the future.
verbose = true

# Log output to standard error. This option is ignored if log_config_append is set. (boolean value)
use_stderr = false

# The messaging driver to use, defaults to rabbit. Other drivers include amqp and zmq. (string value)
rpc_backend = rabbit

[compute]
# Some storage address used to check storage-connectivity. (string value)
storage_server_address = 192.168.0.10, 192.168.0.20

# Count of ping storage gateway. (integer value)
ping_count = 4

# Ping loss ratio that can be accepted. (floating point value)
ping_loss_ratio = 0.5

[keystone_authtoken]
# Complete public Identity API endpoint. (string value)
auth_uri = http://192.168.0.2:5000/v2.0

[oslo_messaging_rabbit]
# The RabbitMQ broker address where a single node is used. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_host
rabbit_host = 192.168.0.6

# The RabbitMQ broker port where a single node is used. (port value)
# Minimum value: 0
# Maximum value: 65535
# Deprecated group/name - [DEFAULT]/rabbit_port
rabbit_port = 5673

# Connect over SSL for RabbitMQ. (boolean value)
# Deprecated group/name - [DEFAULT]/rabbit_use_ssl
rabbit_use_ssl = false

# The RabbitMQ userid. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_userid
rabbit_userid = nova

# The RabbitMQ password. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_password
rabbit_password = 0edA4bYmVlJHDuzkI93mYBs1

# The RabbitMQ virtual host. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_virtual_host
rabbit_virtual_host = /

# Try to use HA queues in RabbitMQ (x-ha-policy: all). If you change this option, you must wipe the RabbitMQ database. In RabbitMQ 3.0,
# queue mirroring is no longer controlled by the x-ha-policy argument when declaring a queue. If you just want to make sure that all queues
# (except  those with auto-generated names) are mirrored across all nodes, run: "rabbitmqctl set_policy HA '^(?!amq\.).*' '{"ha-mode":
# "all"}' " (boolean value)
# Deprecated group/name - [DEFAULT]/rabbit_ha_queues
rabbit_ha_queues = True

# Number of seconds after which the Rabbit broker is considered down if heartbeat's keep-alive fails (0 disable the heartbeat). EXPERIMENTAL
# (integer value)
heartbeat_timeout_threshold = 60

[publisher]
# Secret value for signing messages. Set value empty if signing is not required to avoid computational overhead. (string value)
# Deprecated group/name - [DEFAULT]/metering_secret
# Deprecated group/name - [publisher_rpc]/metering_secret
# Deprecated group/name - [publisher]/metering_secret
telemetry_secret = nmWpu7Pu8DDWhQ3oPaBpVQjn

[service_credentials]
# Region name to use for OpenStack service endpoints. (string value)
# Deprecated group/name - [DEFAULT]/os_region_name
region_name = RegionOne

# Type of endpoint in Identity service catalog to use for communication with OpenStack services. (string value)
# Allowed values: public, internal, admin, auth, publicURL, internalURL, adminURL
# Deprecated group/name - [DEFAULT]/os_endpoint_type
interface = internalURL

# Authentication URL (unknown value)
auth_url = http://192.168.0.2:5000/v2.0/

# Project name to scope to (unknown value)
# Deprecated group/name - [DEFAULT]/tenant-name
project_name = services

# Username (unknown value)
# Deprecated group/name - [DEFAULT]/user-name
username = ceilometer

# User's password (unknown value)
password = a1b2c3d4e5
```
`/etc/ceilometer/pipeline_ha.yaml`:
```sh
---
sources:
    - name: ha_source
      interval: 30
      meters:
            - "storage_connectivity"
      sinks:
          - ha_sink
sinks:
    - name: ha_sink
      transformers:
      publishers:
          - udp://192.168.0.6:4953  #存储网ip及端口
#         - udp://192.168.1.2:4953  #管理网ip及端口
```
##### 启动/重启服务
`service ceilometer-polling start/restart`

#### 控制节点
控制节点安装的ceilometer程序用来收集计算节点上报的数据，然后入库保存，这里涉及数据库核心表有三个：`sample, meter, resource`。

##### 安装
需要安装的包：`python-ceilometer, python-ceilometerclient, ceilometer-common, ceilometer-polling, ceilometer-collector, ceilometer-api, ceilometer-agent-notification`，至于安装过程上文已经提到，在此不做赘述。

##### 配置
涉及到一个配置文件：`/etc/ceilometer/ceilometer.conf`，下面给出这个配置文件的sample。

`/etc/ceilometer/ceilomter.conf`：
```sh
[DEFAULT]
# Polling namespace(s) to be used while resource polling (list value)
# Allowed values: compute, central, ipmi
polling_namespaces = central

# Dispatchers to process metering data. (multi valued)
# Deprecated group/name - [DEFAULT]/dispatcher
meter_dispatchers = database

# If set to true, the logging level will be set to DEBUG instead of the default INFO level. (boolean value)
debug = false

# If set to false, the logging level will be set to WARNING instead of the default INFO level. (boolean value)
# This option is deprecated for removal.
# Its value may be silently ignored in the future.
verbose = true

[collector]
# Address to which the UDP socket is bound. Set to an empty string to disable. (string value)
udp_address = 0.0.0.0

# Port to which the UDP socket is bound. (port value)
# Minimum value: 0
# Maximum value: 65535
udp_port = 4953

[database]
# Number of seconds that samples are kept in the database for (<= 0 means forever). (integer value)
# Deprecated group/name - [database]/time_to_live
metering_time_to_live = 1800

# The SQLAlchemy connection string to use to connect to the database. (string value)
# Deprecated group/name - [DEFAULT]/sql_connection
# Deprecated group/name - [DATABASE]/sql_connection
# Deprecated group/name - [sql]/connection
connection = mysql://ceilometer:ceilometer123@192.168.0.2/ceilometer?charset=utf8

[oslo_messaging_rabbit]
# The RabbitMQ broker address where a single node is used. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_host
rabbit_host = 192.168.0.6

# The RabbitMQ broker port where a single node is used. (port value)
# Minimum value: 0
# Maximum value: 65535
# Deprecated group/name - [DEFAULT]/rabbit_port
rabbit_port = 5673

# Connect over SSL for RabbitMQ. (boolean value)
# Deprecated group/name - [DEFAULT]/rabbit_use_ssl
rabbit_use_ssl = false

# The RabbitMQ userid. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_userid
rabbit_userid = nova

# The RabbitMQ password. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_password
rabbit_password = 0edA4bYmVlJHDuzkI93mYBs1

# The RabbitMQ virtual host. (string value)
# Deprecated group/name - [DEFAULT]/rabbit_virtual_host
rabbit_virtual_host = /

# Try to use HA queues in RabbitMQ (x-ha-policy: all). If you change this option, you must wipe the RabbitMQ database. In RabbitMQ 3.0,
# queue mirroring is no longer controlled by the x-ha-policy argument when declaring a queue. If you just want to make sure that all queues
# (except  those with auto-generated names) are mirrored across all nodes, run: "rabbitmqctl set_policy HA '^(?!amq\.).*' '{"ha-mode":
# "all"}' " (boolean value)
# Deprecated group/name - [DEFAULT]/rabbit_ha_queues
rabbit_ha_queues = True

# Number of seconds after which the Rabbit broker is considered down if heartbeat's keep-alive fails (0 disable the heartbeat). EXPERIMENTAL
# (integer value)
heartbeat_timeout_threshold = 60

[publisher]
# Secret value for signing messages. Set value empty if signing is not required to avoid computational overhead. (string value)
# Deprecated group/name - [DEFAULT]/metering_secret
# Deprecated group/name - [publisher_rpc]/metering_secret
# Deprecated group/name - [publisher]/metering_secret
telemetry_secret = nmWpu7Pu8DDWhQ3oPaBpVQjn
```
##### 创建ceilometer数据库
首先，进入到mysql交互命令行，依次执行如下命令即可：
```sh
create database ceilometer CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON ceilometer.* TO 'ceilometer'@'localhost' IDENTIFIED BY 'ceilometer123';
GRANT ALL PRIVILEGES ON ceilometer.* TO 'ceilometer'@'%' IDENTIFIED BY 'ceilometer123';
```
注意：上面命令中`ceilometer123`是访问`ceilometer`数据库的密码，要跟`/etc/ceilometer/ceilometer.conf`中`connection`的配置保持一致。

其次，对`ceilometer`数据库进行初始化：
```sh
ceilometer-dbsync --config-file /etc/ceilometer/ceilometer.conf
```

最后，收集到的数据是用来做服务判断的，所以没必要做长时间保留，可以通过`metering_time_to_live`配置项设置sample数据保留时间，然后通过crontab定期清理过期数据，比如2个小时清理一次： 
```sh
0 */2 * * * ceilometer-expirer --config-file /etc/ceilometer/ceilometer.conf
```

##### keystone相关
首先，创建`ceilometer`用户：
```sh
openstack user create --domain default --password-prompt ceilometer
```
注意：根据提示设置`ceilometer`用户密码，这个要跟配置保持一致。

其次，添加`admin`角色到`ceilometer`用户上:
```sh
openstack role add --project service --user ceilometer admin
```

然后，创建`ceilometer`服务实体：
```sh
openstack service create --name ceilometer --description "Telemetry" metering
```

最后，创建endpoint：
```sh
# 命令中ip要根据实际情况更改
openstack endpoint create --region RegionOne metering public http://172.31.11.204:8777
openstack endpoint create --region RegionOne metering admin http://192.168.0.2:8777
openstack endpoint create --region RegionOne metering internal http://192.168.0.2:8777
```
##### 启动/重启服务
```sh
service ceilometer-api start/restart
service ceilometer-agent-notification start/restart
service ceilometer-polling start/restart
service ceilometer-collector start/restart
```

至此，ceilometer的安装及部署已经完成，下面开始umha的安装及部署。

## 2. umha安装
这部分程序是安装在控制节点上的，安装umha之前，我们有必要重新明确umha的功能：evacuate和邮件告警：
* evacuate：是umha的核心功能，在某计算节点被检测到故障时，自动将其上的虚机疏散到其他可用节点；
* 邮件告警：这个功能是从用户体验的角度考虑的，可以让用户及时地获知节点或者是虚机的状态。

### 安装
安装包的名称是`umha_xxx_amd64.deb`，直接用dpkg命令安装即可，至于安装过程中的配置问题，跟上面处理类似。

## 3. umha配置
umha相关的配置文件都会统一放在`/etc/umha/`目录下，具体文件可参见如下：

文件路径|
--- |
/etc/umha/umha.conf | 
/etc/umha/ipmi_info |
/etc/umha/node_storage_ip_info |

### umha.conf
这个是umha的核心配置文件，安装完unha之后我们一般都要重新配置，它会涉及到几个section：DEFAULT，认证、数据库、邮件、时间和阈值，具体可参照下面的sample：

```bash
[DEFAULT]

#
# From umha.conf
#

# ipmi have configured (boolean value)
has_IPMI = true

# filename store ipmi info (string value)
ipmi_info = /etc/umha/ipmi_info

# number of ping to check connectivity (integer value)
ping_count = 3

# if allow evacuate local vm (boolean value)
allow_evacuate_local_vm = false

# (boolean value)
on_share_storage = true

# (integer value)
ipmi_check_max_count = 3

# (integer value)
ipmi_check_interval = 10

# (boolean value)
force_evacuate_when_ipmi_unknown = false

# (dict value)
business_connectivity_cfg = enabled:0,severity:0

# (dict value)
path_access_ok_cfg = enabled:0,severity:0

# filename store node storage ip info (string value)
node_storage_ip_info = /etc/umha/node_storage_ip_info

# (boolean value)
really_evacuate = true

# (integer value)
interval_check_nova_service = 10

# (integer value)
max_second_wait_for_service_down = 60

# (integer value)
seconds_before_check_vm_status = 20

[auth]

#
# From umha.conf
#

# (string value)
auth_url = http://192.168.0.2:35357/v2.0

# (string value)
username = admin

# (string value)
password = admin

# (string value)
tenant_name = admin

# (string value)
nova_version = 2

# (string value)
cinder_version = 2

# (string value)
region_name = RegionOne


[db]

#
# From umha.conf
#

# db connection (string value)
connection = mysql://ceilometer:ceilometer123@192.168.0.2/ceilometer?charset=utf8


[email]

#
# From umha.conf
#

# (boolean value)
enable_send_mail = true

# (string value)
send_server = smtp.126.com

# (integer value)
send_port = 25

# (boolean value)
need_auth = true

# (string value)
sender_username = zynf183913831@126.com

# (string value)
sender_pwd = Fch19880810

# (list value)
receiver_list = fuchunhui@umcloud.com

# (string value)
mail_subject = UMHA高可用告警

# (integer value)
send_interval_hours = 3


[time]

#
# From umha.conf
#

# (integer value)
sample_interval = 60

# interval of running check (integer value)
ha_check_interval = 180

# timeout of evacuate (integer value)
evacuate_timeout = 60

[valve]

#
# From umha.conf
#

# (integer value)
storage_number = 6

# (floating point value)
storage_ratio = 0.6

# (integer value)
power_off_number = 6

# (floating point value)
power_off_ratio = 0.6

# (integer value)
panic_number = 6

# (floating point value)
panic_ratio = 0.6

# (integer value)
error_count = 3

# (integer value)
continue_count = 5
```
### 数据库同步
umha新增了几个表：

Table Name|
---|
node_heartbeat|
ipmi_info|
umha_log|
evacuate_history|

首次安装umha程序或者存在表更新时，在安装完umha之后要手动做数据库同步：执行`umha_db_sync`。

### 导入存储网信息
把计算节点存储网的信息记录到`/etc/umha/node_storage_ip_info`，文件内容格式：`hostname:storage_ip`

比如：

`node-1.domain.tld:192.168.0.6`

`node-3.domain.tld:192.168.0.10`

编辑完毕后，然后执行`load_node_storage_ip`

### 导入IPMI信息
把计算节点IPMI信息记录到`/etc/umha/ipmi_info`，文件内容格式：`hostname:ip:username:password`

比如：

`node-1.domain.tld:172.31.11.56:admin:admin`

`node-3.domain.tld:172.31.11.168:root:root123`

编辑完毕后，执行命令`load_ipmi_info`

### 启动/重启服务
```bash
service umha-processing start/restart
```

## 4. umha部署到生产环境
以上ceilometer和umha的安装和部署一般是针对开发或测试环境（通常一个计算节点和一个控制节点即可），如果是部署到生产环境，必须要避免单节点发生故障时服务不可用的情况。鉴于生产环境的这种需求，可以采用corosync和pacemaker来对节点集群进行管理，且一般使用主备模式。

### 部署准备
一般情况下，openstack集群下的控制节点都已安装corosync、pacemaker和crmsh，并且也已经把节点纳管到pacemaker集群管理中，所以我们只需要做资源的添加及配置。

`注：以下所有操作均在控制节点上完成。`

### 属性设置
```bash
crm configure property no-quorum-policy=ignore stonith-enabled=false

```

### 添加资源
需要将ceilometer-collector和umha-processing加入到pacemaker集群资源管理中。

***`另外，对于新添加的这两个资源对应的服务要检查各自的upstart文件是不是配置了禁用开机启动，如果没有，需要配置禁用服务开机自启动。`***

#### 新增ceilometer-collector-ha资源
```bash
crm configure primitive ceilometer-collector-ha upstart:ceilometer-collector-ha op monitor interval=30s timeout=60s
```

#### 新增umha-processing-ha资源
```bash
crm configure primitive umha-processing-ha upstart:umha-processing-ha op monitor interval=30s timeout=60s
```

#### 资源设置节点倾向
随意选择一个控制节点，设置资源对每个节点倾向，节点域名可通过执行`uname -n`获取
```
# ceilometer-collector-ha
crm configure location ceilometer-collector-ha-on-node-7.domain.tld ceilometer-collector-ha 100: node-7.domain.tld
crm configure location ceilometer-collector-ha-on-node-8.domain.tld ceilometer-collector-ha 100: node-8.domain.tld
crm configure location ceilometer-collector-ha-on-node-6.domain.tld ceilometer-collector-ha 100: node-6.domain.tld

# umha-processing-ha
crm configure location umha-processing-ha-on-node-7.domain.tld umha-processing-ha 100: node-7.domain.tld
crm configure location umha-processing-ha-on-node-8.domain.tld umha-processing-ha 100: node-8.domain.tld
crm configure location umha-processing-ha-on-node-8.domain.tld umha-processing-ha 100: node-8.domain.tld
```

#### 设置协同约束
```bash
# 强约束ceilometer-collector和umha-processing运行同一个节点
crm configure colocation ceilometer-collector-ha_with_umha-processing-ha inf: ceilometer-collector-ha umha-processing-ha
```
#### 防火墙问题
如上所述，ceilometer采集和收集数据是通过udp，这里设置了新端口4953，所以控制节点iptables要开放udp 4953端口，否则无法收集数据，具体操作如下：
```bash
# 在/etc/iptables/rules.v4的chain input末尾追加：
-A INPUT -p udp -s 192.168.0.0/24 --dport 4953 -m comment --comment "Allow mgnt traffic to umha" -j ACCEPT
# 保存生效
iptables-restore < /etc/iptables/rules.v4
```
<完>
