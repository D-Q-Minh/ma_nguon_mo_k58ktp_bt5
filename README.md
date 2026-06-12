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
###### file docker-compose.yml
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

###### file nginx/nginx.conf
```
events {}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        
        # Thư mục chứa code HTML/JS/CSS Frontend
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        # Reverse proxy qua Flask API để tránh lộ port 5000 ra ngoài
        location /api/ {
            proxy_pass http://weather_flask_api:5000/api/;
        }
    }
}
```
<img width="546" height="357" alt="image" src="https://github.com/user-attachments/assets/d0bb94ef-eb8d-490a-94f5-73382c68e071" />

###### file nginx/html/index.html
```
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Hệ Thống Giám Sát Thời Tiết Realtime</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f4f6f9; text-align: center; padding: 20px; }
        .container { display: flex; justify-content: space-around; margin-bottom: 20px; }
        .card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); width: 40%; }
        .value { font-size: 2.5rem; font-weight: bold; color: #2c3e50; }
        iframe { width: 90%; height: 450px; border: 1px solid #ccc; border-radius: 8px; }
    </style>
</head>
<body>

    <h1>THỜI TIẾT REALTIME & LỊCH SỬ</h1>
    <p>Tự động cập nhật dữ liệu mỗi 5 giây qua AJAX gọi Flask API</p>

    <div class="container">
        <div class="card">
            <h3>Nhiệt Độ Hiện Tại</h3>
            <div id="temp" class="value">-- °C</div>
        </div>
        <div class="card">
            <h3>Độ Ẩm Hiện Tại</h3>
            <div id="humidity" class="value">-- %</div>
        </div>
    </div>

    <h2>Biểu Đồ Lịch Sử (Grafana)</h2>
    <iframe src="http://localhost:3000/d-solo/weather_dashboard/weather-report?orgId=1&panelId=1&refresh=5s" frameborder="0"></iframe>

    <script>
        // Hàm dùng AJAX (Fetch API) lấy dữ liệu liên tục từ Flask
        async function fetchWeatherData() {
            try {
                const response = await fetch('/api/weather/live');
                if (response.ok) {
                    const data = await response.json();
                    document.getElementById('temp').innerText = data.temperature + " °C";
                    document.getElementById('humidity').innerText = data.humidity + " %";
                }
            } catch (error) {
                console.error("Lỗi lấy dữ liệu:", error);
            }
        }

        // Tự động quét lại sau mỗi 5 giây (Realtime)
        setInterval(fetchWeatherData, 5000);
        fetchWeatherData(); // Chạy lần đầu ngay khi load trang
    </script>
</body>
</html>
```
<img width="475" height="218" alt="image" src="https://github.com/user-attachments/assets/e5b2196c-12e9-473f-9cbb-7b5debf102f4" />

##### chạy docker compose up -d
<img width="230" height="275" alt="image" src="https://github.com/user-attachments/assets/3d0536ba-34d7-473e-9dc4-e8a5a466184c" />

### 2. Cấu hình Node-RED
#### 2.1. Cấu hình bot telegram
###### Tạo bot mới bằng @BotFather
<img width="643" height="66" alt="image" src="https://github.com/user-attachments/assets/207e7574-12bb-439e-a9a3-940e9ebbb498" />
###### tạo group, thêm bot vào
<img width="242" height="67" alt="image" src="https://github.com/user-attachments/assets/344a4122-bccb-455b-af86-61143e09c9e6" />
<img width="364" height="76" alt="image" src="https://github.com/user-attachments/assets/84535b87-b275-4684-91b7-95d10623ee88" />
