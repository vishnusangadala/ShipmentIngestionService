# 📦 Shipment Ingestion Service

The **Shipment Ingestion Service** is the entrypoint for external systems (driver apps, IoT devices, 3rd-party APIs) to send real-time shipment data. It transforms this data into clean Kafka events for internal processing.

---

## 🌟 Responsibilities

### Core Tasks

* Accept external data via:

  * REST API
  * WebSocket (optional)
  * MQTT (optional)
  * Partner API polling
  * CSV upload (optional)
* Validate and normalize the data
* Transform it into standardized Kafka event format
* Publish to Kafka topics
* Handle auth, rate limiting, logging, and monitoring

---

## 📊 Architecture Overview

```plaintext
   ┌────────────────────────────────────┐
   │  External Data Sources     │
   │ (App, GPS, APIs, MQTT)     │
   └────────────────────────────────────┘
                │
                ▼
     ┌────────────────────────────────────┐
     │ Shipment Ingestion Service │
     └────────────────────────────────────┘
                │
 ┌────────────────────────────────────────┐
 ▼                              ▼
Validator                 Kafka Event Publisher
                            │
                            ▼
              ┌────────────────────────────────────┐
              │ Kafka Topic: shipment.*  │
              └────────────────────────────────────┘
```

---

## 🔄 Data Flow

```plaintext
[External System] ─ POST /api/gps/update ─▶ [Ingestion Controller]
       │                                            │
       ▼                                            ▼
 Validate Fields                          Transform to Kafka DTO
       │                                            │
       ▼                                            ▼
Reject (400) or Proceed ───────────────────▶ Publish to Kafka topic
```

---

## 📅 Sample REST API

### POST `/api/gps/update`

```json
Headers:
Authorization: Bearer <JWT>
Content-Type: application/json

Body:
{
  "shipmentId": "SHIP123",
  "vehicleId": "KA-05-AB-1122",
  "lat": 12.9716,
  "lon": 77.5946,
  "timestamp": "2025-06-26T10:30:00Z",
  "temperature": 5.4,
  "shock": false,
  "humidity": 60
}
```

---

## 📦 Kafka Event Format

```json
{
  "eventType": "ShipmentLocationEvent",
  "shipmentId": "SHIP123",
  "vehicleId": "KA-05-AB-1122",
  "coordinates": {
    "lat": 12.9716,
    "lon": 77.5946
  },
  "timestamp": "2025-06-26T10:30:00Z",
  "sensorData": {
    "temperature": 5.4,
    "shock": false,
    "humidity": 60
  },
  "meta": {
    "source": "mobile-app",
    "receivedAt": "2025-06-26T10:30:01Z",
    "ingestionNode": "node-1"
  }
}
```

---

## 💻 Java Code Sample

### DTO

```java
public class ShipmentLocationEvent {
    private String eventType = "ShipmentLocationEvent";
    private String shipmentId;
    private String vehicleId;
    private Coordinates coordinates;
    private Instant timestamp;
    private SensorData sensorData;
    private Meta meta;
    // Getters, Setters, Constructors
}
```

### Kafka Producer

```java
@Autowired
private KafkaTemplate<String, ShipmentLocationEvent> kafkaTemplate;

public void publishLocationEvent(GpsRequest req) {
    ShipmentLocationEvent event = new ShipmentLocationEvent(
        req.getShipmentId(),
        req.getVehicleId(),
        new Coordinates(req.getLat(), req.getLon()),
        req.getTimestamp(),
        new SensorData(req.getTemperature(), req.getShock(), req.getHumidity()),
        new Meta("mobile-app", Instant.now(), "ingestor-1")
    );

    kafkaTemplate.send("shipment.gps.raw", req.getShipmentId(), event);
}
```

---

## 🔐 Security

* OAuth2 + JWT authentication via Spring Security
* Spring Cloud Gateway Rate Limiting (Redis):

  * 100 requests/sec
  * 200 burst capacity

---

## 🧪 Error Handling

* ❌ Invalid Payload → `400 Bad Request`
* ❌ Unauthorized → `401 Unauthorized`
* ❌ Kafka Error → Retry or publish to DLQ: `shipment.dlq`

---

## 📊 Monitoring

* Spring Boot Actuator + Prometheus metrics
* Grafana Dashboards:

  * API call count
  * Kafka publish success/failure
  * Latency and throughput

---

## ⚙️ Scaling Strategy

* Stateless microservice → scalable via Kubernetes
* Kafka acts as buffer → handles burst traffic
* Load balanced via Spring Cloud Gateway

---

## ✅ Summary

> The Shipment Ingestion Service is the **event entrypoint** for all real-time data. It shields downstream systems from raw input chaos and ensures high integrity, speed, and scalability in logistics event processing.
