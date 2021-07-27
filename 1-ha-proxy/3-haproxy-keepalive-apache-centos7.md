# Cài đặt, triển khai Haproxy Keepalive cho Apache trên CentOS 7

## 1. Mô hình hoạt động

<img src="https://imgur.com/uYG5JhR.png">

## 2. Quy hoạch IP 

<img src="https://imgur.com/pb75k0A.png">

## 3. Cài đặt OS cơ bản

Cấu hình trên cả 3 Node:

```
hostnamectl set-hostname node1
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

Update OS:

```
yum install epel-release -y
yum update -y
```

Cài đặt CMD log:

```
curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/ghichep-cmdlog/master/cmdlog.sh | bash
```

Sau khi cài đặt xong tiến hành reboot lại OS:

```
init 6
```

<img src="https://imgur.com/Igkl5nx.png">

## 4. Triển khai Haproxy Keepalive cho Apache trên CentOS 7

### 4.1. Cấu hình Apache

- Trên Node1:

```
yum install httpd -y
cat /etc/httpd/conf/httpd.conf | grep 'Listen 80'
sed -i "s/Listen 80/Listen 10.10.40.71:8081/g" /etc/httpd/conf/httpd.conf
```

```
echo '<h1>Web Node1</h1>' > /var/www/html/index.html
systemctl start httpd
systemctl enable httpd
```

- Thực hiện tương tự trên `Node2` và `Node3`, thay bằng `IP VLAN40` đã được cấu hình vào các Node.

- Kết quả:

<img src="https://imgur.com/8zdAv2c.png">

### 4.2. Cấu hình Keep Alive

- Cài đặt `keepalived` trên 3 Node:

```
yum install keepalived -y
```

- Copy cấu hình Defaults của `keepalived`

```
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
```

```
systemctl start keepalived
systemctl enable keepalived
```

- Cấu hình `/etc/keepalived/keepalived.conf`:

Cấu hình `keepalived` với IPVIP `172.16.2.72/24`

`Node1` cấu hình `MASTER`:

```
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance VI_1 {
    interface eth0
    state MASTER
    virtual_router_id 51
    priority 101
    virtual_ipaddress {
        172.16.2.72/20
    }
    track_script {
        chk_haproxy
    }
}
```

`Node2` và `Node3` cấu hình `BACKUP`:

```
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance VI_1 {
    interface eth0
    state BACKUP
    virtual_router_id 51
    priority 100
    virtual_ipaddress {
        172.16.2.72/20
    }
    track_script {
        chk_haproxy
    }
}
```

Sau đó tiến hành `systemctl restart keepalived` trên cả 3 Node

Kết quả `Node1` nhận IP VIP:

<img src="https://imgur.com/d9sRSvB.png">

### 4.2. Cấu hình haproxy

- Cài đặt `haproxy` trên tất cả các Node:

```
sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y
```

```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

- Thêm cấu hình vào file `/etc/haproxy/haproxy.cfg`

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
    maxconn                 8000
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    retries                 3
    timeout http-request    20s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s

listen stats
    bind *:8080 interface eth0
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats admin if TRUE

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
    server node3 10.10.40.73:8081 check cookie node3 inter 5s fastinter 2s rise 3 fall 3' > /etc/haproxy/haproxy.cfg
```

- Cấu hình log cho HAProxy:

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

- Truy cập vào IP VIP theo link: `http://172.16.2.72:8080/stats` để kiểm qua kết quả

<img src="https://imgur.com/fRz0syc.png">

- Truy cập vào Webserver qua IP VIP:

Cấu hình `sticky session` (thể hiện ở config cookie node1, cookie node2, cookie node3 trong file config của haproxy) trên request vì vậy trong một thời điểm chỉ có thể kết nối tới 1 web server. Để truy cập tới các webserver còn lại mở trình duyệt ẩn danh và truy cập lại.

<img src="https://imgur.com/EzcDcRD.png">

### 4.3. Kiểm tra các trường hợp.

#### Trường hợp Stop Node1

- Tiến hành stop `Node1`:

<img src="https://imgur.com/i6EFpsL.png">

<img src="https://imgur.com/1oKJvKW.png">

- IP VIP nhảy sang `Node2`:

<img src="https://imgur.com/3kkVNre.png">

#### Trường hợp Stop Node1 và Node2

- Stop `Node1` và `Node2`. IP VIP chuyển sang `Node3`:

<img src="https://imgur.com/UkuzUc0.png">

<img src="https://imgur.com/KLesWHn.png">

## Nguồn tham khảo

https://blog.cloud365.vn/linux/haproxy-keepalived-apache/