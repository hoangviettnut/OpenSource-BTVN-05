# OpenSource-BTVN-05
# SINH VIÊN THỰC HIỆN: LƯƠNG HOÀNG VIỆT - K225480106073
# MÔN HỌC: PHÁT TRIỂN ỨNG DỤNG VỚI MÃ NGUỒN MỞ
# BÀI TẬP 05
# BÀI LÀM
## 1. Tạo Folder opensource05 có cấu trúc như sau:

opensource05/
├── docker-compose.yml       # Khởi chạy 6 service độc lập
├── frontend/                # Thư mục web UI (Giao diện hiển thị)
│   ├── index.html           # File cấu trúc giao diện & Iframe
│   ├── style.css            # File làm đẹp
│   └── script.js            # Code Fetch API cập nhật số liệu
├── flask_api/               # Thư mục Backend tự build
│   ├── Dockerfile           # Code đóng gói API
│   ├── requirements.txt     # Các thư viện python (Flask, CORS, MySQL)
│   └── app.py               # Logic kết nối MariaDB và trả JSON
├── mariadb/                 # Dữ liệu & Script cho MariaDB
│   └── init.sql             # Chứa câu lệnh tự động tạo bảng
└── nodered_data/            # Thư mục ánh xạ lưu luồng của NodeRED

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
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
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
opensource05/
└── frontend/            
    ├── index.html          # Giao diện chính
    ├── style.css           # File chỉnh màu sắc, bố cục
    └── script.js           # File xử lý logic gọi API
       

  ## 7. Chạy Ứng Dụng BT5


