# High Availability: Tổng quan về Cluster

## 1. Giới thiệu về Cluster

Cluster là kiến trúc nâng cao khả năng sẵn sàng cho các hệ thống dịch vụ. Hệ thống Cluster cho phép nhiều máy chủ chạy kết hợp, đồng bộ với nhau. Hệ thống Cluster nâng cao khả năng chịu lỗi của hệ thống, tăng cấp độ tin cậy, tăng tính đảm bảo, nâng cao khả năng mở rộng cho hệ thống. Trong trường hợp có lỗi xảy ra, các dịch vụ bên trong Cluster sẽ tự động loại trừ lỗi, cố gắng khôi phục, duy trì tính ổn định, tính sẵn sàng của dịch vụ

Cluster thường được tìm thấy ở các hệ thống thanh toán trực tuyến, ngân hàng, các cơ sở dữ liệu, hệ thống lưu trữ ..

## 2. Tính năng của Cluster

- Cân bằng tải của cụm (**Load Balancing**): Các node bên trong cluster hoạt động song song, chia sẻ các tác vụ để năng cao hiệu năng.

- Tính sẵn sàng cao (**High availability**): Các tài nguyên bên trong cluster luôn sẵn sàng xử lý yêu cầu, ngay cả khi có vấn đề xảy ra với các thành phần bên trong (hardware, software).

- Khả năng mở rộng (**scalability**): Khi tài nguyên có thể sử dụng của hệ thống tới giới hạn, ta có thể dễ dàng bổ sung thêm tài nguyên vào cluster bằng các bổ sung thêm các node.

- Độ tin cậy (**reliability**): Hệ thống Cluster giảm thiểu tần số lỗi có thể xảy ra, giảm thiểu các vấn đề dẫn tới ngừng hoạt động của hệ thống.

## 3. Các thuật ngữ quan trọng

- **Cluster**: Nhóm các server dành riêng để giải quyết 1 vấn đề, có khả năng kết nối, sản sẻ các tác vụ.

- **Node**: Server thuộc Cluster

- **Failover**: Khi 1 hoặc nhiều node trong Cluster xảy ra vấn đề, các tài nguyên (resource) tự động được chuyển tới các node sẵn sàng phục vụ

- **Failback**: Khi node lỗi phục hồi, sẵn sàng phục vụ, Cluster tự động trả lại tài nguyên cho node

- **Fault-tolerant cluster**: Để cập đến khả năng chịu lỗi của hệ thống trên các thành phần, cho phép dịch vụ hoạt động ngay cả khi một vài thành phần gặp sự cố

- **Heartbeat**: Tín hiệu xuất phát từ các node trong cụm với mục đích xác minh chùng còn sống và đang hoạt động. Nếu heartbeat tại 1 node ngừng hoạt động, cluster sẽ đánh dấu thành phần đó gặp sự cố và chuyển tài nguyên tại node lỗi sang node đang sẵn sàng phục vụ

- **Interconnect**: Kết nối giữa các node. Thường là các thông tin về trạng thái, heartbeat, dữ liệu chia sẻ.

- **Primary server, secondary server**: Trong cluster dạng `Active/Passise`, node đang đáp ứng giải quyết yêu cầu gọi là Primary server. Node đang chờ hay dự phòng cho node `Primary server` được gọi là `secondary server`

- **Quorum**: Trong Cluster chứa nhiều tài nguyên, nên dễ xảy ra sự phân mảnh (split-brain - Tức cluster lớn bị tách ra thành nhiều cluster nhỏ). Điều này sẽ dẫn đến sự mất đồng bộ giữa các tài nguyên, ảnh hướng tới sự toàn vẹn của hệ thống. Quorum được thiết kế để ngăn chặn hiện tượng phân mảnh.

- **Resource**: Tài nguyên của cụm, cũng có thể hiểu là tài nguyên mà dịch vụ cung cấp

- **STONITH/ Fencing**: STONITH là viết tắt của cụm từ Shoot Other Node In The Head đây là một kỹ thuật dành cho fencing. Fencing là kỹ thuật cô lập tài nguyên tại từng node trong Cluster. Mục tiêu STONITH là tắt hoặc khởi động lại node trong trường hợp Node trong trường hợp dịch vụ không thể khôi phục.