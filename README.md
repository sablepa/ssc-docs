# ssc-docs
Swift Supply Chain Docs

Supply Chain Platform: System Design & Roadmap
This document outlines the proposed architecture, design, and roadmap for a microservices-based supply chain platform catering to small to mid-sized businesses across various sectors like furniture, fitness, and electronics.

1. Project Vision & Goals
Core Functionality: Enable store front offices to manage customer profiles, create orders, handle shipping, generate receipts, and track order status.

Back Office Capabilities: Provide tools for inventory management, stock control, and order processing from various vendors.

Scalability & Reliability: Design a system that can scale with business growth and ensure data consistency, especially for inventory in a distributed environment.

Technology Stack: Leverage Spring Boot for microservices, Spring Security, centralized configuration, Redis for caching/session management, AWS services (SQS, RDS/DynamoDB, EKS, API Gateway), Docker, and Kubernetes.

2. System Architecture
2.1. High-Level Overview
The system will be a collection of independently deployable microservices communicating via REST APIs and asynchronous messaging (AWS SQS). An API Gateway will serve as the single entry point for all client requests (Store Front Office UI, Back Office UI).

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
        ServiceRegistry[Service Registry (e.g., Eureka/Consul)]
        AuthService[Authentication & Authorization Service (Spring Security)]
    end

    subgraph Data Stores & Messaging
        UserDB[(User DB - RDS)]
        ProductDB[(Product DB - RDS)]
        InventoryDB[(Inventory DB - RDS/DynamoDB)]
        OrderDB[(Order DB - RDS)]
        RedisCache[Redis Cache]
        SQSQueue[AWS SQS]
    end

    StoreFrontUI --> APIGateway
    BackOfficeUI --> APIGateway

    APIGateway --> AuthService
    APIGateway --> UserService
    APIGateway --> ProductService
    APIGateway --> InventoryService
    APIGateway --> OrderService
    APIGateway --> ShippingService
    APIGateway --> NotificationService

    UserService --> UserDB
    ProductService --> ProductDB
    InventoryService --> InventoryDB
    OrderService --> OrderDB
    OrderService --> InventoryService
    OrderService --> ShippingService
    OrderService --> NotificationService

    InventoryService -.-> SQSQueue  # For inventory update events
    OrderService -.-> SQSQueue      # For order events (e.g., order created, status change)
    ShippingService -.-> SQSQueue   # For shipment events

    NotificationService -.-> SQSQueue # Consumes events for notifications

    UserService -.-> RedisCache
    ProductService -.-> RedisCache
    AuthService -.-> RedisCache # Session Management

    UserService -.-> ConfigServer
    ProductService -.-> ConfigServer
    InventoryService -.-> ConfigServer
    OrderService -.-> ConfigServer
    NotificationService -.-> ConfigServer
    ShippingService -.-> ConfigServer
    AuthService -.-> ConfigServer

    UserService -.-> ServiceRegistry
    ProductService -.-> ServiceRegistry
    InventoryService -.-> ServiceRegistry
    OrderService -.-> ServiceRegistry
    NotificationService -.-> ServiceRegistry
    ShippingService -.-> ServiceRegistry
    AuthService -.-> ServiceRegistry

2.2. Technology Stack Summary
Backend: Java, Spring Boot, Spring Cloud (Config, Gateway/Service Registry), Spring Security

Database: AWS RDS (PostgreSQL/MySQL) for transactional data. Potentially AWS DynamoDB for specific high-throughput, flexible schema needs (e.g., audit logs, certain inventory aspects if hyper-scaling is needed).

Caching: Redis (AWS ElastiCache)

Messaging: AWS SQS

Containerization & Orchestration: Docker, Kubernetes (AWS EKS)

API Gateway: AWS API Gateway

Configuration Management: Spring Cloud Config Server

Logging & Monitoring: ELK Stack (Elasticsearch, Logstash, Kibana) or AWS CloudWatch.

CI/CD: Jenkins, GitLab CI, or AWS CodePipeline.

3. Microservice Details
User Service:

