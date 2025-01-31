**# System Design Document: Scalable Microservice-based E-commerce Application**

## **1. Overview**
This document outlines the architecture of a scalable microservice-based e-commerce application. The system consists of independent services that handle user authentication, product management, and order processing.

## **2. Architecture Design**
The system follows a microservices architecture, where each service operates independently and communicates via RESTful APIs or messaging queues.

### **2.1 High-Level Architecture**
- **User Service** → Manages user registration, authentication, and authorization.
- **Product Service** → Handles product catalog management.
- **Order Service** → Manages order placement, payments, and transactions.
- **API Gateway** → Centralized entry point for handling requests.
- **Message Queue (RabbitMQ/Kafka)** → Ensures asynchronous processing for tasks like email notifications.
- **Database Layer (MySQL)** → Stores persistent data.
- **Caching Layer (Redis)** → Speeds up frequent queries.
- **Logging & Monitoring (Prometheus, Grafana, Winston)** → Observability.

## **3. Diagrams**

### **3.1 Component Diagram**
This shows how the User Service, Product Service, and Order Service interact with external systems like the database, API Gateway, and message queue.
```bash
graph TD;
    A[User] -->|Registers/Login| B(User Service)
    A -->|Views Products| C(Product Service)
    A -->|Places Order| D(Order Service)
    
    B -->|JWT Authentication| E[Auth System]
    B -->|Stores Data| F[(MySQL Database)]
    
    C -->|Manages Product Listings| F
    D -->|Manages Orders| F
    D -->|Sends Notifications| G[Message Queue (RabbitMQ/Kafka)]
    
    G -->|Triggers Emails| H[Email Service]
```

### **3.2 Sequence Diagram**
This shows the step-by-step process of a user registering on the platform.
```bash
sequenceDiagram
    participant User
    participant API Gateway
    participant UserService
    participant Database
    participant MessageQueue
    participant EmailService

    User->>API Gateway: POST /register
    API Gateway->>UserService: Forward request
    UserService->>Database: Store user details
    Database-->>UserService: Confirmation
    UserService->>MessageQueue: Publish "User Registered" event
    MessageQueue->>EmailService: Consume event & send email
    EmailService-->>User: "Welcome" email sent
    UserService-->>API Gateway: Return success response
    API Gateway-->>User: Registration successful
```
Example: **User Registration Flow**
1. User sends a `POST /register` request.
2. User Service validates input and stores user in the database.
3. A message is sent to RabbitMQ to trigger a welcome email.
4. Email Service processes the message and sends an email.
5. Response is sent back to the user.

### **3.3 Deployment Diagram**
This shows how services are deployed using Docker and Kubernetes.
```bash
graph TD;
    subgraph Kubernetes Cluster
        subgraph API Layer
            A[API Gateway]
        end
        subgraph Services
            B[User Service]
            C[Product Service]
            D[Order Service]
        end
        subgraph Infrastructure
            E[(MySQL Database)]
            F[Redis Cache]
            G[Message Queue (RabbitMQ)]
        end
    end
```


## **4. Service Descriptions**

### **4.1 User Service**
- **Responsibilities:** User registration, authentication, JWT-based authorization.
- **Endpoints:**
  - `POST /register` → Registers a new user.
  - `POST /login` → Authenticates a user.
  - `GET /users/:id` → Retrieves user details.
- **Database:** Stores user data in MySQL.
- **Security:** Password hashing with bcrypt, JWT for authentication.

### **4.2 Product Service**
- **Responsibilities:** Manages product listings.
- **Endpoints:**
  - `POST /products` → Adds a new product.
  - `GET /products/:id` → Fetches product details.
  - `PUT /products/:id` → Updates product details.
  - `DELETE /products/:id` → Deletes a product.
- **Database:** Stores product information in MySQL.

### **4.3 Order Service**
- **Responsibilities:** Manages orders, payments, and transactions.
- **Endpoints:**
  - `POST /orders` → Places an order.
  - `GET /orders/:id` → Fetches order details.
  - `PUT /orders/:id/cancel` → Cancels an order.
- **Database:** Stores order records in MySQL.

## **5. Communication Between Services**
- **Synchronous Communication:** REST APIs for direct service-to-service calls.

Great choice! **REST APIs** are easier to implement and integrate, especially for a microservice-based system like this.

---

## **🔗 Service Communication (REST API)**
Each microservice will expose **RESTful endpoints** to communicate with other services.

### **1️⃣ API Gateway**
- Acts as a single entry point for all client requests.
- Routes requests to the appropriate microservices.
- Can handle authentication and rate limiting.

**Example Routes:**
```
POST /api/users/register  → User Service
POST /api/users/login     → User Service
GET  /api/products/:id    → Product Service
POST /api/orders          → Order Service
```

---

### **2️⃣ Inter-Service Communication**
Since each service runs independently, they will **communicate via HTTP REST APIs**.

| **Service**  | **Consumer** | **Method** | **Endpoint** |
|-------------|-------------|------------|--------------|
| User Service | Product Service | `GET` | `/api/users/:id` |
| Order Service | Product Service | `GET` | `/api/products/:id` |
| Order Service | User Service | `GET` | `/api/users/:id` |

---











- **Asynchronous Communication:** RabbitMQ/Kafka for handling events (e.g., sending email notifications after registration).

## **6. Data Consistency & Fault Tolerance**
### **6.1 Data Consistency**
- Transactions use **ACID** properties.
- **Two-Phase Commit (2PC)** can be used for distributed transactions.
- **Eventual Consistency** for async operations.

### **6.2 Fault Tolerance**
### **1️⃣ Handling Failures & Timeouts**
To **prevent cascading failures**, we'll use:
- **Retries & Circuit Breaker** (e.g., **axios-retry** in NestJS)
- **Timeouts** (Set request timeouts to avoid long waits)
- **Fallback Mechanism** (e.g., If Product Service is down, return cached data)

Example **circuit breaker** pattern:
```ts
import axios from 'axios';
import CircuitBreaker from 'opossum';

const options = {
  timeout: 3000, // 3s timeout
  errorThresholdPercentage: 50, // Trip breaker if 50% requests fail
  resetTimeout: 5000 // Try again after 5s
};

const breaker = new CircuitBreaker(() => axios.get('http://product-service/api/products/1'), options);

breaker.fallback(() => ({ message: "Product service unavailable, please try again later" }));

breaker.fire()
  .then(console.log)
  .catch(console.error);
```

---

### **2️⃣ Ensuring Security**
- **JWT Authentication:**  
  - Users receive a **JWT token** after login.
  - Every API call must include `Authorization: Bearer <token>`.
  - Services verify tokens before processing requests.

- **Rate Limiting:**  
  - Prevent abuse by limiting API requests per user.
  - Example using `express-rate-limit`:
    ```ts
    import rateLimit from 'express-rate-limit';
    const limiter = rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100 // limit each IP to 100 requests per windowMs
    });
    app.use(limiter);
    ```

---
- **Retry Mechanism:** Retries failed API calls.
- **Circuit Breaker Pattern:** Prevents repeated failures.
- **Database Replication:** Ensures availability of data.

## **7. Scalability**
- **Horizontal Scaling:** Each microservice runs in multiple instances behind a load balancer.
- **Database Scaling:** Read replicas for high availability.
- **Caching:** Redis to reduce database queries.

## **8. Conclusion**
This design ensures a scalable, fault-tolerant, and efficient e-commerce system by leveraging microservices, messaging queues, and robust authentication mechanisms. The system is designed for easy maintainability and extensibility.

