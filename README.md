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

### 1. Các file cấu hình:
###### cấu trúc file:
```
weather_monitor/
│
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── html/
│       └── index.html
└── flask_api/
    ├── app.py
    └── requirements.txt
```
###### file docker_compose.yml
```
version: '3.8'

services:
  # 1. Database lưu giá trị tức thời
  mariadb:
    image: mariadb:10.6
    container_name: weather_mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: weather_db
    ports:
      - "3306:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - weather_net

  # 2. Database lưu lịch sử (Time-series)
  influxdb:
    image: influxdb:1.8
    container_name: weather_influxdb
    restart: always
    environment:
      - INFLUXDB_DB=weather_history
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb
    networks:
      - weather_net

  # 3. Node-RED thu thập dữ liệu và xử lý Alert
  nodered:
    image: nodered/node-red:latest
    container_name: weather_nodered
    restart: always
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    depends_on:
      - mariadb
      - influxdb
    networks:
      - weather_net

  # 4. Flask API (Tự build để lấy dữ liệu tức thời từ MariaDB)
  flask_api:
    build: ./flask_api
    container_name: weather_flask_api
    restart: always
    ports:
      - "5000:5000"
    depends_on:
      - mariadb
    networks:
      - weather_net

  # 5. Grafana trực quan hóa dữ liệu lịch sử
  grafana:
    image: grafana/grafana:latest
    container_name: weather_grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ALLOW_EMBEDDING=true # Cho phép nhúng iframe vào web
      - GF_AUTH_ANONYMOUS_ENABLED=true   # Cho phép xem biểu đồ không cần login
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - influxdb
    networks:
      - weather_net

  # 6. Web Server Nginx (Chạy Frontend)
  nginx:
    image: nginx:alpine
    container_name: weather_nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/html:/usr/share/nginx/html:ro
    depends_on:
      - flask_api
    networks:
      - weather_net

networks:
  weather_net:
    driver: bridge

volumes:
  mariadb_data:
  influxdb_data:
  nodered_data:
  grafana_data:

```
<img width="393" height="296" alt="image" src="https://github.com/user-attachments/assets/17399ad0-012e-4edc-8b15-91ba8a7cc469" />

###### file flask_api/requirements.txt
```
Flask
Flask-Cors
mysql-connector-python
```
<img width="343" height="101" alt="image" src="https://github.com/user-attachments/assets/00f4a2fe-1b2b-4e5a-a1ef-fb22dd93c067" />

###### file flask_api/app.py
```

```
<img width="538" height="295" alt="image" src="https://github.com/user-attachments/assets/9a6d4523-a1ce-4e47-a9da-f84b1daab28f" />

###### file flask_api/file dockerfile (Để Docker Compose build được service flask_api)
<img width="343" height="155" alt="image" src="https://github.com/user-attachments/assets/1f43864d-dc1e-4509-8127-b1bdc7bbde87" />

###### 
