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

### Mô hình không có Load Balancing

<img src="https://imgur.com/gKkfbAO.png">

Người dùng sẽ kết nối trực tiếp tới Webserver, và không sử dụng dịch vụ cân bằng tải. Nếu web server xảy ra vấn đề, người dùng sẽ không thể kết nối tới Web được nữa. Và nếu trong trường hợp nhiều người cùng truy cập, webserver có thể không đáp ứng được các request, dẫn đến trải nhiệm sử dụng sẽ giảm xuống.

### Layer 4 Load Balancing

Cách đơn giản nhất để cân bằng tải các request tới nhiều server là sử dụng cân bằng tải mức layer 4 TCP (Tầng giao vận - transport layer). Phương pháp sẽ điều hướng các request dựa trên IP và Port. Theo ví dụ, nếu request tới Website thì request sẽ được điều hướng tới backend `web-backend` để xử lý.

<img src="https://imgur.com/TDm1WJ9">

- Hai máy chủ web cần phục vụ nội dung giống nhau. Nếu không, người dùng sẽ nhận thông tin không thống nhất (Tùy theo thuật toán cân bằng tải).

- Nên sử dụng chung database giữ 2 web server.

### Layer 7 Load Balancing

Một cách phức tạp hơn để cân bằng tải lưu lượng mạng là dùng `layer 7 (application layer)`. Dùng layer 7 cho phép load balancer chuyển hướng request đến các máy chủ backend khác nhau dựa trên nội dung request. Chế độ cân bằng tải này cho phép bạn chạy nhiều máy chủ ứng dụng web dưới cùng domain và port.

<img src="https://imgur.com/0f5Gnlv.png">