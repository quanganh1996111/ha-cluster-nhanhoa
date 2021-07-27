# Tổng quan và các khái niệm quan trọng về cân bằng tải trong HAProxy

Tổng quan và các khái niệm quan trọng về cân bằng tải trong HAProxy

## Tổng quan về HAProxy

HAProxy viết tắt của High Availability Proxy, là công cụ mã nguồn mở nổi tiếng ứng dụng cho giải pháp cân bằng tải TCP/HTTP cũng như giải pháp máy chủ Proxy (Proxy Server). HAProxy có thể chạy trên các mỗi trường Linux, Solaris, FreeBSD. Công dụng phổ biến nhất của HAProxy là cải thiện hiệu năng, tăng độ tin cậy của hệ thống máy chủ bằng cách phân phối khối lượng công việc trên nhiều máy chủ (như Web, App, cơ sở dữ liệu). HAProxy hiện đã và đang được sử dụng bởi nhiều website lớn như GoDaddy, GitHub, Bitbucket, Stack Overflow, Reddit, Speedtest.net, Twitter và trong nhiều sản phẩm cung cấp bởi Amazon Web Service.

## Một số thuật ngữ trong HAProxy

### Access Control List (ACL)

Access Control List (ACL) sử dụng để kiểm tra một số điều kiện và thực hiện hành động tiếp theo dựa trên kết quả kiểm tra(VD lựa chọn một server, chặn một request). Sử dụng ACL cho phép điều tiết lưu lượng mạng linh hoạt dựa trên các yếu tố khác nhau (VD: dựa theo đường dẫn, dựa theo số lượng kết nối tới backend)

### Backend

Backend là tập các server nhận các request đã được điều tiết (HAProxy điều tiết các request tới các backend). Các Backend được định nghĩa trong mục `backend` khi cấu hình HAProxy.

2 cấu hình thường được định nghĩa trong mục `backend`:

- Thuật toán cân bằng tải (Round Robin, Least Connection, IP Hash).

- Danh sách các Server, Port (Nhận, xử lý request).

Backend có thể chứa một hoặc nhiều server. Việc thêm nhiều server vào backend sẽ cải thiện tải, hiệu năng, tăng độ tin cậy dịch vụ. Và khi một server thuộc backend không khả dụ, các server khác thuộc backend sẽ chịu tải thay cho server xảy ra vấn đề.

```
backend web-backend
   balance roundrobin
   server web1 web1.yourdomain.com:80 check
   server web2 web2.yourdomain.com:80 check

backend blog-backend
   balance roundrobin
   mode http
   server blog1 blog1.yourdomain.com:80 check
   server blog1 blog1.yourdomain.com:80 check
```

`balance roundrobin` chỉ định thuật toán cân bằng tải: các Request phân phối tuần tự tới các server, đây cũng là phương thức được sử dụng mặc định.

`mode http` chỉ định proxy layer 7 sẽ được sử dụng

### Frontend

Frontend định nghĩa cách các request điều tiết tới backend. Các cấu hình Frontend được định nghĩa trong mục frontend khi cấu hình HAProxy.

Các cấu hình frontend bao gồm các thành phần:

- Tập các IP và port (VD: 10.10.10.86:80, *:443)

- Các ACL

- Các backend nhận, xử lý request.

## Các mô hình Load Balancing

### 1. Mô hình không có Load Balancing

<img src="https://imgur.com/gKkfbAO.png">

Người dùng sẽ kết nối trực tiếp tới Webserver, và không sử dụng dịch vụ cân bằng tải. Nếu web server xảy ra vấn đề, người dùng sẽ không thể kết nối tới Web được nữa. Và nếu trong trường hợp nhiều người cùng truy cập, webserver có thể không đáp ứng được các request, dẫn đến trải nhiệm sử dụng sẽ giảm xuống.

### 2. Layer 4 Load Balancing

Cách đơn giản nhất để cân bằng tải các request tới nhiều server là sử dụng cân bằng tải mức layer 4 TCP (Tầng giao vận - transport layer). Phương pháp sẽ điều hướng các request dựa trên IP và Port. Theo ví dụ, nếu request tới Website thì request sẽ được điều hướng tới backend `web-backend` để xử lý.

<img src="https://imgur.com/TDm1WJ9">

- Hai máy chủ web cần phục vụ nội dung giống nhau. Nếu không, người dùng sẽ nhận thông tin không thống nhất (Tùy theo thuật toán cân bằng tải).

- Nên sử dụng chung database giữ 2 web server.

### 3. Layer 7 Load Balancing

