# Hướng dẫn cài đặt Galera 3 node trên CentOS 7

## Tổng quan
MariaDB là một sản phẩm mã đóng tách ra từ mã mở do cộng đồng phát triển của hệ quản trị cơ sở dữ liệu quan hệ MySQL nhằm theo hướng không phải trả phí với GNU GPL. MariaDB được phát triển từ sự dẫn dắt của những nhà phát triển ban đầu của MySQL, do lo ngại khi MySQL bị Oracle Corporation mua lại. Những người đóng góp được yêu cầu chia sẽ quyền tác giả của họ với MariaDB Foundation.

MariaDB được định hướng để duy trì khả năng tương thích cao với MySQL, để đảm bảo khả năng hỗ trợ về thư viện đồng thời kết hợp một cách tốt nhất với các API và câu lệnh của MySQL. MariaDB đã có công cụ hỗ lưu trữ XtraDB thay cho InnoDB.

MariaDB Galera Cluster là giải pháp sao chép đồng bộ nâng cao tính sẵn sàng cho MariaDB. Galera hỗ trợ chế độ Active-Active tức có thể truy cập, ghi dữ liệu đồng thời trên tất các node MariaDB thuộc Galera Cluster.

## Phần 1. Chuẩn bị

### 1. Mô hình hoạt động

<img src="https://imgur.com/HP7Vjn7.png">

### 2. Quy hoạch IP 

<img src="https://imgur.com/pb75k0A.png">

Chúng ta có thể Roll backup lại và sử dụng Quy Hoạch IP theo như bài [Cài đặt, triển khai Haproxy Keepalive cho Apache trên CentOS 7](https://github.com/quanganh1996111/ha-cluster-nhanhoa/blob/main/1-ha-proxy/3-haproxy-keepalive-apache-centos7.md)

### 3. Thiết lập cấu hình ban đầu

- Thiếu lập hostname cho các Node tương ứng:

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

```
yum install epel-release -y
yum update -y
```

- Cài đặt NTPD:

```
yum install chrony -y 

systemctl start chronyd 
systemctl enable chronyd
systemctl restart chronyd 

chronyc sources -v
```

```
sudo date -s "$(wget -qSO- --max-redirect=0 google.com 2>&1 | grep Date: | cut -d' ' -f5-8)Z"
ln -f -s /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime
```

- Cấu hình file `/etc/hosts` cho các Node:

```
echo "172.16.2.56 node1" >> /etc/hosts
echo "172.16.2.59 node2" >> /etc/hosts
echo "172.16.2.62 node3" >> /etc/hosts
```

## Phần 2. Cài đặt MariaDB

- Khai báo Repo để cài đặt:

```
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo
yum -y update
```

- Cài đặt MariaDB:

```
yum install -y mariadb mariadb-server
```

- Cài đặt Galera và gói hỗ trợ:

```
yum install -y galera rsync
```

- Stop Mariadb. **Lưu ý**: Không khởi động dịch vụ mariadb sau khi cài (Liên quan tới cấu hình Galera Mariadb)

```
systemctl stop mariadb
```

## Phần 3. Cấu hình Galera Cluster

### Node 1:

```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=172.16.2.56

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
bind-address=172.16.2.56
# this server ip, change for each server
wsrep_node_address="10.10.40.71"
# this server name, change for each server
wsrep_node_name="node1"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]
' > /etc/my.cnf.d/server.cnf
```

**Lưu ý**:

- `wsrep_cluster_address`: Danh sách các node thuộc Cluster, sử dụng địa chỉ IP (Ví dụ sử dụng dải IP Replicate 172.16.2.56, 172.16.2.59, 172.16.2.62)

- `wsrep_cluster_name`: Tên của cluster

- `wsrep_node_address`: Địa chỉ IP của node đang thực hiện.

- `wsrep_node_name`: Tên node (Giống với hostname)

- Không được bật mariadb (Quan trọng, nếu không sẽ dẫn tới lỗi khi khởi tạo Cluster)

### Node 2 và Node 3

Cấu hình tương ứng Node 1, **Lưu ý** thay thế các tham số theo đúng IP và Hostname của Node đó.

### Khởi động lại MariaDB

- Tại Node 1:

```
galera_new_cluster
systemctl start mariadb
systemctl enable mariadb
```

- Tại Node 2 và Node 3:

```
systemctl start mariadb
systemctl enable mariadb
```

- Kiểm tra trạng thái tại Node 1:

<img src="https://imgur.com/hOKyixC.png">

## Nguồn tham khảo

https://blog.cloud365.vn/linux/cai-dat-galera-mariadb/

https://github.com/domanhduy/ghichep/blob/master/DuyDM/Cluster-HA/Cluster/docs/2.Cai-dat-galare-3node-centos7.md