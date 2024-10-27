This is same README.md but making sure main server can process continous frames and this bellow readme written by Vietnamese

### **Tổng quan kiến trúc hệ thống**

1. **App Client**:
   - Người dùng mở camera để quay video trực tiếp về bề mặt đường.
   - App liên tục **gửi từng frames** từ video lên **Server Node.js**.
2. **Node.js Server**:

   - Đảm bảo NodeJS nhận frames by frames từ App Flutter (tầm 10fps)
   - Nhận từng frame từ App.
   - Thực hiện một số tiền xử lý cần thiết.
   - Gửi các frames tới **Triton Server** thông qua một hàng đợi Kafka để thực hiện inference phát hiện pothole/crack.
   - Nhận kết quả từ inference và thực hiện hậu xử lý.
   - Gửi phản hồi kết quả về App hoặc lưu trữ để phân tích sau.

3. **Kafka**:

   - Dùng làm hệ thống **message broker** để quản lý và phân phối frames từ Node.js Server tới các Worker xử lý inference.
   - Kafka giúp hệ thống xử lý bất đồng bộ, giảm tải cho Node.js Server và có khả năng chịu tải cao.

4. **Triton Server**:
   - Chạy mô hình YOLOv8 đã được huấn luyện để phát hiện pothole/crack.
   - Kết nối với các Worker qua gRPC để xử lý các frames từ Node.js Server.
   - Trả kết quả vị trí pothole/crack về Node.js Server.

### **Phân tích chi tiết từng thành phần**

#### **1. App Client**

- **Chức năng chính**:
  - Mở camera và liên tục gửi các frames từ video tới Node.js Server.
  - Định dạng của các frame có thể là **JPEG** hoặc **PNG** để tối ưu dung lượng truyền tải.
  - Gửi qua **HTTP POST** hoặc sử dụng **WebSocket** nếu muốn giữ kết nối liên tục để giảm độ trễ.

#### **2. Node.js Server**

- **Chức năng chính**:

  - Nhận frames từ App Client.
  - Tiền xử lý dữ liệu, ví dụ như thay đổi kích thước, chuẩn hóa ảnh nếu cần.
  - Đẩy các frames vào hàng đợi Kafka để xử lý bất đồng bộ.
  - Nhận kết quả từ Kafka Consumer hoặc trực tiếp từ Worker và trả kết quả cho App.

- **Workflow chi tiết**:

  1. **Nhận Frame từ App**: Mỗi frame được nhận từ App sẽ được chuyển thành một task để gửi tới Kafka.
  2. **Gửi Frame tới Kafka**: Node.js sẽ đóng gói thông tin frame (như Buffer ảnh, metadata) vào một message và gửi tới topic trong Kafka.
  3. **Nhận kết quả từ Worker**: Sau khi Worker hoàn tất việc inference, kết quả được trả về qua Kafka.
  4. **Hậu xử lý và phản hồi**: Node.js Server nhận kết quả, thực hiện các bước hậu xử lý (nếu cần) và phản hồi lại App.

- **Công nghệ gợi ý**:
  - **Express.js** để xây dựng API.
  - **Kafka Node.js Client** (`kafkajs` hoặc `node-rdkafka`) để giao tiếp với Kafka.

#### **3. Kafka - Message Broker**

- **Chức năng chính**:

  - Quản lý việc truyền tải frames từ Node.js Server tới các Worker xử lý inference.
  - **Topic**: Bạn sẽ cần ít nhất một topic cho các frames cần xử lý, ví dụ `pothole_detection`.
  - **Partition**: Sử dụng partition của Kafka để phân tải công việc đến nhiều Worker.
  - **Consumer Group**: Các Worker thuộc cùng một nhóm tiêu dùng để đảm bảo mỗi frame chỉ được xử lý một lần.

- **Chi tiết về cấu hình**:
  - **Producer**: Node.js Server sẽ đóng vai trò là Producer gửi frames tới Kafka.
  - **Consumer**: Các OCR Worker sẽ là Consumer lấy frames từ Kafka và gửi tới Triton Server để inference.

#### **4. Worker - Inference Process**

- **Chức năng chính**:

  - Nhận frames từ Kafka, gửi đến Triton Server để inference và trả kết quả về lại Kafka.
  - Xử lý bất đồng bộ để giảm tải cho hệ thống Node.js Server.
  - Worker cần sử dụng **gRPC client** để giao tiếp với Triton Server.

- **Workflow chi tiết**:
  1. **Nhận Frame từ Kafka**: Worker sẽ lấy frame từ topic trong Kafka.
  2. **Gửi Frame tới Triton Server** qua gRPC để chạy mô hình YOLOv8 và nhận kết quả phát hiện pothole/crack.
  3. **Gửi Kết quả lại Kafka**: Worker đóng gói kết quả phát hiện (vị trí pothole/crack) và gửi vào một topic kết quả trong Kafka (ví dụ `detection_results`).

