# High Availability - Phần 5: Hướng dẫn triển khai Haproxy Pacemaker cho Cluster Galera 3 node trên CentOS 7

## Tổng quan

**HAProxy** viết tắt của **High Availability Proxy**, là công cụ mã nguồn mở nổi tiếng ứng dụng cho giải pháp cân bằng tải TCP/HTTP cũng như giải pháp máy chủ Proxy (Proxy Server). HAProxy có thể chạy trên các mỗi trường Linux, Solaris, FreeBSD. Công dụng phổ biến nhất của HAProxy là cải thiện hiệu năng, tăng độ tin cậy của hệ thống máy chủ bằng cách phân phối khối lượng công việc trên nhiều máy chủ (như Web, App, cơ sở dữ liệu). HAProxy hiện đã và đang được sử dụng bởi nhiều website lớn như GoDaddy, GitHub, Bitbucket, Stack Overflow, Reddit, Speedtest.net, Twitter và trong nhiều sản phẩm cung cấp bởi Amazon Web Service.

**MariaDB Galera Cluster** là giải pháp sao chép đồng bộ nâng cao tính sẵn sàng cho MariaDB. Galera hỗ trợ chế độ Active-Active tức có thể truy cập, ghi dữ liệu đồng thời trên tất các node MariaDB thuộc Galera Cluster.

**Pacemaker** là trình quản lý tài nguyên trong cluster được phát triển bởi ClusterLabs. Pacemaker tương thích với rất nhiều dịch vụ phổ biến hiện có và hoàn toàn có thể tự phát triển module để quản lý các tài nguyên mà pacemaker chưa hỗ trợ.

## Phần 1. Chuẩn bị

### 1. Mô hình hoạt động

<img src="https://imgur.com/Bm3tkWG.png">

### 2. Quy hoạch IP 

<img src="https://imgur.com/pb75k0A.png">

### 3. Chuẩn bị