Responsibilities: Manages user profiles (customers, store staff), addresses, authentication tokens (works with AuthService).

Data: User accounts, roles, permissions, addresses.

Key APIs: CRUD for users, addresses, login, registration.

Product Service:

Responsibilities: Manages product catalogs for different businesses. Includes product details, categories, pricing specific to businesses.

Data: Products, SKUs, categories, business-specific pricing.

Key APIs: CRUD for products, categories, search/filter products.

Inventory Service:

Responsibilities: Manages stock levels across potentially multiple warehouses. Source of truth for inventory. Handles stock adjustments, reservations.

Data: Inventory items (product, warehouse, quantity), stock movements.

Key APIs: Get stock level, reserve stock, confirm stock deduction, adjust stock.

Stale Inventory Mitigation:

Publishes inventory update events to SQS (e.g., StockUpdatedEvent).

Uses optimistic locking for concurrent updates.

May implement a reservation system where stock is held for a short period during order processing.

Periodic reconciliation jobs.

Order Service:

Responsibilities: Handles order creation, processing, status updates, and history. Coordinates with Inventory, Shipping, and Notification services.

Data: Orders, order items, payment details (references), order status history.

Key APIs: Create order, get order details, update order status, list orders.

SQS Usage:

Publishes OrderCreatedEvent to SQS. Inventory Service subscribes to deduct stock. Notification Service subscribes to send order confirmation.

Publishes OrderStatusChangedEvent (e.g., Dispatched, Delivered).

Shipping Service:

Responsibilities: Manages shipping information, generates tracking numbers (can integrate with third-party shipping APIs), updates shipment status. Generates receipts for dispatch and delivery.

Data: Shipments, tracking info, carrier details, delivery addresses.

Key APIs: Create shipment, get shipment status, update tracking.

SQS Usage:

Publishes ShipmentDispatchedEvent, ShipmentDeliveredEvent. Notification Service subscribes.

Notification Service:

Responsibilities: Sends notifications (email, SMS - potentially via AWS SNS) for various events like order confirmation, dispatch, delivery.

Data: Notification templates, logs.

SQS Usage: Subscribes to events from Order Service, Shipping Service, and Inventory Service (e.g., low stock alerts) to trigger notifications.

Authentication Service (Leveraging Spring Security):

Responsibilities: Handles user authentication (login) and token generation (JWT). Manages authorization rules.

Integration: Works closely with API Gateway and other microservices to secure endpoints.

Redis Usage: Stores session information or JWT revocation lists.

API Gateway (AWS API Gateway):

Responsibilities: Single entry point, request routing, rate limiting, authentication/authorization enforcement (delegating to AuthService), SSL termination, request/response transformation.

Configuration Service (Spring Cloud Config Server):

Responsibilities: Centralized management of configuration for all microservices. Configurations stored in a Git repository.

4. Data Entities (Conceptual Model)
Business: (BusinessID, Name, Type, ContactInfo)

StoreFront: (StoreID, BusinessID, Name, Location, StaffUserIDs)

User: (UserID, Username, PasswordHash, Email, FirstName, LastName, Role, IsActive, CreatedAt, UpdatedAt)

Address: (AddressID, UserID, Street, City, State, ZipCode, Country, Type (Shipping/Billing), IsDefault)

Product: (ProductID, BusinessID, SKU, Name, Description, Category, UnitPrice, AttributesJSON, IsActive, CreatedAt, UpdatedAt)

Warehouse: (WarehouseID, Name, Location, ContactInfo)

InventoryItem: (InventoryItemID, ProductID, WarehouseID, QuantityOnHand, QuantityReserved, QuantityAvailable, LastStockUpdate, Version) (Version for optimistic locking)

Order: (OrderID, UserID, StoreID, OrderDate, Status (Pending, Confirmed, Processing, Dispatched, Delivered, Cancelled), TotalAmount, Currency, ShippingAddressID, BillingAddressID, CreatedAt, UpdatedAt)

OrderItem: (OrderItemID, OrderID, ProductID, Quantity, UnitPriceAtOrder, TotalPrice)