#### **5. Triton Server**

- **Chức năng chính**:
  - Chạy mô hình YOLOv8 để phát hiện pothole/crack.
  - Nhận frames qua gRPC từ các Worker và trả về kết quả inference.
- **Workflow**:
  1. **Chuẩn bị mô hình YOLOv8**: Đảm bảo mô hình YOLOv8 đã được huấn luyện và chuyển đổi sang định dạng tương thích với Triton (ONNX hoặc TensorRT).
  2. **Cấu hình Triton**: Tạo cấu hình `config.pbtxt` để thiết lập batch size, kiểu dữ liệu đầu vào, đầu ra.
  3. **Inference**: Xử lý inference các frames theo batch nếu có thể để tăng hiệu suất.
  4. **Trả kết quả**: Trả lại kết quả cho Worker qua gRPC.

#### **Workflow hoàn chỉnh chi tiết**

1. **App Client**:

   - Người dùng mở camera, App gửi frames lên **Node.js Server** liên tục qua HTTP POST hoặc WebSocket.

2. **Node.js Server**:

   - Nhận frame từ App.
   - Đóng gói frame vào message và gửi tới Kafka topic `pothole_detection`.

3. **Kafka**:

   - Topic `pothole_detection` chứa các frames cần xử lý.
   - Các Worker (Consumer) sẽ lấy frames từ topic này để xử lý.

4. **Worker (Consumer)**:

   - Nhận frame từ Kafka.
   - Gửi frame tới **Triton Server** qua gRPC để inference bằng YOLOv8.
   - Nhận kết quả phát hiện pothole/crack từ Triton.
   - Gửi kết quả về Kafka vào topic `detection_results`.

5. **Node.js Server**:
   - Lắng nghe Kafka topic `detection_results` để lấy kết quả từ Worker.
   - Thực hiện hậu xử lý kết quả, lưu trữ hoặc phản hồi lại cho App.

### **Tối ưu hóa và Mở rộng**

#### **1. Tối ưu hóa mô hình YOLOv8**

- Sử dụng **TensorRT** để tối ưu mô hình YOLOv8 khi deploy trên Triton Server.
- Điều chỉnh kích thước batch cho mô hình YOLOv8 trong `config.pbtxt` để xử lý nhiều frames cùng lúc.
- Kiểm tra việc nén và chuẩn hóa ảnh trước khi gửi tới Triton để giảm độ trễ.

#### **2. Tối ưu Kafka**

- **Partitioning**: Sử dụng nhiều partition cho topic để phân tải đến các Worker khác nhau.
- **Replication**: Thiết lập replication để đảm bảo tính chịu lỗi.
- **Consumer Group**: Sử dụng Consumer Group để mở rộng số lượng Worker xử lý đồng thời.

#### **3. Tối ưu Worker**

- Sử dụng **Batching**: Worker có thể xử lý các frames theo batch nếu yêu cầu đến gần nhau trong khoảng thời gian ngắn.
- **Load Balancing**: Nếu có nhiều Worker, sử dụng load balancer để phân phối tải đều.

### **Công nghệ và công cụ gợi ý**

1. **Node.js**:

   - `kafkajs` hoặc `node-rdkafka` để làm việc với Kafka.
   - `grpc` package cho gRPC client để giao tiếp với Triton Server.

2. **Kafka**:

   - Dùng **Kafka Manager** để quản lý và theo dõi các topic, partition, consumer.
   - Thiết lập **schema registry** nếu cần quản lý schema cho các message truyền qua Kafka.

3. **Triton Server**:
   - `ONNX` hoặc `TensorRT` để triển khai mô hình YOLOv8.
   - `tritonserver` command line để khởi chạy và quản lý server.
   - **Prometheus/Grafana** để giám sát hiệu suất inference nếu cần.

### **Ưu điểm của kiến trúc này**

- **Tính bất đồng bộ**: Nhờ Kafka, Node.js Server không bị ảnh hưởng bởi độ trễ khi xử lý inference.
- **Mở rộng dễ dàng**: Có thể thêm nhiều Worker nếu lượng request tăng cao mà không cần thay đổi hệ thống chính.
- **Hiệu suất cao**: Triton Server và mô hình YOLOv8 được tối ưu hóa cho GPU, hỗ trợ batch inference.

Nếu bạn cần thêm chi tiết về cấu hình cụ thể hoặc ví dụ code mẫu cho từng phần, hãy cho mình biết nhé!
