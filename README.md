# BÀI TẬP 5: HỆ THỐNG GIÁM SÁT REALTIME VỚI DOCKER COMPOSE
**Sinh viên thực hiện:** Lương Ngọc Nam  
**Mã số sinh viên (MSSV):** K225480106049   
**Chuyên ngành:** Kỹ thuật Máy tính (Computer Engineering)

---

## PHẦN I: LÝ THUYẾT BÁO CÁO

### 1. Docker là gì?
**Docker** là một nền tảng mã nguồn mở cho phép lập trình viên tự động hóa việc đóng gói, triển khai và quản lý ứng dụng dưới dạng các **Container** độc lập. 

Khác với công nghệ ảo hóa truyền thống (Virtual Machines - VM) đòi hỏi phải chạy một Hệ điều hành khách (Guest OS) hoàn chỉnh cho mỗi máy ảo, Docker sử dụng chung nhân hệ điều hành (Kernel) của máy Host và cô lập ứng dụng ở tầng tiến trình (Process-level). 

* **Container:** Là một thực thể chạy độc lập, chứa toàn bộ những gì ứng dụng cần (mã nguồn, runtime, thư viện hệ thống, cấu hình) nhằm đảm bảo ứng dụng có thể chạy đồng nhất trên mọi môi trường.

---

### 2. Các từ khóa (Keywords) sử dụng trong `docker-compose.yml`

File `docker-compose.yml` dùng để định nghĩa và quản lý đa container. Dưới đây là các từ khóa cốt lõi chia theo 3 thành phần chính:

#### A. Thuộc tính mô tả Dịch vụ (Services)
* **`image`**: Chỉ định Docker Image (từ Docker Hub hoặc local) được dùng làm phôi để build container.
    * *Ý nghĩa:* Giúp Docker biết cần tải bản phân phối hay dịch vụ nào về chạy.
    * *Ví dụ minh họa:* `image: mariadb:10.6`
* **`build`**: Dùng khi muốn tự tạo image cục bộ dựa trên một tệp `Dockerfile` thay vì kéo image có sẵn.
    * *Ý nghĩa:* Trỏ đến thư mục chứa mã nguồn và tệp hướng dẫn build.
    * *Ví dụ minh họa:* `build: ./flask_api`
* **`ports`**: Ánh xạ (mapping) cổng từ Máy Host vào cổng bên trong Container theo cú pháp `[Cổng_Host]:[Cổng_Container]`.
    * *Ý nghĩa:* Giúp bên ngoài máy host có thể truy cập được dịch vụ nằm biệt lập bên trong container.
    * *Ví dụ minh họa:* `- "5000:5000"` (Truy cập localhost:5000 ở máy ngoài sẽ dẫn thẳng vào cổng 5000 của Flask).
* **`environment`**: Thiết lập các biến môi trường cấu hình bên trong container.
    * *Ý nghĩa:* Truyền các tham số cấu hình bảo mật như mật khẩu, tên database, tài khoản mà không cần sửa code.
    * *Ví dụ minh họa:*
        ```yaml
        environment:
          - MYSQL_ROOT_PASSWORD=root_password
          - MYSQL_DATABASE=monitor_db
        ```
* **`depends_on`**: Định nghĩa thứ tự ưu tiên khởi chạy giữa các dịch vụ.
    * *Ý nghĩa:* Đảm bảo các service nền tảng (như Database) phải chạy trước thì service ứng dụng (như Node-RED, Flask) mới khởi chạy sau.
    * *Ví dụ minh họa:*
        ```yaml
        depends_on:
          - mariadb
        ```
* **`restart`**: Quy định chính sách tự động khởi động lại của container khi xảy ra sự cố đột ngột.
    * *Ý nghĩa:* Đảm bảo tính sẵn sàng cao của hệ thống. Giá trị `unless-stopped` giúp container tự bật lại trừ khi bị lập trình viên chủ động tắt.
    * *Ví dụ minh họa:* `restart: unless-stopped`

#### B. Thuộc tính mô tả Lưu trữ (Volumes)
* **`volumes` (Thuộc tính con trong Service):** Thực hiện gắn kết (gắn thẻ) một thư mục lưu trữ của máy host hoặc một Volume đặt tên vào thư mục bên trong container.
    * *Ý nghĩa:* Giúp bảo toàn dữ liệu. Khi container bị xóa hoặc cập nhật, dữ liệu (ví dụ database) vẫn nằm an toàn trên ổ cứng máy host.
    * *Ví dụ minh họa:* `- mariadb_data:/var/lib/mysql`
* **`volumes` (Cấp gốc - Top-level):** Khai báo danh sách các volume độc lập do Docker trực tiếp quản lý tập trung.
    * *Ví dụ minh họa:*
        ```yaml
        volumes:
          mariadb_data:
        ```

