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