Shipment: (ShipmentID, OrderID, CarrierName, TrackingNumber, DispatchDate, EstimatedDeliveryDate, ActualDeliveryDate, Status (PendingDispatch, InTransit, Delivered, FailedDelivery), SourceWarehouseID, ShippingCost)

Receipt: (ReceiptID, OrderID, ShipmentID (optional), Stage (Ordered, Dispatched, Delivered), GeneratedAt, Content (e.g., PDF link or structured data))

Vendor: (VendorID, Name, ContactInfo, ProductsSuppliedJSON)

StockMovement: (MovementID, ProductID, WarehouseID, Type (Inbound, Outbound, Adjustment), Quantity, Reason, Timestamp, OrderID (optional), UserID (staff who made adjustment))

5. Key Business Flows
Store Front Creates User Profile:

Store staff accesses Store Front UI.

Enters customer details -> POST /users to User Service via API Gateway.

Adds addresses -> POST /users/{userId}/addresses.

Store Front Creates Order:

Staff selects customer, adds products to cart.

UI calculates total -> POST /orders to Order Service.

Order Service:

Validates request.

(Optional) Creates a temporary stock reservation with Inventory Service.

Saves order in Pending state.

Publishes OrderCreatedEvent to SQS.

Inventory Service (consumes OrderCreatedEvent):

Decrements stock for ordered items (using optimistic locking).

If stock unavailable, publishes StockUnavailableEvent (Order Service might listen to update order status or initiate backorder flow).

Publishes StockUpdatedEvent.

Notification Service (consumes OrderCreatedEvent):

Sends "Order Confirmed" receipt/notification to the customer.

Order Service updates order status to Confirmed.

Inventory Management (Back Office):

Staff views inventory levels (GET /inventory?productId=...&warehouseId=...).

Staff updates stock (e.g., new shipment from vendor) -> PUT /inventory/items/{itemId} or POST /inventory/adjustments.

Inventory Service updates stock and publishes StockUpdatedEvent.

Order Fulfillment & Tracking:

Dispatch:

Back office staff marks order for dispatch.

Triggers Shipping Service -> POST /shipments (associating with OrderID).

Shipping Service generates tracking, updates status to InTransit.

Publishes ShipmentDispatchedEvent to SQS.

Order Service consumes event, updates order status to Dispatched.

Notification Service consumes event, sends "Order Dispatched" receipt/notification with tracking info.

Delivery:

Shipping Service (receives update from carrier or manual update) updates shipment status to Delivered.

Publishes ShipmentDeliveredEvent.

Order Service consumes, updates order status to Delivered.

Notification Service consumes, sends "Order Delivered" receipt/notification.

Receipt Generation:

Receipts are generated at different stages (Ordered, Dispatched, Delivered).

Notification Service can be responsible for compiling receipt data (from Order, Product, User services) and generating it (e.g., PDF or structured email content) upon consuming relevant events.

6. Technology Integration Details
Redis Usage:

Caching: Product details, user profiles, frequently accessed configurations.

Example: Cache product data in Product Service to reduce DB load.

Session Management: Store user session data for API Gateway/AuthService.

Rate Limiting: Implement API rate limits at the API Gateway level, potentially using Redis for counters.

Distributed Locks (Use with Caution): For short-lived critical sections if optimistic locking is insufficient in Inventory Service, though often database-level optimistic locking or a dedicated distributed lock manager (like ZooKeeper/etcd if already in use) is preferred for robustness. For inventory, event-driven approaches with compensation are generally better.

Leaderboards/Real-time Stats (Future): If you need to show top-selling products, etc.

AWS SQS Usage:

Decoupling Services:

Order Service publishes OrderCreatedEvent -> Inventory Service & Notification Service subscribe.

Inventory Service publishes StockUpdatedEvent -> Potentially other services (e.g., a replenishment service) subscribe.

Shipping Service publishes ShipmentStatusChangedEvent -> Order Service & Notification Service subscribe.

Asynchronous Task Processing:

Generating complex reports.