#### C. Thuộc tính mô tả Mạng nội bộ (Networks)
* **`networks` (Thuộc tính con trong Service):** Chỉ định container tham gia vào mạng nội bộ nào.
    * *Ý nghĩa:* Cho phép các container trong cùng mạng có thể gọi nhau thông qua **Tên Service** (Cơ chế DNS nội bộ của Docker).
    * *Ví dụ minh họa:* `- monitor_network`
* **`networks` (Cấp gốc - Top-level):** Khai báo khởi tạo mạng ảo độc lập.
    * *Ý nghĩa:* Cách ly luồng mạng của dự án này với các dự án khác trên máy host để tăng tính bảo mật.
    * *Ví dụ minh họa:*
        ```yaml
        networks:
          monitor_network:
            driver: bridge
        ```

---

### 3. Ưu điểm khi triển khai ứng dụng sử dụng Docker

* **Nhất quán môi trường (Environment Consistency):** Loại bỏ hoàn toàn lỗi *"Code chạy tốt trên máy em nhưng lên máy khác bị lỗi"*. Mọi thành phần phụ thuộc (Dependencies), thư viện, hệ điều hành nền đều được đóng gói cố định vào Image.
* **Tối ưu hóa tài nguyên phần cứng:** Do không cần giả lập toàn bộ phần cứng và hệ điều hành khách như máy ảo VM, các container chạy trực tiếp trên OS Host nên khởi động siêu nhanh (tính bằng giây) và tiêu tốn cực ít RAM/CPU.
* **Quản lý kiến trúc Microservices dễ dàng:** Giúp chia nhỏ hệ thống thành các khối độc lập (ví dụ bài tập này gồm 6 khối dịch vụ riêng biệt). Một container lỗi hoặc cần nâng cấp sẽ không gây ảnh hưởng trực tiếp hay làm sập toàn bộ hệ thống còn lại.
* **Tính di động cực cao (Portability):** Chỉ cần một tệp cấu hình `docker-compose.yml`, hệ thống có thể lập tức triển khai mượt mà trên Windows, Linux, macOS hoặc các hạ tầng điện toán đám mây lớn mà không cần cài đặt môi trường thủ công.

---

### 4. Quy trình triển khai ứng dụng lên Máy chủ vật lý KHÔNG CÓ INTERNET (Môi trường Offline)

Khi máy chủ thật bị cô lập hoàn toàn về mạng mạng internet, quy trình chuyển giao hệ thống ứng dụng Docker được tiến hành nghiêm ngặt qua 4 bước sau:

#### Môi trường A (Laptop cá nhân - Có Internet)
* **Bước 1: Đóng gói và Kiểm thử hệ thống**
    Triển khai toàn bộ dự án bằng `docker compose up -d`, thực hiện test các chức năng thu thập, hiển thị dữ liệu và cảnh báo hoạt động mượt mà để chắc chắn các image đã được tải đầy đủ về bộ nhớ cache cục bộ.
* **Bước 2: Xuất (Export) các Image thành file nén vật lý**
    Sử dụng lệnh `docker save` gom tất cả các Image cần thiết của dự án lại thành một tệp đóng gói định dạng `.tar`.
    ```bash
    docker save -o backup_bt5_images.tar monitor_nginx monitor_flask_api nodered/node-red:latest mariadb:10.6 influxdb:2.7 grafana/grafana:latest
    ```
* **Bước 3: Sao chép tài nguyên chuyển giao**
    Copy file nén `backup_bt5_images.tar` cùng toàn bộ thư mục dự án (chứa file `docker-compose.yml`, mã nguồn thư mục `flask_api/`, thư mục web `frontend/`) vào thiết bị lưu trữ ngoại vi (USB, Ổ cứng di động).

#### Môi trường B (Máy chủ thật - Không có Internet)
*(Yêu cầu: Máy chủ đã được cài đặt sẵn Docker Engine offline qua gói cài đặt cục bộ `.deb` hoặc `.rpm` trước đó).*

* **Bước 4: Nạp (Load) Image và Khởi chạy hệ thống**
    * Cắm USB vào máy chủ, copy toàn bộ file và thư mục mã nguồn vào vị trữ lưu trữ trên máy chủ.
    * Thực hiện giải nén nạp các Image vào Docker Engine của máy chủ bằng lệnh:
        ```bash
        docker load -i backup_bt5_images.tar
        ```
    * Di chuyển vào thư mục chứa tệp `docker-compose.yml` trên máy chủ và kích hoạt hệ thống offline bằng lệnh:
        ```bash
        docker compose up -d
        ```

---

## PHẦN II: THỰC HÀNH ÁP DỤNG (MONITOR & ALERT REALTIME)

