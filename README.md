# ssc-docs
Swift Supply Chain Docs

# Supply Chain Platform: System Design & Roadmap

## 1. Project Vision & Goals
**Core Functionality**:  
Enable store front offices to manage:
- Customer profiles
- Order creation
- Shipping handling
- Receipt generation
- Order status tracking  

**Back Office Capabilities**:
- Inventory management
- Stock control
- Vendor order processing  

**Scalability & Reliability**:
- Scale with business growth
- Ensure data consistency (especially inventory in distributed systems)  

**Technology Stack**:
- **Microservices**: Spring Boot
- **Security**: Spring Security
- **Caching**: Redis
- **Cloud**: AWS (SQS, RDS/DynamoDB, EKS, API Gateway)
- **Containerization**: Docker & Kubernetes

---

## 2. System Architecture
### 2.1. High-Level Overview

```mermaid
flowchart TD
 subgraph subGraph0["Client Applications"]
        StoreFrontUI["Store Front Office UI"]
        BackOfficeUI["Back Office UI"]
  end
 subgraph subGraph1["API Layer"]
        APIGateway["AWS API Gateway"]
  end
 subgraph subGraph2["Core Microservices"]
        UserService["User Service"]
        ProductService["Product Service"]
        InventoryService["Inventory Service"]
        OrderService["Order Service"]
        NotificationService["Notification Service"]
        ShippingService["Shipping Service"]
  end
 subgraph subGraph3["Supporting Services"]
        ConfigServer["Spring Cloud Config Server"]
        ServiceRegistry["Service Registry"]
        AuthService["Authentication & Authorization Service"]
  end
 subgraph subGraph4["Data Stores & Messaging"]
        UserDB[("User DB - RDS")]
        ProductDB[("Product DB - RDS")]
        InventoryDB[("Inventory DB - RDS/DynamoDB")]
        OrderDB[("Order DB - RDS")]
        RedisCache["Redis Cache"]
        SQSQueue["AWS SQS"]
  end
    StoreFrontUI --> APIGateway
    BackOfficeUI --> APIGateway
    APIGateway --> AuthService & UserService & ProductService & InventoryService & OrderService & ShippingService & NotificationService
    UserService --> UserDB
    ProductService --> ProductDB
    InventoryService --> InventoryDB
    OrderService --> OrderDB & InventoryService & ShippingService & NotificationService
    InventoryService -.-> SQSQueue & ConfigServer & ServiceRegistry
    OrderService -.-> SQSQueue & ConfigServer & ServiceRegistry
    ShippingService -.-> SQSQueue & ConfigServer & ServiceRegistry
    NotificationService -.-> SQSQueue & ConfigServer & ServiceRegistry
    UserService -.-> RedisCache & ConfigServer & ServiceRegistry
    ProductService -.-> RedisCache & ConfigServer & ServiceRegistry
    AuthService -.-> RedisCache & ConfigServer & ServiceRegistry
```

