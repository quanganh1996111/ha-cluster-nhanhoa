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
rabbitmqctl add_user admin nhcluster
rabbitmqctl set_user_tags admin administrator
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