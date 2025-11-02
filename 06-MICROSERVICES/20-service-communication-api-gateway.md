# Module 20: Service Communication & API Gateway

## 📚 Table of Contents
1. [Overview](#overview)
2. [API Gateway Pattern](#api-gateway-pattern)
3. [Implementing API Gateway](#implementing-api-gateway)
4. [REST Communication](#rest-communication)
5. [gRPC Communication](#grpc-communication)
6. [Message Queue Communication](#message-queue-communication)
7. [Service Discovery](#service-discovery)
8. [Load Balancing](#load-balancing)
9. [Rate Limiting](#rate-limiting)
10. [Request Aggregation](#request-aggregation)
11. [Best Practices](#best-practices)

---

## Overview

This module covers implementing an API Gateway and different service communication patterns. The API Gateway acts as a single entry point for clients and routes requests to appropriate microservices.

**Topics:**
- API Gateway implementation
- REST service communication
- gRPC inter-service calls
- Message queue integration
- Service discovery
- Load balancing strategies

---

## API Gateway Pattern

### What is an API Gateway?

An API Gateway is a server that acts as a single entry point for all client requests. It handles:
- **Request Routing**: Routes requests to appropriate services
- **Authentication**: Validates tokens, authorizes requests
- **Rate Limiting**: Protects services from overload
- **Load Balancing**: Distributes load across service instances
- **Protocol Translation**: Converts between protocols
- **Request/Response Transformation**: Adapts data formats

### Architecture

```
Client → API Gateway → [User Service, Product Service, Order Service]
```

**Benefits:**
- Single entry point for clients
- Centralized cross-cutting concerns
- Service abstraction
- Protocol flexibility

---

## Implementing API Gateway

### Basic API Gateway

```typescript
// api-gateway.service.ts
import { Injectable, HttpService, HttpException } from '@nestjs/common';

@Injectable()
export class ApiGatewayService {
  // Service URLs (in production, use service discovery)
  private readonly services = {
    users: process.env.USER_SERVICE_URL || 'http://localhost:3001',
    products: process.env.PRODUCT_SERVICE_URL || 'http://localhost:3002',
    orders: process.env.ORDER_SERVICE_URL || 'http://localhost:3003',
  };

  constructor(private httpService: HttpService) {}

  // Route request to appropriate service
  async routeRequest(
    service: string,
    path: string,
    method: string,
    data?: any,
    headers?: Record<string, string>,
  ): Promise<any> {
    const serviceUrl = this.services[service];
    
    if (!serviceUrl) {
      throw new HttpException(`Service ${service} not found`, 404);
    }

    try {
      const config = {
        method: method as any,
        url: `${serviceUrl}${path}`,
        data,
        headers: {
          ...headers,
          // Forward original headers if needed
        },
        timeout: 10000, // 10 second timeout
      };

      const response = await this.httpService.axiosRef(config);
      return response.data;
    } catch (error) {
      // Handle service errors
      if (error.response) {
        // Service returned error
        throw new HttpException(
          error.response.data,
          error.response.status,
        );
      } else if (error.request) {
        // Request made but no response (service down)
        throw new HttpException(
          `Service ${service} is unavailable`,
          503,
        );
      } else {
        // Request setup error
        throw new HttpException(
          'Internal gateway error',
          500,
        );
      }
    }
  }

  // Convenience methods for each service
  async callUserService(path: string, method: string, data?: any) {
    return this.routeRequest('users', path, method, data);
  }

  async callProductService(path: string, method: string, data?: any) {
    return this.routeRequest('products', path, method, data);
  }

  async callOrderService(path: string, method: string, data?: any) {
    return this.routeRequest('orders', path, method, data);
  }
}
```

### API Gateway Controller

```typescript
// api-gateway.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, Headers } from '@nestjs/common';
import { ApiGatewayService } from './api-gateway.service';
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';

@Controller('api')
export class ApiGatewayController {
  constructor(private apiGatewayService: ApiGatewayService) {}

  // User Service Routes
  @Get('users/:id')
  @UseGuards(JwtAuthGuard)
  async getUser(@Param('id') id: string, @Headers() headers: any) {
    // Extract auth token from headers
    const authHeaders = {
      authorization: headers.authorization,
    };
    
    return this.apiGatewayService.callUserService(
      `/users/${id}`,
      'GET',
      null,
      authHeaders,
    );
  }

  @Post('users')
  async createUser(@Body() createUserDto: any) {
    return this.apiGatewayService.callUserService(
      '/users',
      'POST',
      createUserDto,
    );
  }

  // Product Service Routes
  @Get('products')
  async getProducts() {
    return this.apiGatewayService.callProductService('/products', 'GET');
  }

  @Get('products/:id')
  async getProduct(@Param('id') id: string) {
    return this.apiGatewayService.callProductService(`/products/${id}`, 'GET');
  }

  // Order Service Routes
  @Post('orders')
  @UseGuards(JwtAuthGuard)
  async createOrder(@Body() createOrderDto: any, @Headers() headers: any) {
    // Forward user info from JWT to order service
    const authHeaders = {
      authorization: headers.authorization,
    };
    
    return this.apiGatewayService.callOrderService(
      '/orders',
      'POST',
      createOrderDto,
      authHeaders,
    );
  }

  @Get('orders')
  @UseGuards(JwtAuthGuard)
  async getOrders(@Headers() headers: any) {
    const authHeaders = {
      authorization: headers.authorization,
    };
    
    return this.apiGatewayService.callOrderService(
      '/orders',
      'GET',
      null,
      authHeaders,
    );
  }
}
```

---

## REST Communication

### Service-to-Service REST Calls

```typescript
// orders.service.ts (Order Service calling User Service)
import { Injectable, HttpService } from '@nestjs/common';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class OrdersService {
  private readonly userServiceUrl = process.env.USER_SERVICE_URL;

  constructor(private httpService: HttpService) {}

  async createOrder(orderData: CreateOrderDto, userId: number) {
    // Call User Service to validate user
    try {
      const userResponse = await firstValueFrom(
        this.httpService.get(`${this.userServiceUrl}/users/${userId}`),
      );
      
      const user = userResponse.data;
      
      if (!user || !user.isActive) {
        throw new BadRequestException('Invalid user');
      }

      // Create order
      const order = await this.ordersRepository.save({
        ...orderData,
        userId,
      });

      return order;
    } catch (error) {
      if (error.response?.status === 404) {
        throw new NotFoundException('User not found');
      }
      throw new InternalServerErrorException('Failed to validate user');
    }
  }
}
```

### Using Client Module

```typescript
// users-client.module.ts
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';

@Module({
  imports: [
    HttpModule.register({
      baseURL: process.env.USER_SERVICE_URL,
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  exports: [HttpModule],
})
export class UsersClientModule {}

// Usage in Order Service
@Module({
  imports: [UsersClientModule],
})
export class OrdersModule {}
```

---

## gRPC Communication

### gRPC Client Setup

```typescript
// users-client.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { join } from 'path';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USERS_PACKAGE',
        transport: Transport.GRPC,
        options: {
          package: 'users',
          protoPath: join(__dirname, '../protos/users.proto'),
          url: process.env.USER_SERVICE_GRPC_URL || 'localhost:5000',
        },
      },
    ]),
  ],
  exports: [ClientsModule],
})
export class UsersClientModule {}
```

### Using gRPC Client

```typescript
// orders.service.ts
import { Injectable, Inject, OnModuleInit } from '@nestjs/common';
import { ClientGrpc } from '@nestjs/microservices';

interface UsersServiceClient {
  getUser(data: { userId: number }): Observable<any>;
  createUser(data: any): Observable<any>;
}

@Injectable()
export class OrdersService implements OnModuleInit {
  private usersService: UsersServiceClient;

  constructor(@Inject('USERS_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    // Get service stub
    this.usersService = this.client.getService<UsersServiceClient>('UsersService');
  }

  async createOrder(orderData: CreateOrderDto, userId: number) {
    // Call User Service via gRPC
    const userResponse = await firstValueFrom(
      this.usersService.getUser({ userId }),
    );

    if (!userResponse.success || !userResponse.user) {
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

---

## Message Queue Communication

### RabbitMQ Client

```typescript
// orders.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class OrdersService {
  constructor(
    @Inject('USERS_SERVICE') private usersClient: ClientProxy,
    @Inject('PRODUCTS_SERVICE') private productsClient: ClientProxy,
  ) {}

  async createOrder(orderData: CreateOrderDto) {
    // Validate user via message queue
    const user = await firstValueFrom(
      this.usersClient.send({ cmd: 'get_user' }, { userId: orderData.userId }),
    );

    // Check product availability
    const product = await firstValueFrom(
      this.productsClient.send(
        { cmd: 'check_stock' },
        { productId: orderData.productId, quantity: orderData.quantity },
      ),
    );

    if (!product.available) {
      throw new BadRequestException('Product out of stock');
    }

    // Create order
    const order = await this.ordersRepository.save(orderData);

    // Emit events
    this.usersClient.emit('order.created', {
      orderId: order.id,
      userId: order.userId,
    });

    this.productsClient.emit('order.created', {
      orderId: order.id,
      productId: order.productId,
      quantity: order.quantity,
    });

    return order;
  }
}
```

### RabbitMQ Client Module

```typescript
// users-client.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USERS_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: [process.env.RABBITMQ_URL || 'amqp://localhost:5672'],
          queue: 'users_queue',
          queueOptions: {
            durable: true,
          },
        },
      },
    ]),
  ],
  exports: [ClientsModule],
})
export class UsersClientModule {}
```

---

## Service Discovery

### Consul Service Discovery

```typescript
// service-discovery.service.ts
import { Injectable } from '@nestjs/common';
import * as consul from 'consul';

@Injectable()
export class ServiceDiscoveryService {
  private consulClient: consul.Consul;

  constructor() {
    this.consulClient = new consul({
      host: process.env.CONSUL_HOST || 'localhost',
      port: 8500,
    });
  }

  async discoverService(serviceName: string): Promise<string> {
    const services = await this.consulClient.health.service({
      service: serviceName,
      passing: true, // Only healthy services
    });

    if (!services[0] || services[0].length === 0) {
      throw new Error(`Service ${serviceName} not found`);
    }

    // Load balance: select random healthy instance
    const service = services[0][Math.floor(Math.random() * services[0].length)];
    const address = service.Service.Address;
    const port = service.Service.Port;

    return `http://${address}:${port}`;
  }

  async registerService(serviceName: string, port: number) {
    await this.consulClient.agent.service.register({
      name: serviceName,
      port,
      check: {
        http: `http://localhost:${port}/health`,
        interval: '10s',
      },
    });
  }
}
```

---

## Load Balancing

### Round-Robin Load Balancer

```typescript
// load-balancer.service.ts
@Injectable()
export class LoadBalancerService {
  private serviceInstances: Map<string, string[]> = new Map();
  private currentIndex: Map<string, number> = new Map();

  getNextInstance(serviceName: string): string {
    const instances = this.serviceInstances.get(serviceName) || [];
    
    if (instances.length === 0) {
      throw new Error(`No instances available for ${serviceName}`);
    }

    // Round-robin
    const currentIndex = this.currentIndex.get(serviceName) || 0;
    const instance = instances[currentIndex];
    this.currentIndex.set(serviceName, (currentIndex + 1) % instances.length);

    return instance;
  }

  addInstance(serviceName: string, instanceUrl: string) {
    const instances = this.serviceInstances.get(serviceName) || [];
    if (!instances.includes(instanceUrl)) {
      instances.push(instanceUrl);
      this.serviceInstances.set(serviceName, instances);
    }
  }

  removeInstance(serviceName: string, instanceUrl: string) {
    const instances = this.serviceInstances.get(serviceName) || [];
    const filtered = instances.filter(url => url !== instanceUrl);
    this.serviceInstances.set(serviceName, filtered);
  }
}
```

---

## Rate Limiting

### API Gateway Rate Limiting

```typescript
// rate-limiter.service.ts
@Injectable()
export class RateLimiterService {
  private requests: Map<string, number[]> = new Map();
  private readonly limit = 100; // requests
  private readonly window = 60000; // 1 minute

  canProceed(clientId: string): boolean {
    const now = Date.now();
    const clientRequests = this.requests.get(clientId) || [];

    // Remove old requests outside window
    const recentRequests = clientRequests.filter(
      timestamp => now - timestamp < this.window,
    );

    if (recentRequests.length >= this.limit) {
      return false; // Rate limit exceeded
    }

    // Add current request
    recentRequests.push(now);
    this.requests.set(clientId, recentRequests);

    return true;
  }
}

// Usage in Gateway Guard
@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(private rateLimiter: RateLimiterService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const clientId = request.ip || request.headers['x-forwarded-for'];

    if (!this.rateLimiter.canProceed(clientId)) {
      throw new HttpException('Rate limit exceeded', 429);
    }

    return true;
  }
}
```

---

## Request Aggregation

Aggregate multiple service calls into one response.

```typescript
// api-gateway.controller.ts
@Get('dashboard')
@UseGuards(JwtAuthGuard)
async getDashboard(@CurrentUser() user: User) {
  // Fetch data from multiple services in parallel
  const [orders, products, notifications] = await Promise.all([
    this.apiGatewayService.callOrderService(
      `/users/${user.id}/orders`,
      'GET',
    ),
    this.apiGatewayService.callProductService('/products/recommended', 'GET'),
    this.apiGatewayService.callNotificationService(
      `/users/${user.id}/notifications`,
      'GET',
    ),
  ]);

  return {
    orders,
    products,
    notifications,
  };
}
```

---

## Best Practices

### 1. Use Service Discovery

Don't hardcode service URLs; use service discovery.

### 2. Implement Circuit Breakers

Protect against cascading failures.

### 3. Add Timeouts

Always set timeouts for service calls.

### 4. Log All Requests

Log service calls for debugging and monitoring.

### 5. Use Connection Pooling

Reuse HTTP connections.

### 6. Implement Retry Logic

Retry failed requests with exponential backoff.

---

## Next Steps

✅ Understand API Gateway pattern  
✅ Implement service communication  
✅ Use REST, gRPC, and message queues  
✅ Set up service discovery  
✅ Implement load balancing  
✅ Move to Module 21: Distributed Systems Patterns  

---

*You now know how to implement service communication and API Gateway! 🌐*

