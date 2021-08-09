# Cài đặt rabitmq 3 node CentOS7



### Thực hiện trên tất cả các Node

- Cài đặt Rabbitmq-server:

```
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/rabbitmq_v3_6_10/rabbitmq-server-3.6.10-1.el7.noarch.rpm
rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
rpm -Uvh rabbitmq-server-3.6.10-1.el7.noarch.rpm
```

- Khởi động dịch vụ:

```
systemctl start rabbitmq-server
systemctl enable rabbitmq-server
systemctl status rabbitmq-server
```

### Thực hiện trên Node1

- Kiểm tra trạng thái node:

```
sudo rabbitmqctl status|grep rabbit
```

- Tạo User cho App (nhcluster), phân quyền:

```
rabbitmqctl add_user admin nhcluster //Tạo user admin/nhcluster
rabbitmqctl set_user_tags admin administrator //Phân quyền administrator
rabbitmqctl add_vhost admin_vhost
rabbitmqctl set_permissions -p admin_vhost admin ".*" ".*" ".*"
```

- Copy file `/var/lib/rabbitmq/.erlang.cookie` từ `node1` sang các node còn lại. (Có nhập password)

```
scp /var/lib/rabbitmq/.erlang.cookie root@node2:/var/lib/rabbitmq/.erlang.cookie

scp /var/lib/rabbitmq/.erlang.cookie root@node3:/var/lib/rabbitmq/.erlang.cookie
```

- Cấu hình policy HA Rabbit Cluster:

```
rabbitmqctl -p admin_vhost set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'
```

- Kiểm tra trạng thái cluster:

```
rabbitmqctl cluster_status
```

```
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1]}]},
 {running_nodes,[rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node1,[]}]}]
```

- Khởi chạy app:

```
rabbitmqctl start_app
```

```
[root@node1 ~]# rabbitmqctl start_app
Starting node rabbit@node1
```

### Thực hiện trên Node2 và Node3

#### Trên Node2

- Phân quyền file `/var/lib/rabbitmq/.erlang.cookie`

```
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```

- Khởi động lại Dịch vụ:

```
systemctl restart rabbitmq-server.service
```

- Join cluster `node1`

```
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

<img src="https://imgur.com/N360HoC.png">

#### Trên Node3

- Phân quyền file `/var/lib/rabbitmq/.erlang.cookie`

```
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```

- Khởi động lại Dịch vụ:

```
systemctl restart rabbitmq-server.service
```

- Join cluster `node1`

```
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

<img src="https://imgur.com/X6qkdl4.png">

#### Kiểm tra trên tất cả các node

```
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node3,rabbit@node2,rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node3,[]},{rabbit@node2,[]},{rabbit@node1,[]}]}]
```

```
[root@node2 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node2
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node3,rabbit@node1,rabbit@node2]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node3,[]},{rabbit@node1,[]},{rabbit@node2,[]}]}]
```

```
[root@node3 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node3
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node1,rabbit@node2,rabbit@node3]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node1,[]},{rabbit@node2,[]},{rabbit@node3,[]}]}]
```

#### Kích hoạt plugin rabbit management

- Thực hiện trên tất cả các Node

```
rabbitmq-plugins enable rabbitmq_management
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq
```

- Truy cập giao diện web quản lý rabbitmq

```
http://IP:15672
```

Tài khoản: `admin / nhcluster`

<img src="https://imgur.com/G86U2rW.png">