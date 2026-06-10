# OpenSource-BTVN-05
# SINH VIÊN THỰC HIỆN: LƯƠNG HOÀNG VIỆT - K225480106073
# MÔN HỌC: PHÁT TRIỂN ỨNG DỤNG VỚI MÃ NGUỒN MỞ
# BÀI TẬP 05
# BÀI LÀM
## 1. Tạo Folder opensource05 có cấu trúc như sau:

```text
opensource05/
├── docker-compose.yml       # Khởi chạy 6 service độc lập
├── frontend/                # Thư mục chứa giao diện hiển thị
│   ├── index.html           # File cấu trúc (nhúng Iframe Grafana)
│   ├── style.css            # File làm đẹp
│   └── script.js            # Code Fetch API cập nhật số liệu
├── nginx/                   # Thư mục cấu hình Web Server
│   └── nginx.conf           # Cấu hình cache và chống lỗi web
├── flask_api/               # Thư mục Backend API tự build
│   ├── Dockerfile           # Code đóng gói API
│   ├── requirements.txt     # Các thư viện python (Flask, CORS, MySQL)
│   └── app.py               # Logic kết nối MariaDB và trả JSON
├── mariadb/                 # Dữ liệu & Script khởi tạo
│   └── init.sql             # Chứa câu lệnh tự động tạo bảng
└── nodered_data/            # Thư mục ánh xạ lưu luồng của NodeRED
```

## 2. Viết file Docker-compose.yml

```
services:
  bt5_mariadb:
    image: mariadb:10.6
    container_name: bt5_mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: realtime_db
      MYSQL_USER: bt5user
      MYSQL_PASSWORD: bt5password
    ports:
      - "3306:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./mariadb/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - bt5_net

  bt5_influxdb:
    image: influxdb:1.8
    container_name: bt5_influxdb
    restart: always
    environment:
      - INFLUXDB_DB=sensor_data
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=adminpassword
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb
    networks:
      - bt5_net

  bt5_nodered:
    image: nodered/node-red:latest
    container_name: bt5_nodered
    restart: always
    ports:
      - "1880:1880"
    volumes:
      - ./nodered_data:/data
    networks:
      - bt5_net
    depends_on:
      - bt5_mariadb
      - bt5_influxdb

  bt5_grafana:
    image: grafana/grafana:latest
    container_name: bt5_grafana
    restart: always
    environment:
      - GF_SECURITY_ALLOW_EMBEDDING=true
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - bt5_net
    depends_on:
      - bt5_influxdb

  bt5_flask_api:
    build: ./flask_api
    container_name: bt5_flask_api
    restart: always
    ports:
      - "5000:5000"
    networks:
      - bt5_net
    depends_on:
      - bt5_mariadb

  bt5_nginx:
    image: nginx:alpine
    container_name: bt5_nginx
    restart: always
    ports:
      - "8080:80"
    volumes:
      - ./frontend:/usr/share/nginx/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    networks:
      - bt5_net

networks:
  bt5_net:
    driver: bridge

volumes:
  mariadb_data:
  influxdb_data:
  grafana_data:
```

## 3. Cấu Hình Nginx

```
server {
    listen       80;
    server_name  localhost;

    # Cấu hình đường dẫn thư mục gốc chứa Frontend
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        
        # Hỗ trợ tốt hơn cho việc refresh trang nếu dùng Router (VD: React/Vue)
        try_files $uri $uri/ /index.html;
    }

    # Tối ưu hóa cache cho các file tĩnh (js, css)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        root /usr/share/nginx/html;
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Trang báo lỗi mặc định
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## 4. Cơ sở dữ liệu MariaDB (MySQL)

File khởi tạo Database. Paste vào mariadb/init.sql

```
CREATE DATABASE IF NOT EXISTS realtime_db;
USE realtime_db;

CREATE TABLE IF NOT EXISTS realtime_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sensor_name VARCHAR(50) NOT NULL,
    value FLOAT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO realtime_data (sensor_name, value) VALUES ('MOCK_DATA', 0.0);
```

## 5. Xây dựng Flask API bằng Python
### a) flask_api/requirements.txt

```
Flask==2.3.2
flask-cors==4.0.0
mysql-connector-python==8.0.33
```

### b) flask_api/Dockerfile

```
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

