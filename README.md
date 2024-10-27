### **Architecture**

1. **FastAPI Server** (Receives Requests from APP)

   - Like before, the FastAPI Server will continue receiving images from the application (APP).
   - It sends image processing requests to **Kafka** as asynchronous tasks.
   - Listens for results from the Workers and returns the processed results to the APP.

2. **Detection Worker** (Uses YOLOv8 to detect license plates from images in Kafka)

   - Listens to requests from Kafka, retrieving images that need processing.
   - Connects with the **Nvidia Triton Server** via gRPC to perform YOLOv8 inference.
   - Receives results from Triton and sends them back to Kafka for the FastAPI Server.

3. **Nvidia Triton Server**
   - Handles inference tasks for images sent from the **Detection Worker** using the YOLOv8 model.
   - Supports batch processing of images to optimize performance.

### **Detailed Analysis of Each Adjusted Component**

#### **1. FastAPI Server**

- **Main Function**: No changes compared to the initial architecture.

#### **2. Detection Worker** (using YOLOv8)

- **Main Function**:

  - Listens for requests from the Kafka Topic **"detection_requests"** (renamed from **"ocr_requests"** to better reflect the use of YOLOv8).
  - Processes images by connecting to the Nvidia Triton Server via gRPC with the YOLOv8 model.
  - Receives inference results and sends them back to Kafka Topic **"detection_results"**.

- **Workflow**:

  1. **Detection Worker** listens to images from Kafka Topic **"detection_requests"**.
  2. Upon receiving an image, it sends the image to the **Nvidia Triton Server** via gRPC for YOLOv8 inference.
  3. Gets results from the Triton Server, including information on detected objects and positions (specifically, license plates).
  4. Sends processed results back to Kafka Topic **"detection_results"**.

- **Recommended Technology**: Similar to the **OCR Worker**, but switch from PaddleOCR to YOLOv8.
  - **YOLOv8** can be deployed using ONNX or TensorRT to take advantage of GPU performance.
  - Configure the YOLOv8 model on the Nvidia Triton Server.

#### **3. Nvidia Triton Server**

- **Main Function**:

  - Receives images from the **Detection Worker** and performs inference with the YOLOv8 model.
  - Optimizes inference through batch processing to minimize overhead and maximize GPU resources.

- **Workflow**:

  1. **Detection Worker** sends images to Triton Server via gRPC with the YOLOv8 model.
  2. Triton Server processes images using the deployed YOLOv8 model (optimized with **ONNX** or **TensorRT**).
  3. Returns results to the **Detection Worker**.

### **Complete System Workflow (Adjusted)**

1. **APP** sends a photo of a license plate to the **FastAPI Server**.
2. **FastAPI Server** creates a task containing the image or image path and sends it to **Kafka Topic "detection_requests"**.
3. **Detection Worker** listens to Kafka Topic **"detection_requests"**:
   - Receives the image to be processed from Kafka.
   - Sends the image via gRPC to **Nvidia Triton Server** for YOLOv8 inference.
   - Receives detection results from Triton (including positions and detected license plates).
   - Sends results back to Kafka Topic **"detection_results"**.
4. **FastAPI Server** monitors Kafka Topic **"detection_results"** to retrieve the results corresponding to the initial request.
5. **FastAPI Server** returns the result to **APP** when completed.

### **Detailed Kafka Configuration (Adjusted)**

- **Topic Structure**:
  - **detection_requests**: Stores license plate detection requests.
  - **detection_results**: Stores results after processing from Detection Worker.
- **Consumer Group**: Unchanged from the initial architecture.

### **Communication with Nvidia Triton Server via gRPC (Adjusted)**

- **Definition of the `.proto` file** does not require significant changes, only adjustments to match the output data from YOLOv8.

  - Example:

    ```proto
    syntax = "proto3";

    service DetectionService {
      rpc DetectPlate (DetectionRequest) returns (DetectionResponse);
    }

    message DetectionRequest {
      bytes image_data = 1; // Image data sent as bytes
    }

    message DetectionResponse {
      repeated DetectedObject detected_objects = 1; // List of detected objects
    }

    message DetectedObject {
      string class_name = 1; // Name of the detected object (e.g., "license_plate")
      float confidence = 2;  // Confidence level of the detection
      BoundingBox bbox = 3;  // Location of the object in the image
    }

    message BoundingBox {
      float xmin = 1;
      float ymin = 2;
      float xmax = 3;
      float ymax = 4;
    }
    ```

### **Example FastAPI Code with Kafka**

```python
from fastapi import FastAPI
from kafka import KafkaProducer, KafkaConsumer
import json

app = FastAPI()
producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

@app.post("/detect")
async def detect_plate(image_data: bytes):
    # Send image to Kafka topic "detection_requests"
    producer.send('detection_requests', {'image_data': image_data})
    # Wait for results from Kafka topic "detection_results"
    consumer = KafkaConsumer(
        'detection_results',
        bootstrap_servers='localhost:9092',
        group_id='fastapi_group',
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )
    for message in consumer:
        # When the result is received, return it to the APP
        return message.value
```

### **System Optimization (Adjusted)**

- **Batching**: Configure Triton Server to process image batches to reduce the number of inference calls.
- **Post-processing**: If needed, add a post-inference step to clean up data from YOLOv8 (e.g., filtering out non-license plate objects).
- **Monitoring** and **Load Balancing**: Remain the same as before.