Một cách phức tạp hơn để cân bằng tải lưu lượng mạng là dùng `layer 7 (application layer)`. Dùng layer 7 cho phép load balancer chuyển hướng request đến các máy chủ backend khác nhau dựa trên nội dung request. Chế độ cân bằng tải này cho phép bạn chạy nhiều máy chủ ứng dụng web dưới cùng domain và port.

<img src="https://imgur.com/0f5Gnlv.png">

Phương pháp phức tạp hơn, cân bằng tải sử dụng tại tầng layer 7 mức request (Tầng ứng dụng - Application layer). Sử dụng bộ cần bằng tại layer 7 sẽ điều hướng request tới các backend khác nhau dựa trên nội dung của request.

Chế độ này cho phép bạn có thể triển khai nhiều web server khác nhau trên cùng 1 domain.

Trong ví dụ nếu người dùng yêu cầu yourdomain.com/blog, họ sẽ được chuyển hướng đến blog-backend, là tập các máy chủ chạy ứng dụng blog. Các request khác được chuyển hướng đến web-backend, mà có thể chạy các ứng dụng khác. Trong ví dụ này, cả 2 backend dùng cùng máy chủ database.

Cấu hình 1 frontend tên http sẽ xử lý lưu lượng vào trên port 80:

```
frontend http
  bind *:80
  mode http

  acl url_blog path_beg /blog
  use_backend blog-backend .if url_blog

  default_backend web-backend
```

- Dòng `acl url_blog path_beg /blog` match khi 1 request có đường dẫn bắt đầu với /blog.

- Dòng `use_backend blog-backend if url_blog` dùng ACL để proxy lưu lượng đến blog-backend.

- Dòng `default_backend web-backend` chỉ định rằng tất cả các lưu lượng khác sẽ chuyển hướng đến web-backend.

## Các thuật toán cân bằng tải

Thuật toán cân bằng tải được sử dụng nhắm định nghĩa các request được điều hướng tới các server nằm trong backend trong quá trình load balancing. HAProxy cung cấp một số thuật toán mặc định:

- `roundrobin`: các request sẽ được chuyển đến server theo lượt. Đây là thuật toán mặc định được sử dụng cho HAProxy

- `leastconn`: các request sẽ được chuyển đến server nào có ít kết nối đến nó nhất

- `source`: các request được chuyển đến server bằng các hash của IP người dùng. Phương pháp này giúp người dùng đảm bảo luôn kết nối tới một server

## Sticky Session

Một số ứng dụng yêu cầu người dùng phải giữ kết nối tới cùng một server thuộc backend, để giữ kết nối giữa client với một backend server bạn có thể sử dụng tùy chọn `sticky sessions`, xem thêm [tại đây](https://www.haproxy.com/blog/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/)

## Health Check

HAProxy sử dụng `health check` để phát hiện các backend server sẵn sàng xử lý request. Kỹ thuật này sẽ tránh việc loại bỏ server khỏi backend thủ công khi backend server không sẵn sàng. `health check` sẽ cố gắnh thiết lập kết nối TCP tới server để kiểm tra backend server có sẵn sàng xử lý request.

Nếu `health check` không thể kết nối tới server, nó sẽ tự động loại bỏ server khởi backend, các traffic tới sẽ không được forward tới server cho đến khi nó có thể thực hiện được `health check`. Nếu tất cả server thuộc backend đều xảy vấn đề, dịch vụ sẽ trở trên không khả dụ (trả lại status code 500) cho đến khi 1 server thuộc backend từ trạng thái không khả dụ chuyển sang trạng thái sẵn sàng.

## Mô hình kết hợp với keepalived

<img src="https://imgur.com/RzWel3w.png">

Đối với các cài đặt load balancing `layer 4` hoặc `layer 7` phía trên , chúng sử dụng 1 load balancer để điều khiển traffic tới một hoặc nhiều backend server. tuy nhiên nếu load balancer bị lỗi thì dữ liệu sẽ bị ứ đọng dẫn tới downtime (bottleneck - nghẽn cổ chai). `keepalived` sinh ra để giải quyết vấn đề này.

Nếu có nhiều `load balancer` (1 active và một hoặc nhiều passive). Khi người dùng kết nối đến một server thông qua ip public của active load balancer, nếu load balancer ấy fails, phương thức failover sẽ detect nó và tự động gán ip tới 1 passive server khác.

## Nguồn tham khảo

https://blog.cloud365.vn/linux/tong-quan-haproxy/

http://cbonte.github.io/haproxy-dconv/configuration-1.4.html#4.2-balance