### 1. Kiến trúc thư mục dự án `bt5`
Hệ thống được tổ chức phân rã cấu trúc theo mô hình Microservices:
```text
bt5/
├── docker-compose.yml
├── flask_api/
│   ├── app.py
│   └── Dockerfile
└── frontend/
    └── index.html

```

> **Minh chứng 1:** Cấu trúc các thư mục `flask_api`, `frontend` tạo thành công trong hệ điều hành:

<img width="1623" height="181" alt="image" src="https://github.com/user-attachments/assets/fd534ba2-1d2c-46f1-a653-45720056fc59" />

### 2. Kích hoạt hệ thống Multi-Container với Docker Compose
Sử dụng tệp cấu hình `docker-compose.yml` tổng hợp để kéo (pull), build và thiết lập mạng kết nối cho 6 dịch vụ đồng thời.

```bash
docker compose up -d
```

### 3. Cấu hình Logic dòng chảy dữ liệu trên Node-RED
* **Thu thập:** Node-RED tạo luồng nghiệp vụ gọi HTTP Request lấy dữ liệu liên tục sau một chu kỳ ngắn (ví dụ lấy dữ liệu chứng khoán hoặc thời tiết).
* **Lưu trữ:** Dữ liệu tức thời được đẩy qua Driver MySQL vào bảng `current_data` trong **MariaDB**, dữ liệu chuỗi lịch sử được ghi nhận vào **InfluxDB**.
* **Cảnh báo (Alert):** Sử dụng Node Function để phân tách ngưỡng bất thường. Khi dữ liệu vượt ngưỡng an toàn ($[A..B]$), kích hoạt Bot Telegram gửi tin nhắn định dạng rõ ràng vào Group chứa 3 thành viên (bao gồm User ID `1875746636`).

> **Minh chứng 3:** Giao diện lập trình Flow và khối kết nối thành công của Node-RED:
<img width="1919" height="1077" alt="image" src="https://github.com/user-attachments/assets/054f3f47-1d10-4ee3-9c0f-9589f5655409" />

> **Minh chứng 4:** Tin nhắn cảnh báo gửi về nhóm Telegram hiển thị rõ giá trị lỗi vượt ngưỡng:
<img width="1555" height="1079" alt="image" src="https://github.com/user-attachments/assets/be2383d9-1645-42ba-b3ae-280fccae3115" />


---

### 4. Giao diện Giám sát Dashboard (Nginx Frontend + Flask API + Grafana)
* Nginx Web Server chạy giao diện tĩnh `index.html` (chứa các thành phần HTML/CSS/JS).
* Đoạn mã Javascript chạy ngầm gọi AJAX đến Flask API (`cổng 5000`) để truy xuất giá trị tức thời từ MariaDB hiển thị thời gian thực lên khối trung tâm.
* Khung `iFrame` nhúng trực tiếp Panel đồ thị đường từ Grafana hiển thị trực quan xu hướng lịch sử đã lưu trữ trong InfluxDB.

> **Minh chứng 5:** Giao diện trang Web Monitor hiển thị số liệu tự động nhảy và đồ thị nhúng mượt mà:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/80b5fd55-a19f-46b3-ba2a-e7eaf472981b" />


---

### 5. Kịch bản Đóng gói - Xóa - Khôi phục Hệ thống Offline
Thực hiện giả lập xuất hệ thống thành tệp nén, dọn dẹp môi trường cũ để chứng minh tính độc lập và khả năng khôi phục nguyên trạng trên máy chủ không có internet.

```bash
# 1. Đóng gói hệ thống ra file .tar vật lý
sudo cp ~/luongngocnam_bttl.tar.gz ~/monitor-app/frontend/

```

<img width="1919" height="1078" alt="image" src="https://github.com/user-attachments/assets/4ef60dae-c9c0-4b28-8bb1-5a62c949fd68" />


# 2. Xóa sạch container và các image cũ trên máy host để làm sạch môi trường
```
docker compose down
sudo rm -rf ~/monitor-app
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/d16146dc-5cbd-4b04-b5bd-f1b55f730f0a" />


# 3. Khôi phục hoàn toàn từ file nén vật lý không cần Internet
```
tar -xzvf ~/luongngocnam_bttl.tar.gz -C ~
docker compose up -d

```
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/af5cb7e0-68db-444b-bee4-d0d1aae78954" />



> **Minh chứng 6:** Lệnh nạp lại thành công ảnh đĩa từ file nén và hệ thống tái khởi động bình thường, dữ liệu được giữ nguyên vẹn thông qua các ổ đĩa cục bộ (Volumes):

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/c9c599c7-5aba-4ca9-b341-4430573a067a" />

<img width="1919" height="1075" alt="image" src="https://github.com/user-attachments/assets/1bedfcd3-bd80-425b-a996-91386db17f3b" />