### c) flask_api/app.py

```
from flask import Flask, jsonify
from flask_cors import CORS
import mysql.connector
import time

app = Flask(__name__)
CORS(app) 

def get_db_connection():
    retries = 5
    while retries > 0:
        try:
            conn = mysql.connector.connect(
                host='bt5_mariadb', 
                user='bt5user',
                password='bt5password',
                database='realtime_db'
            )
            return conn
        except mysql.connector.Error as err:
            retries -= 1
            time.sleep(2)
    return None

@app.route('/api/current-data', methods=['GET'])
def get_current_data():
    conn = get_db_connection()
    if conn is None:
        return jsonify({"error": "Database connection failed"}), 500
        
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT sensor_name, value, timestamp FROM realtime_data ORDER BY id DESC LIMIT 1")
    row = cursor.fetchone()
    
    cursor.close()
    conn.close()
    
    if row:
        return jsonify(row)
    else:
        return jsonify({"sensor_name": "N/A", "value": 0, "timestamp": "N/A"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## 6. Xây dựng Giao diện Frontend

Viết giao diện và cấu hình chúng với cấu trúc như sau

```text
opensource05/
└── frontend/            
    ├── index.html          # Giao diện chính
    ├── style.css           # File chỉnh màu sắc, bố cục
    └── script.js           # File xử lý logic gọi API
```       

## 7. Chạy Ứng Dụng BT5
```
Docker compose up -d
```
Sau đó kiểm tra

```
Docker ps
```

<img width="1692" height="212" alt="image" src="https://github.com/user-attachments/assets/eff954c8-ccf3-4cdd-8175-099f0b660db9" />

Cấu hình Cloudflare để Public website lên Internet:

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/7b8f6983-f376-428f-ab12-0df5aade66ec" />

Truy cập https://btvn05.luonghoangviet.io.vn/ để kiểm tra giao diện

<img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/5a0d11a0-5293-4624-9d94-348c96b873af" />

## 8. Cài đặt Node-RED và Logic Cảnh Báo

Truy cập http://192.168.1.10:1880 để tiến hành cấu hình cho Node-RED

### a) Cài đặt Package

node-red-node-mysql, node-red-contrib-influxdb, node-red-contrib-telegrambot.

### b) Tạo Flow cho Node-RED như sau:

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/dc676bd6-142c-48fe-9b12-8fd43d95c68f" />

Cài đặt cảnh báo giá BTC khi không nằm trong khoảng 60000$-61000$:

<img width="626" height="930" alt="image" src="https://github.com/user-attachments/assets/eb6ceeb7-f835-476a-a2de-826edf65df3c" />

### c) Cấu hình kết nối đến telegram

Tạo Bot và lấy token, điền vào Node "TelegramSender"

Tạo Groupchat (lấy ID) điền vào Node "Format tin nhắn tele"

Tùy vào cài đặt, Bot sẽ định kỳ gửi giá BTC vào nhóm sau mỗi N giây (n là số thời gian tự cài theo ý thích)

Kết quả:

<img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/884d0707-6c77-47b9-a8ab-caad173a860b" />

## 9. Cấu Hình Grafana Dashboard

Truy cập https://btvn05.luonghoangviet.io.vn/grafana/login, đăng nhập:

<img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/601a5a43-a1e7-4634-a2b6-09723260bce0" />

Sau đó vào Datasource và cấu hình như sau: 

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/a6c09378-e425-4c5a-a1c6-ff823cf68c0e" />

Save & Test

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/1ecc6069-ecfd-4da9-99b0-218542c108ed" />

Tạo 1 Dashboard mới và lấy dữ liệu giá tiền BTC từ InfluxDB

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/0398ad9a-7405-4fe9-961b-5ec954d6753d" />

Lấy Iframe và nhúng vào FrontEnd ở bước 6

```
<iframe
src="/grafana/d-solo/adr8stf/btc?orgId=1&from=1781111253150&to=1781113053150&timezone=browser&theme=dark&panelId=1"width="100%" height="450" frameborder="0">
</iframe>
```

Truy cập btvn05.luonghoangviet.io.vn để kiểm tra:

<img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/4f2707d2-6c04-4ee8-94d2-c689307927ad" />