Sending bulk notifications.

Processing batch inventory updates.

Buffering & Smoothing Load:

During peak order times, SQS can buffer requests to the Inventory Service, preventing it from being overwhelmed.

Dead Letter Queues (DLQs): Configure DLQs for SQS queues to handle messages that cannot be processed successfully, allowing for investigation and reprocessing.

Spring Security:

Implement OAuth2 or JWT-based authentication.

Central Authentication Service (or integrated within API Gateway/User Service initially).

Role-based access control (RBAC) for different user types (customer, store staff, back-office admin).

Secure inter-service communication (e.g., using service accounts or token relay).

Docker & Kubernetes (AWS EKS):

Each microservice will be packaged as a Docker container.

Kubernetes will be used for deployment, scaling (Horizontal Pod Autoscaler), service discovery, load balancing, and self-healing of services on AWS EKS.

7. Addressing Stale Inventory in Distributed Systems
This is a critical challenge. A multi-pronged approach:

Single Source of Truth: The Inventory Service is the only service that directly modifies inventory quantities.

Eventual Consistency: Accept that other services might have slightly delayed views of inventory. The UI should reflect this or manage expectations.

Optimistic Locking: Use a version number in your InventoryItem entity. When updating, check if the version matches. If not, the update fails, and the operation needs to be retried or handled.

// In InventoryItem entity
// @Version
// private Long version;

Asynchronous Updates via SQS: When an order is placed, the Order Service can publish an OrderPendingConfirmationEvent. The Inventory Service consumes this, attempts to allocate stock. If successful, it publishes StockAllocatedEvent; if not, StockAllocationFailedEvent. The Order Service listens for these to confirm or reject the order.

Compensating Transactions (Saga Pattern): For complex operations spanning multiple services (e.g., order creation involving payment, inventory, notification), if one step fails, subsequent steps must be rolled back. Sagas can be implemented using choreography (events) or orchestration (a central coordinator).

Example (Choreography): Order Service creates order (pending) -> emits OrderCreated. Inventory Service listens, reserves stock -> emits StockReserved. Payment Service listens, processes payment -> emits PaymentProcessed. If StockReservationFailed or PaymentFailed, compensating events are emitted to roll back.

Reservation System: Inventory Service can offer an API to temporarily "reserve" stock for a short period (e.g., 5-10 minutes) while a customer completes checkout. If the order isn't confirmed within that window, the reservation expires.

Real-time Communication (Optional, for UI): For UIs displaying stock, consider WebSockets or Server-Sent Events (SSE) fed by the Inventory Service to provide near real-time updates, but this adds complexity. A more common approach is to re-fetch or rely on slightly delayed data with clear indicators.

Atomic Operations where Possible: If a business has only one front office and one warehouse, some operations can be simpler. The design should cater to the more complex distributed scenario.

Business Logic for Discrepancies: Define how to handle situations where an order is placed for an out-of-stock item due to staleness (e.g., backorder, offer alternatives, cancel item).

8. API Design Examples (High-Level)
User Service:

POST /api/v1/users (Create user)

GET /api/v1/users/{userId}

PUT /api/v1/users/{userId}

POST /api/v1/users/{userId}/addresses

GET /api/v1/users/{userId}/addresses

POST /api/v1/auth/login

POST /api/v1/auth/register (if self-service registration is allowed)

Product Service:

POST /api/v1/products (Admin only)

GET /api/v1/products?businessId={businessId}&category={category}&searchTerm={term}

GET /api/v1/products/{productId}

Inventory Service:

GET /api/v1/inventory/stock?productId={productId}&warehouseId={warehouseId}

POST /api/v1/inventory/reservations (Reserve stock)

DELETE /api/v1/inventory/reservations/{reservationId} (Cancel reservation)

POST /api/v1/inventory/adjustments (Adjust stock levels, e.g., for received goods, discrepancies)

Order Service:

POST /api/v1/orders (Body: { userId, storeId, items: [{productId, quantity}], shippingAddressId, billingAddressId })

GET /api/v1/orders/{orderId}