Thực hiện cài đặt chuẩn bị môi trường, galera mariadb theo tài liệu [tại đây](https://github.com/quanganh1996111/ha-cluster-nhanhoa/blob/main/2-cluster/5-install-galare-3node-centos7.md)

### 4. Cài đặt Galera database 3 node CentOS7 Wordpress

#### 4.1. Cài đặt Haproxy bản 1.8

**Thực hiện trên tất cả các node**

- Cài đặt một số phần mềm cần thiết:

```
sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y
```

- Cấu hình file config cho haproxy:

```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

```
echo 'global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats
    bind :8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics

listen galera
    bind 172.16.2.72:3306
    balance source
    mode tcp
    option tcpka
    option tcplog
    option clitcpka
    option srvtcpka
    timeout client 28801s
    timeout server 28801s
    option mysql-check user haproxy
    server node1 172.16.2.56:3306 check inter 5s fastinter 2s rise 3 fall 3
    server node2 172.16.2.59:3306 check inter 5s fastinter 2s rise 3 fall 3 backup
    server node3 172.16.2.62:3306 check inter 5s fastinter 2s rise 3 fall 3 backup' > /etc/haproxy/haproxy.cfg
```

- Cấu hình Log cho HAProxy

```
sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf

echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf

systemctl restart rsyslog
```

- Bổ sung cấu hình cho phép kernel có thể binding tới IP VIP

```
echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf

sysctl -p
```

- Tắt dịch vụ HAProxy

```
systemctl stop haproxy
systemctl disable haproxy
```

- Trên `node1` tạo user `haproxy`, phục vụ plugin health check của HAProxy (option mysql-check user haproxy)

```
mysql -uroot -p
```

```
CREATE USER 'haproxy'@'node1';
CREATE USER 'haproxy'@'node2';
CREATE USER 'haproxy'@'node3';
CREATE USER 'haproxy'@'%';
```

<img src="https://imgur.com/xRoWhXa.png">

## Phần 3. Triển khai Cluster Pacemaker

### Bước 1: Cài đặt pacemaker corosync

**Thực hiện trên tất cả các node**

- Cài đặt gói `pacemaker pcs`:

```
yum -y install pacemaker pcs

systemctl start pcsd 
systemctl enable pcsd
```

- Thiết lập mật khẩu cho user `hacluster`:

```
passwd hacluster
```

**Lưu ý**: Nhập chính xác và nhớ mật khẩu user hacluster, đồng bộ mật khẩu trên tất cả các node.

### Bước 2: Tạo Cluster

- Chứng thực cluster (Chỉ thực thiện trên cấu hình trên một node duy nhất, trong bài sẽ thực hiện trên `node1`), nhập chính xác tài khoản user `hacluster`.

```
pcs cluster auth node1 node2 node3

Username: hacluster
Password: *********
```

- Kết quả:

<img src="https://imgur.com/4pyozUX.png">

- Khởi tạo cấu hình cluster ban đầu:

```
pcs cluster setup --name ha_cluster node1 node2 node3
```

<img src="https://imgur.com/SUILqBu.png">

- Trong đó:

    - `ha_cluster`: Tên của cluster khởi tạo

    - `node1, node2, node3`: Hostname các node thuộc cluster, yêu cầu khai báo trong /etc/host.

- Khởi động Cluster:

```
pcs cluster start --all
```

- Cho phép cluster khởi động cùng OS:

```
pcs cluster enable --all
```

<img src="https://imgur.com/J9QCk2K.png">

### Bước 3: Thiết lập Cluster  (Thực hiện trên 1 node ở bài lab là node 1)

- Bỏ qua cơ chế STONITH:

```
pcs property set stonith-enabled=false
```

- Cho phép Cluster chạy kể cả khi mất quorum:

```
pcs property set no-quorum-policy=ignore
```

- Hạn chế Resource trong cluster chuyển node sau khi Cluster khởi động lại:

```
pcs property set default-resource-stickiness="INFINITY"
```

- Kiểm tra thiết lập cluster:

```
pcs property list
```

<img src="https://imgur.com/mymdOq1.png">

- Tạo Resource IP VIP Cluster:

```
pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=172.16.2.72 cidr_netmask=20 op monitor interval=30s
```

- Tạo Resource quản trị dịch vụ HAProxy:

```
pcs resource create Loadbalancer_HaProxy systemd:haproxy op monitor timeout="5s" interval="5s"
```

- Ràng buộc thứ tự khởi động dịch vụ, khởi động dịch vụ Virtual_IP sau đó khởi động dịch vụ Loadbalancer_HaProxy:

```
pcs constraint order start Virtual_IP then Loadbalancer_HaProxy kind=Optional
```

- Ràng buộc resource Virtual_IP phải khởi động cùng node với resource Loadbalancer_HaProxy:

```
pcs constraint colocation add Virtual_IP Loadbalancer_HaProxy INFINITY
```

- Kiểm tra trạng thái Cluster:

```
pcs status
```

- Kiểm tra cấu hình Resource:

```
pcs resource show --full
```

- Kiểm tra ràng buộc trên resource:

```
pcs constraint
```

## Phần 4. Kiểm tra

### Kiểm tra trạng thái dịch vụ

- Truy cập IP VIP `http://172.16.2.72:8080/stats`:

<img src="https://imgur.com/qv7VcuI.png">

- Kết nối tới database MariaDB thông qua IP VIP:

```
mysql -h 172.16.2.72 -u haproxy -p
```

<img src="https://imgur.com/RmIwmUs.png">

- Tắt `node2` chứa `Virtual_IP`, `Loadbalancer_HaProxy`

Kiểm tra trạng thái Cluster, `node2` đã bị tắt. Dịch vụ `Virtual_IP` và `Loadbalancer_HaProxy` được chuyển sang `node1` tự động.

<img src="https://imgur.com/L6goZHh.png">

Tại thời điểm `node2` bị tắt, `Pacemaker Cluster` sẽ tự đánh giá, di chuyển các dịch vụ `Virtual_IP` và `Loadbalancer_HaProxy` sang node đang sẵn sàng trong Cluster, duy trì dịch vụ luôn hoạt động dù cho 1 node trong cluster gặp sự cố. Đồng thời, `Cluster Galera` sẽ vẫn hoạt động bình thường dù 1 node trong cluster xảy ra sự cố.

<img src="https://imgur.com/FJD6MjF.png">

- Tiến hành tắt thêm `node1`:

<img src="https://imgur.com/mgfoSvl.png">

<img src="https://imgur.com/buA7BkA.png">

- Thử kết nối tới Database MariaDB thông qua IP VIP:

<img src="https://imgur.com/ClZ7iT2.png">

- Bật lại `node1` và `node2`

Thời gian `uptime` của 2 Node hiển thị:

<img src="https://imgur.com/2cN5gbi.png">

## Phần 5. Triển khai Webserver Wordpress

### Bước 1: Tạo database sử dụng cho wordpress

Do sử dụng galera để quản lý cluster database nên chỉ cần đứng ở 1 node tạo database sẽ đồng bộ sang các node khác.

Đứng ở `node1`

- Đặt lại thông tin mysql:

```
mysql_secure_installation
```

- Tạo DB cho wordpress:

```
mysql -u root -p
```

```
create database wpdatabase;
create user 'wpuser'@'%' identified by 'Wordpress';
GRANT ALL PRIVILEGES ON wpdatabase.* TO 'wpuser'@'%' IDENTIFIED BY 'Wordpress';
flush privileges;
exit
```

### Bước 2: Cài đặt Services cho Webserver

**Triển khai trên cả 3 node**

- Cài đặt Apache:

```
yum install httpd -y
systemctl start httpd
systemctl enable httpd
```

- Cài đặt PHP:

```
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install yum-utils -y
yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm -y
yum-config-manager --enable remi-php72
```

```
yum install php72 php72-php-fpm php72-php-mysqlnd php72-php-opcache php72-php-xml php72-php-xmlrpc php72-php-gd php72-php-mbstring php72-php-json php php-mysql -y
```

- Kiểm tra version PHP:

```
php -v
echo "<?php phpinfo(); ?>" > /var/www/html/info.php
systemctl restart httpd
```

```
http://172.16.2.56/info.php
http://172.16.2.59/info.php
http://172.16.2.62/info.php
```

### Bước 3: Triển khai trên wordpress trên 3 node

- Cài đặt các phần mềm hỗ trợ:

```
yum -y install php-gd
yum install wget -y
```

- Download Wordpress:

```
wget http://wordpress.org/latest.tar.gz
tar xvfz latest.tar.gz
cp -Rvf /root/wordpress/* /var/www/html
cd /var/www/html
cp wp-config-sample.php wp-config.php
```

- Cấu hình Database vừa tạo ở trên vào cấu hình của Wordpress:

```
vi wp-config.php
```

```
/** The name of the database for WordPress */
define( 'DB_NAME', 'wpdatabase' );

/** MySQL database username */
define( 'DB_USER', 'wpuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'Wordpress' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```

- Đặt Host `172.16.2.72 anhtq.xyz` để truy cập Wordpress theo tên miền:

<img src="https://imgur.com/UrGuJe1.png">

<img src="https://imgur.com/eHhs1q5.png">

## Phần 6. Đưa web server apache vào pacemaker để quản lý

### Đưa web server apache vào pacemaker để quản lý kiểu đổi port 80 sang port khác

Thực hiện trên cả 3 node

Lúc này luồng request sẽ là `Người dùng` -> `IP_VIP` -> `Truy cập đến 3 node`.

Chuyển port web server:

- Trên node 1:

```
sed -i "s/Listen 80/Listen 10.10.40.71:8081/g" /etc/httpd/conf/httpd.conf
systemctl restart httpd
```

- Tương tự trên `node2` và `node3` với IP VLAN40 tương ứng trên Node đó.

```
vi /etc/haproxy/haproxy.cfg
```

- Thêm đoạn cấu hình sau:

```
listen web-backend
    bind *:80
    balance  roundrobin
    cookie SERVERID insert indirect nocache
    mode  http
    option  httpchk
    option  httpclose
    option  httplog
    option  forwardfor
    server node1 10.10.40.71:8081 check cookie node1 inter 5s fastinter 2s rise 3 fall 3
    server node2 10.10.40.72:8081 check cookie node2 inter 5s fastinter 2s rise 3 fall 3
    server node3 10.10.40.72:8081 check cookie node3 inter 5s fastinter 2s rise 3 fall 3
```

- Restart lại `haproxy` và `pcs`:

```
systemctl restart haproxy

pcs resource restart Loadbalancer_HaProxy
```

### Đưa web server apache vào pacemaker để quản lý kiểu giữ nguyên port 80

Chuyển port web server:

```
sed -i "s/Listen 80/Listen 10.10.40.71:80/g" /etc/httpd/conf/httpd.conf
systemctl restart httpd
```

- Tương tự với `node2` và `node3`

- Cấu hình lại listen haproxy trên cả 3 node

```
listen web-backend
    bind 172.16.2.72:80
    balance  roundrobin
    cookie SERVERID insert indirect nocache
    mode  http
    option  httpchk
    option  httpclose
    option  httplog
    option  forwardfor
    server node1 10.10.40.71:80 check cookie node1 inter 5s fastinter 2s rise 3 fall 3
    server node2 10.10.40.72:80 check cookie node2 inter 5s fastinter 2s rise 3 fall 3
    server node3 10.10.40.73:80 check cookie node3 inter 5s fastinter 2s rise 3 fall 3
```

```
systemctl restart haproxy
```

## Phần 7. Một số lưu ý

Mô hình trên mysql bind qua IP public tới các IP Public của các node client

Chuyển về quy trình Connect từ ngoài vào -> IP VIP Public 3306 -> Tới các client qua được haproxy xử lý qua IP LAB

**Thực hiện trên cả 3 node**

- Sửa file hosts về IP Local:

```
vi /etc/hosts
```

```
10.10.40.71 node1
10.10.40.72 node2
10.10.40.73 node3
```

- Cấu hình lại haproxy trên cả 3 node:

```
vi /etc/haproxy/haproxy.cfg
```

```
listen galera
    bind 172.16.2.72:3306
    balance source
    mode tcp
    option tcpka
    option tcplog
    option clitcpka
    option srvtcpka
    timeout client 28801s
    timeout server 28801s
    option mysql-check user haproxy
    server node1 10.10.40.71:3306 check inter 5s fastinter 2s rise 3 fall 3
    server node2 10.10.40.72:3306 check inter 5s fastinter 2s rise 3 fall 3 backup
    server node3 10.10.40.73:3306 check inter 5s fastinter 2s rise 3 fall 3 backup
```

- Cấu hình lại mysql, galera trên cả 3 node bind-address về IP Local của node đó:

```
vi /etc/my.cnf.d/server.cnf
```

```
[server]
[mysqld]
bind-address=10.10.40.71

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://10.10.40.71,10.10.40.72,10.10.40.73"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="db_cluster"
# Allow server to accept connections on all interfaces.
bind-address=10.10.40.71
# this server ip, change for each server
wsrep_node_address="10.10.40.71"
# this server name, change for each server
wsrep_node_name="node1"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]
```

- Restart lại các services:

```
systemctl restart mariadb
systemctl restart haproxy
```