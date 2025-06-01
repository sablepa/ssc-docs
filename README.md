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
graph TD
    subgraph Client Applications
        StoreFrontUI[Store Front Office UI]
        BackOfficeUI[Back Office UI]
    end

    subgraph API Layer
        APIGateway[AWS API Gateway]
    end

    subgraph Core Microservices
        UserService[User Service]
        ProductService[Product Service]
        InventoryService[Inventory Service]
        OrderService[Order Service]
        NotificationService[Notification Service]
        ShippingService[Shipping Service]
    end

    subgraph Supporting Services
        ConfigServer[Spring Cloud Config Server]
        ServiceRegistry[Service Registry]
        AuthService[Auth Service]
    end

    subgraph Data Stores
        UserDB[(User DB)]
        ProductDB[(Product DB)]
        InventoryDB[(Inventory DB)]
        OrderDB[(Order DB)]
        RedisCache[Redis Cache]
        SQSQueue[AWS SQS]
    end

    StoreFrontUI --> APIGateway
    BackOfficeUI --> APIGateway
    APIGateway --> AuthService
    APIGateway --> Core Microservices
    OrderService --> InventoryService
    OrderService --> ShippingService
    Core Microservices --> Data Stores
    InventoryService -.-> SQSQueue
    OrderService -.-> SQSQueue
```
