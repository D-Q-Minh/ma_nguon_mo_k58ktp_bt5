# Môn: Mã nguồn mở
#### Bài 5: APP MONITOR + ALERT DATA REALTIME
######
    sử dụng docker compose có nhiều serivce 
    và các thành phần cần thiết để tạo thành ứng dụng:
     + nodered liên tục lấy dữ liệu từ nguồn nào đó (chứng khoán, thời tiết, giá vàng,...)
       nguồn thực tế, số liệu luôn động sau thời gian ngắn
     + nodered lưu trữ dữ liệu vào 2 database: mariadb để lưu giá trị tức thời
       lưu lịch sử vào influxdb
     + sử dụng grafana để trực quan hoá dữ liệu: vẽ biểu đồ
     + sử dụng nginx để làm webserver
       chạy 1 trang web html+js+css làm front-end
       js: lấy dữ liệu tức thời trong mariadb qua (ajax | socket) 
           gọi api (api tự build bằng Flask giống bt1)
           api trả về giá trị tức thời trong mariadb
           hiển thị lên web, auto hiển thị số mới khi thay đổi
       sử dụng iframe để gọi grafana
       hiển thị biểu đồ dữ liệu lịch sử của thông số đã lưu
     + QUAN SÁT DỮ LIỆU LỊCH SỬ => GIÁ TRỊ BẤT THƯỜNG
       (VD MIỀN A..B: OK, DƯỚI A: ALERT LOW, TRÊN B: ALERT HIGH)
     + nodered: kết hợp bot Telegram
       khi dữ liệu not OK, thì gửi tin nhắn từ bot => group trên telegram
       group đã add bot vào: (nhóm đã có 2 người), add thêm 1875746636 thành 3 người
       mỗi khi bot gửi dữ liệu vào nhóm: mọi member of group đều nhận đc
       nội dung alert: tường minh, có value gây alert

     xuất tất cả các container ra file nén.
     xoá mọi container đang chạy
     load lại các container  từ file nén để khôi phục các container đã xoá
