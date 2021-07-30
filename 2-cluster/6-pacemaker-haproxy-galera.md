# High Availability - Phần 5: Hướng dẫn triển khai Haproxy Pacemaker cho Cluster Galera 3 node trên CentOS 7

## Tổng quan

**HAProxy** viết tắt của **High Availability Proxy**, là công cụ mã nguồn mở nổi tiếng ứng dụng cho giải pháp cân bằng tải TCP/HTTP cũng như giải pháp máy chủ Proxy (Proxy Server). HAProxy có thể chạy trên các mỗi trường Linux, Solaris, FreeBSD. Công dụng phổ biến nhất của HAProxy là cải thiện hiệu năng, tăng độ tin cậy của hệ thống máy chủ bằng cách phân phối khối lượng công việc trên nhiều máy chủ (như Web, App, cơ sở dữ liệu). HAProxy hiện đã và đang được sử dụng bởi nhiều website lớn như GoDaddy, GitHub, Bitbucket, Stack Overflow, Reddit, Speedtest.net, Twitter và trong nhiều sản phẩm cung cấp bởi Amazon Web Service.

**MariaDB Galera Cluster** là giải pháp sao chép đồng bộ nâng cao tính sẵn sàng cho MariaDB. Galera hỗ trợ chế độ Active-Active tức có thể truy cập, ghi dữ liệu đồng thời trên tất các node MariaDB thuộc Galera Cluster.

**Pacemaker** là trình quản lý tài nguyên trong cluster được phát triển bởi ClusterLabs. Pacemaker tương thích với rất nhiều dịch vụ phổ biến hiện có và hoàn toàn có thể tự phát triển module để quản lý các tài nguyên mà pacemaker chưa hỗ trợ.

## Phần 1. Chuẩn bị

### 1. Mô hình hoạt động

### 2. Quy hoạch IP 

<img src="https://imgur.com/pb75k0A.png">