GET /api/v1/orders?userId={userId}&status={status}

PUT /api/v1/orders/{orderId}/status (e.g., cancel order)

Shipping Service:

POST /api/v1/shipments (Body: { orderId, carrier, warehouseId }) -> returns tracking info

GET /api/v1/shipments/{shipmentId}

GET /api/v1/shipments/tracking/{trackingNumber}

POST /api/v1/shipments/{shipmentId}/receipts (Generate dispatch/delivery receipt)

Notification Service (Internal or triggered via events):

(Typically consumes SQS messages, may not have many direct external APIs unless for manual trigger/testing)

9. Development Roadmap (Phased Approach)
Phase 1: Core MVP (3-6 Months)

Services:

User Service (CRUD, basic roles)

Product Service (CRUD)

Inventory Service (Single warehouse, basic stock updates, optimistic locking)

Order Service (Order creation, status updates - Pending, Confirmed, Cancelled)

Auth Service (JWT based, basic Spring Security integration)

Infrastructure:

API Gateway (AWS API Gateway basic setup)

Spring Cloud Config Server

Relational Database (AWS RDS)

Basic Docker setup for local development.

Frontends:

Simple Store Front UI: User creation, product listing, order creation.

Simple Back Office UI: Product management, basic inventory view & update, order viewing.

Key Flows: User creation by store staff, order placement, basic inventory deduction.

Phase 2: Enhancements & Integrations (Next 3-4 Months)

Services:

Shipping Service (Basic: create shipment, manual tracking update)

Notification Service (Email for order confirmation using SQS)

Inventory Service: Multi-warehouse support (conceptual), stock reservation API.

Features:

Receipt generation (Order Confirmed - basic PDF/email)

SQS integration for OrderCreatedEvent -> Inventory deduction & Notification.

Enhanced Spring Security (more granular permissions).

Redis for caching (User profiles, Product data).

Infrastructure:

Initial Kubernetes (EKS) setup for staging/QA.

CI/CD pipeline basics.

Phase 3: Scaling, Reliability & Advanced Features (Next 4-6 Months)

Services:

Full SQS integration across services for decoupling (dispatch, delivery events).

Shipping Service: Integration with a test shipping provider API for tracking.

Inventory Service: Advanced strategies for stale inventory (e.g., Saga pattern for order processing if needed, refined reservation timeouts).

Features:

Full receipt lifecycle (Ordered, Dispatched, Delivered).

Comprehensive back-office inventory management (adjustments, movements, vendor PO tracking - conceptual).

Advanced order tracking visible to store staff/customer.

User address management.

Infrastructure:

Full deployment to Kubernetes on AWS EKS for production.

Horizontal Pod Autoscaling.

Centralized logging and monitoring (CloudWatch/ELK).

Robust backup and recovery for databases.

Phase 4: Optimization & Future Growth (Ongoing)

Features:

Support for more diverse business types and their specific needs.

Advanced analytics and reporting dashboards.

Mobile app for store staff or customers.

Third-party integrations (accounting, payment gateways if not done earlier).

Vendor portal for direct inventory updates by vendors.

Optimization:

Performance tuning of services and databases.

Cost optimization of AWS resources.

Security hardening and regular audits.

10. Security Considerations
Authentication: Strong authentication mechanisms (OAuth 2.0 / JWT).

Authorization: Role-Based Access Control (RBAC) at API Gateway and service levels.

Data Encryption: TLS for data in transit. Encryption at rest for databases and S3 (if used).

Input Validation: Rigorous validation at API Gateway and service layers to prevent injection attacks (SQLi, XSS).

Secrets Management: Use AWS Secrets Manager or HashiCorp Vault for database credentials, API keys.

Secure Inter-service Communication: mTLS or token-based auth between services.

Regular Security Audits & Penetration Testing.

OWASP Top 10: Adhere to best practices.

This detailed plan should provide a strong starting point. Remember that this is a high-level design, and many details will be refined during the development process. Agile methodologies will be beneficial to adapt to changing requirements and learnings.
