# Module 18: Microservices Architecture

## 📚 Table of Contents
1. [Overview](#overview)
2. [Monolithic vs Microservices](#monolithic-vs-microservices)
3. [Microservices Principles](#microservices-principles)
4. [Service Communication](#service-communication)
5. [API Gateway Pattern](#api-gateway-pattern)
6. [Service Discovery](#service-discovery)
7. [Circuit Breaker Pattern](#circuit-breaker-pattern)
8. [Distributed Transactions](#distributed-transactions)
9. [Data Management](#data-management)
10. [When to Use Microservices](#when-to-use-microservices)
11. [Best Practices](#best-practices)

---

## Overview

Microservices architecture breaks applications into small, independent services that communicate over well-defined APIs. This module covers microservices fundamentals and patterns.

**Microservices Benefits:**
- **Independent Deployment**: Deploy services independently
- **Technology Diversity**: Use different tech stacks per service
- **Fault Isolation**: Failures in one service don't crash the system
- **Scalability**: Scale services independently
- **Team Autonomy**: Teams can work independently

**Challenges:**
- **Complexity**: More moving parts to manage
- **Distributed Systems**: Network latency, failures
- **Data Consistency**: Harder to maintain
- **Testing**: More complex integration testing

---

## Monolithic vs Microservices

### Monolithic Architecture

**Characteristics:**
- Single codebase
- Single deployment unit
- Shared database
- All features in one application

**Pros:**
- Simple to develop
- Easy to test
- Straightforward deployment
- Good for small teams

**Cons:**
- Tightly coupled
- Hard to scale specific parts
- Technology lock-in
- Slow deployment cycles

### Microservices Architecture

**Characteristics:**
- Multiple independent services
- Independent deployment
- Each service has its own database
- Services communicate via APIs

**Pros:**
- Independent scaling
- Technology flexibility
- Fault isolation
- Team autonomy

**Cons:**
- Distributed system complexity
- Network latency
- Data consistency challenges
- Operational overhead

### When to Choose Each

**Choose Monolithic if:**
- Small team (< 10 developers)
- Simple application
- Rapid prototyping
- Limited scaling needs

**Choose Microservices if:**
- Large team
- Complex domain
- Need independent scaling
- Multiple teams working in parallel

---

## Microservices Principles

### 1. Single Responsibility

Each service should have one clear purpose.

**Example:**
- **User Service**: User management only
- **Product Service**: Product catalog only
- **Order Service**: Order processing only

### 2. Autonomous Services

Services should be independently deployable and scalable.

**Autonomy Includes:**
- Own database
- Own deployment pipeline
- Own versioning
- Own team

### 3. Decentralized Data Management

Each service manages its own data.

**Benefits:**
- Data isolation
- Independent scaling
- Technology choice per service

**Challenges:**
- Data consistency
- Distributed transactions

### 4. Communication via APIs

Services communicate through well-defined APIs.

**Communication Methods:**
- REST APIs (synchronous)
- Message Queues (asynchronous)
- gRPC (high performance)

---

## Service Communication

### Synchronous Communication (REST)

**HTTP/REST:**
- Request-response pattern
- Simple and widely understood
- Easy to debug

**Example:**
```typescript
// Order Service calling User Service
@Injectable()
export class OrderService {
  constructor(
    private httpService: HttpService,
  ) {}

  async createOrder(orderData: CreateOrderDto, userId: number) {
    // Synchronous call to User Service
    const user = await this.httpService
      .get(`http://user-service/users/${userId}`)
      .toPromise();

    if (!user) {
      throw new NotFoundException('User not found');
    }

    // Create order
    return this.ordersRepository.save({
      ...orderData,
      userId,
    });
  }
}
```

**Pros:**
- Simple
- Immediate response
- Easy to understand

**Cons:**
- Tight coupling
- Blocking calls
- Cascading failures

### Asynchronous Communication (Message Queue)

**Message Queues:**
- Publish-subscribe pattern
- Decoupled services
- Better fault tolerance

**Example with RabbitMQ:**
```typescript
// Order Service publishes event
@Injectable()
export class OrderService {
  constructor(
    @InjectQueue('order-events') private orderQueue: Queue,
  ) {}

  async createOrder(orderData: CreateOrderDto) {
    const order = await this.ordersRepository.save(orderData);

    // Publish event asynchronously
    await this.orderQueue.add('order-created', {
      orderId: order.id,
      userId: order.userId,
      total: order.total,
    });

    return order;
  }
}

// Email Service subscribes to event
@Processor('order-events')
export class EmailProcessor {
  @Process('order-created')
  async handleOrderCreated(job: Job) {
    const { orderId, userId, total } = job.data;
    
    // Send order confirmation email
    await this.emailService.sendOrderConfirmation(userId, orderId);
  }
}
```

**Pros:**
- Decoupled
- Fault tolerant
- Scalable

**Cons:**
- Eventual consistency
- Message ordering
- Complexity

### gRPC Communication

**gRPC:**
- High performance RPC
- Protocol buffers
- Streaming support

**When to Use:**
- High performance needs
- Internal service communication
- Streaming data

---

## API Gateway Pattern

An API Gateway is a single entry point for all client requests.

### Why API Gateway?

**Benefits:**
- **Single Entry Point**: Clients interact with one endpoint
- **Request Routing**: Route requests to appropriate services
- **Authentication**: Centralized auth
- **Rate Limiting**: Protect services
- **Load Balancing**: Distribute load
- **Request Transformation**: Adapt requests/responses

### API Gateway Implementation

```typescript
// api-gateway.service.ts
@Injectable()
export class ApiGatewayService {
  constructor(
    private httpService: HttpService,
  ) {}

  async routeRequest(service: string, path: string, method: string, data?: any) {
    const serviceUrl = this.getServiceUrl(service);
    
    const config = {
      method,
      url: `${serviceUrl}${path}`,
      data,
    };

    try {
      const response = await this.httpService.axiosRef(config);
      return response.data;
    } catch (error) {
      if (error.response) {
        throw new HttpException(
          error.response.data,
          error.response.status,
        );
      }
      throw new InternalServerErrorException('Service unavailable');
    }
  }

  private getServiceUrl(service: string): string {
    const services = {
      'users': process.env.USER_SERVICE_URL,
      'products': process.env.PRODUCT_SERVICE_URL,
      'orders': process.env.ORDER_SERVICE_URL,
    };

    return services[service] || '';
  }
}

// api-gateway.controller.ts
@Controller('api')
export class ApiGatewayController {
  constructor(private apiGatewayService: ApiGatewayService) {}

  @Get('users/:id')
  async getUser(@Param('id') id: string) {
    return this.apiGatewayService.routeRequest('users', `/users/${id}`, 'GET');
  }

  @Post('orders')
  async createOrder(@Body() createOrderDto: CreateOrderDto) {
    return this.apiGatewayService.routeRequest(
      'orders',
      '/orders',
      'POST',
      createOrderDto,
    );
  }
}
```

---

## Service Discovery

Service Discovery helps services find and communicate with each other.

### Client-Side Discovery

Client queries service registry to find service instances.

```typescript
@Injectable()
export class ServiceDiscovery {
  private serviceRegistry: Map<string, string[]> = new Map();

  async discoverService(serviceName: string): Promise<string> {
    const instances = this.serviceRegistry.get(serviceName);
    
    if (!instances || instances.length === 0) {
      throw new ServiceUnavailableException(`Service ${serviceName} not found`);
    }

    // Simple round-robin load balancing
    const randomInstance = instances[Math.floor(Math.random() * instances.length)];
    return randomInstance;
  }

  registerService(serviceName: string, instanceUrl: string) {
    const instances = this.serviceRegistry.get(serviceName) || [];
    instances.push(instanceUrl);
    this.serviceRegistry.set(serviceName, instances);
  }
}
```

### Server-Side Discovery

Load balancer queries service registry and routes requests.

---

## Circuit Breaker Pattern

Circuit Breaker prevents cascading failures by stopping requests to failing services.

### Implementation

```typescript
// circuit-breaker.service.ts
@Injectable()
export class CircuitBreakerService {
  private circuitStates: Map<string, CircuitState> = new Map();

  async execute<T>(
    serviceName: string,
    operation: () => Promise<T>,
  ): Promise<T> {
    const state = this.circuitStates.get(serviceName) || {
      state: 'CLOSED',
      failures: 0,
      lastFailureTime: null,
    };

    // Check if circuit is open
    if (state.state === 'OPEN') {
      // Check if timeout period has passed
      if (Date.now() - state.lastFailureTime < 60000) {
        throw new ServiceUnavailableException(
          `Circuit breaker is OPEN for ${serviceName}`,
        );
      }
      // Try to reset
      state.state = 'HALF_OPEN';
    }

    try {
      const result = await operation();
      
      // Success: Reset circuit
      if (state.state === 'HALF_OPEN') {
        state.state = 'CLOSED';
        state.failures = 0;
      }
      
      return result;
    } catch (error) {
      state.failures++;
      state.lastFailureTime = Date.now();

      // Open circuit after threshold
      if (state.failures >= 5) {
        state.state = 'OPEN';
      }

      this.circuitStates.set(serviceName, state);
      throw error;
    }
  }
}

// Usage
@Injectable()
export class OrderService {
  constructor(
    private circuitBreaker: CircuitBreakerService,
    private httpService: HttpService,
  ) {}

  async getUser(userId: number) {
    return this.circuitBreaker.execute('user-service', async () => {
      const response = await this.httpService
        .get(`http://user-service/users/${userId}`)
        .toPromise();
      return response.data;
    });
  }
}
```

**Circuit States:**
- **CLOSED**: Normal operation
- **OPEN**: Blocking requests (service failing)
- **HALF_OPEN**: Testing if service recovered

---

## Distributed Transactions

In microservices, maintaining ACID transactions across services is challenging.

### Saga Pattern

Saga coordinates distributed transactions using local transactions.

**Choreography (Event-driven):**
- Services publish events
- Other services react to events
- No central coordinator

**Orchestration (Centralized):**
- Orchestrator coordinates steps
- Centralized control
- Easier to understand

### Example: Order Saga

```typescript
// Order Orchestrator
@Injectable()
export class OrderOrchestrator {
  async createOrder(orderData: CreateOrderDto) {
    try {
      // Step 1: Reserve inventory
      await this.productService.reserveInventory(orderData.items);
      
      // Step 2: Charge payment
      await this.paymentService.charge(orderData.paymentInfo);
      
      // Step 3: Create order
      const order = await this.orderService.create(orderData);
      
      // Step 4: Send confirmation
      await this.notificationService.sendConfirmation(order.id);
      
      return order;
    } catch (error) {
      // Compensate: Rollback previous steps
      await this.compensate(orderData);
      throw error;
    }
  }

  private async compensate(orderData: CreateOrderDto) {
    // Reverse inventory reservation
    await this.productService.releaseInventory(orderData.items);
    
    // Refund payment
    await this.paymentService.refund(orderData.paymentInfo);
  }
}
```

---

## Data Management

### Database per Service

Each service has its own database:

**Benefits:**
- Data isolation
- Technology choice
- Independent scaling

**Challenges:**
- Data consistency
- Cross-service queries

### Shared Database Anti-pattern

**Don't share databases** - breaks service boundaries.

### Data Synchronization

Use events to synchronize data:

```typescript
// User Service publishes user-updated event
@EventPattern('user-updated')
handleUserUpdated(data: { userId: number; email: string }) {
  // Update local cache/copy
  this.userCache.set(data.userId, data);
}
```

---

## When to Use Microservices

### Good Candidates

- Large development teams
- Complex business domains
- Different scaling needs
- Need for technology diversity
- Independent deployment needs

### Not Good Candidates

- Small applications
- Tightly coupled features
- Small teams
- Simple domains
- High transaction consistency needs

---

## Best Practices

### 1. Start Monolithic, Evolve

Begin with monolith, extract services as needed.

### 2. Domain-Driven Design

Use DDD to identify service boundaries.

### 3. API Versioning

Version APIs to support evolution.

### 4. Implement Circuit Breakers

Protect against cascading failures.

### 5. Use Message Queues

For async communication when appropriate.

### 6. Centralized Logging

Aggregate logs from all services.

### 7. Distributed Tracing

Track requests across services.

### 8. Health Checks

Monitor service health continuously.

---

## Next Steps

✅ Understand microservices principles  
✅ Know when to use microservices  
✅ Understand service communication patterns  
✅ Implement API Gateway  
✅ Use circuit breakers  
✅ Move to Module 19: NestJS Microservices  

---

*You now understand microservices architecture! 🏗️*

