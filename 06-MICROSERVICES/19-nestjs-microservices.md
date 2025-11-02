# Module 19: NestJS Microservices

## 📚 Table of Contents
1. [Overview](#overview)
2. [NestJS Microservices Setup](#nestjs-microservices-setup)
3. [Transport Layers](#transport-layers)
4. [Message Patterns](#message-patterns)
5. [gRPC Implementation](#grpc-implementation)
6. [RabbitMQ Integration](#rabbitmq-integration)
7. [Redis Transport](#redis-transport)
8. [Hybrid Applications](#hybrid-applications)
9. [Request-Response Pattern](#request-response-pattern)
10. [Event-Based Pattern](#event-based-pattern)
11. [Error Handling](#error-handling)
12. [Best Practices](#best-practices)

---

## Overview

NestJS provides built-in support for microservices with various transport layers. This module covers implementing microservices using NestJS's microservices package.

**Transport Options:**
- **TCP**: Fast, reliable, internal communication
- **Redis**: Pub/sub messaging, scalable
- **RabbitMQ**: Message broker, routing
- **gRPC**: High performance, type-safe
- **NATS**: Lightweight messaging
- **Kafka**: High-throughput streaming

**Communication Patterns:**
- Request-Response: Synchronous calls
- Event-Based: Asynchronous messaging
- Message Queue: Producer-consumer

---

## NestJS Microservices Setup

### Install Dependencies

```bash
# For TCP transport (default)
npm install @nestjs/microservices

# For gRPC
npm install @grpc/grpc-js @grpc/proto-loader

# For RabbitMQ
npm install amqplib amqp-connection-manager

# For Redis
npm install ioredis
```

### Create Microservice

```typescript
// main.ts (Microservice entry point)
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  // Create microservice application
  // Different from HTTP application - uses microservices adapter
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      // Transport: Communication mechanism
      transport: Transport.TCP,
      
      // Options: Transport-specific configuration
      options: {
        host: 'localhost',
        port: 3001, // Microservice port (different from HTTP port)
      },
    },
  );

  // Start listening for messages
  await app.listen();
  console.log('Microservice is listening on port 3001');
}
bootstrap();
```

**Key Differences from HTTP:**
- Uses `createMicroservice()` instead of `create()`
- No HTTP server
- Listens for messages, not HTTP requests
- Uses message handlers instead of route handlers

---

## Transport Layers

### TCP Transport (Default)

TCP is the default transport, ideal for internal service communication.

```typescript
// main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.TCP,
    options: {
      host: '0.0.0.0', // Listen on all interfaces
      port: 3001,
      // Retry configuration
      retryAttempts: 5,
      retryDelay: 3000,
    },
  },
);
```

**TCP Characteristics:**
- Fast and reliable
- Low latency
- Direct connection
- No message broker needed

### Redis Transport

Redis transport uses Redis pub/sub for messaging.

```typescript
// main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.REDIS,
    options: {
      host: 'localhost',
      port: 6379,
      // Optional Redis options
      password: process.env.REDIS_PASSWORD,
      db: 0,
    },
  },
);
```

**Redis Characteristics:**
- Pub/sub messaging
- Decoupled services
- Scalable
- Requires Redis instance

### RabbitMQ Transport

RabbitMQ provides advanced message routing and queuing.

```typescript
// main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.RMQ,
    options: {
      urls: ['amqp://localhost:5672'],
      queue: 'users_queue', // Queue name
      queueOptions: {
        durable: true, // Queue survives broker restart
      },
      // Connection options
      socketOptions: {
        heartbeatIntervalInSeconds: 60,
      },
    },
  },
);
```

**RabbitMQ Characteristics:**
- Advanced routing
- Message queuing
- Durability
- Complex setup

### gRPC Transport

gRPC provides high-performance, type-safe RPC.

```typescript
// main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.GRPC,
    options: {
      package: 'users', // Proto package name
      protoPath: join(__dirname, 'users.proto'), // Path to .proto file
      url: 'localhost:5000', // gRPC server address
    },
  },
);
```

**gRPC Characteristics:**
- Very high performance
- Type-safe with Protocol Buffers
- Streaming support
- Complex setup

---

## Message Patterns

### Request-Response Pattern

Request-response is synchronous - client waits for response.

```typescript
// users.service.ts (Microservice)
import { Injectable } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';

@Injectable()
export class UsersService {
  // @MessagePattern() handles incoming messages
  // Pattern is the message identifier (like route path)
  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() data: { userId: number }) {
    // Extract userId from message payload
    const { userId } = data;
    
    // Process request (fetch from database, etc.)
    const user = await this.usersRepository.findOne({ where: { id: userId } });
    
    // Return response (automatically sent back to client)
    return {
      success: true,
      data: user,
    };
  }

  @MessagePattern({ cmd: 'create_user' })
  async createUser(@Payload() data: CreateUserDto) {
    const user = await this.usersRepository.save(data);
    return {
      success: true,
      data: user,
    };
  }

  @MessagePattern({ cmd: 'get_all_users' })
  async getAllUsers() {
    const users = await this.usersRepository.find();
    return {
      success: true,
      data: users,
    };
  }
}
```

**Client Calling Microservice:**

```typescript
// orders.service.ts (Client application)
import { Injectable, Inject, OnModuleInit } from '@nestjs/common';
import { ClientProxy, ClientProxyFactory, Transport } from '@nestjs/microservices';

@Injectable()
export class OrdersService implements OnModuleInit {
  private client: ClientProxy;

  constructor() {
    // Create client proxy to communicate with microservice
    this.client = ClientProxyFactory.create({
      transport: Transport.TCP,
      options: {
        host: 'localhost',
        port: 3001, // Microservice port
      },
    });
  }

  async onModuleInit() {
    // Connect to microservice
    await this.client.connect();
  }

  async getUser(userId: number) {
    // Send message and wait for response
    // Pattern must match @MessagePattern in microservice
    return this.client.send({ cmd: 'get_user' }, { userId }).toPromise();
  }

  async createUser(userData: CreateUserDto) {
    return this.client.send({ cmd: 'create_user' }, userData).toPromise();
  }
}
```

**Understanding Request-Response:**
- `send()` method sends message and waits for response
- Pattern must match exactly
- Payload is sent as second parameter
- Returns Promise with response
- Synchronous communication

### Event-Based Pattern

Event-based is asynchronous - fire and forget.

```typescript
// users.service.ts (Microservice - Event Handler)
import { EventPattern } from '@nestjs/microservices';

@Injectable()
export class UsersService {
  // @EventPattern() handles events (no response expected)
  @EventPattern('user.created')
  async handleUserCreated(@Payload() data: { userId: number; email: string }) {
    console.log('User created event received:', data);
    
    // Process event (send email, update cache, etc.)
    await this.sendWelcomeEmail(data.email);
    
    // No return value needed (event is one-way)
  }

  @EventPattern('user.updated')
  async handleUserUpdated(@Payload() data: { userId: number }) {
    // Invalidate cache, update search index, etc.
    await this.cacheService.invalidate(`user:${data.userId}`);
  }
}
```

**Client Emitting Events:**

```typescript
// auth.service.ts (Client - Event Emitter)
@Injectable()
export class AuthService {
  private client: ClientProxy;

  constructor() {
    this.client = ClientProxyFactory.create({
      transport: Transport.TCP,
      options: {
        host: 'localhost',
        port: 3001,
      },
    });
  }

  async register(userData: CreateUserDto) {
    // Create user in auth service
    const user = await this.createUser(userData);

    // Emit event (no response expected)
    // emit() sends event and doesn't wait
    this.client.emit('user.created', {
      userId: user.id,
      email: user.email,
    });

    return user;
  }
}
```

**Understanding Event-Based:**
- `emit()` sends event without waiting
- Events are one-way communication
- Multiple handlers can receive same event
- No response expected
- Asynchronous processing

---

## gRPC Implementation

gRPC provides high-performance RPC with Protocol Buffers.

### Define Proto File

```protobuf
// users.proto
syntax = "proto3";

package users;

// Service definition
service UsersService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
  rpc GetAllUsers (Empty) returns (GetAllUsersResponse);
}

// Request/Response messages
message GetUserRequest {
  int32 userId = 1;
}

message GetUserResponse {
  bool success = 1;
  User user = 2;
}

message CreateUserRequest {
  string email = 1;
  string password = 2;
  string firstName = 3;
  string lastName = 4;
}

message CreateUserResponse {
  bool success = 1;
  User user = 2;
}

message GetAllUsersResponse {
  bool success = 1;
  repeated User users = 2;
}

message User {
  int32 id = 1;
  string email = 2;
  string firstName = 3;
  string lastName = 4;
}

message Empty {}
```

### gRPC Service Implementation

```typescript
// users.controller.ts
import { Controller } from '@nestjs/common';
import { GrpcMethod } from '@nestjs/microservices';

@Controller()
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // @GrpcMethod() maps to proto service method
  // Method name matches proto RPC name
  @GrpcMethod('UsersService', 'GetUser')
  async getUser(data: { userId: number }) {
    const user = await this.usersService.findOne(data.userId);
    return {
      success: true,
      user: {
        id: user.id,
        email: user.email,
        firstName: user.firstName,
        lastName: user.lastName,
      },
    };
  }

  @GrpcMethod('UsersService', 'CreateUser')
  async createUser(data: CreateUserDto) {
    const user = await this.usersService.create(data);
    return {
      success: true,
      user,
    };
  }

  @GrpcMethod('UsersService', 'GetAllUsers')
  async getAllUsers() {
    const users = await this.usersService.findAll();
    return {
      success: true,
      users,
    };
  }
}
```

**gRPC Client:**

```typescript
// orders.service.ts
@Injectable()
export class OrdersService {
  private client: ClientGrpc;

  constructor(@Inject('USERS_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    // Get service stub
    this.usersService = this.client.getService<UsersServiceClient>('UsersService');
  }

  async getUser(userId: number) {
    // Call gRPC method
    return this.usersService.getUser({ userId }).toPromise();
  }
}
```

---

## RabbitMQ Integration

### RabbitMQ Setup

```typescript
// main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.RMQ,
    options: {
      urls: [process.env.RABBITMQ_URL || 'amqp://localhost:5672'],
      queue: 'users_queue',
      queueOptions: {
        durable: true, // Queue survives broker restart
      },
      // Prefetch: How many unacked messages per consumer
      prefetchCount: 1,
      // Socket options
      socketOptions: {
        heartbeatIntervalInSeconds: 60,
        reconnectTimeInSeconds: 5,
      },
    },
  },
);
```

### RabbitMQ Message Handlers

```typescript
@Injectable()
export class UsersService {
  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() data: { userId: number }) {
    // Same as TCP, but uses RabbitMQ queues
    return await this.usersRepository.findOne({ where: { id: data.userId } });
  }

  @EventPattern('user.created')
  async handleUserCreated(@Payload() data: any) {
    // Event handler for RabbitMQ events
    await this.sendWelcomeEmail(data.email);
  }
}
```

---

## Redis Transport

### Redis Setup

```typescript
// main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.REDIS,
    options: {
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT) || 6379,
      password: process.env.REDIS_PASSWORD,
      db: 0,
    },
  },
);
```

**Redis Uses Pub/Sub:**
- Services publish to channels
- Services subscribe to channels
- Decoupled communication
- Good for event-driven architecture

---

## Hybrid Applications

Hybrid applications combine HTTP and microservices in one app.

```typescript
// main.ts
async function bootstrap() {
  // Create HTTP application
  const app = await NestFactory.create(AppModule);
  
  // Connect microservice to same app
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: {
      port: 3001,
    },
  });

  // Start both HTTP and microservice
  await app.startAllMicroservices();
  await app.listen(3000);
  
  console.log('HTTP server on port 3000');
  console.log('Microservice on port 3001');
}
bootstrap();
```

**Use Cases:**
- HTTP API that calls microservices
- API Gateway pattern
- Expose HTTP endpoints that delegate to microservices

---

## Error Handling

### Exception Filters

```typescript
// rpc-exception.filter.ts
import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { RpcException } from '@nestjs/microservices';

@Catch()
export class ExceptionFilter implements RpcExceptionFilter<RpcException> {
  catch(exception: RpcException, host: ArgumentsHost): Observable<any> {
    // Handle RPC exceptions
    return throwError(() => exception.getError());
  }
}

// Usage in microservice
const app = await NestFactory.createMicroservice(AppModule);
app.useGlobalFilters(new ExceptionFilter());
```

### Throwing RPC Exceptions

```typescript
@MessagePattern({ cmd: 'get_user' })
async getUser(@Payload() data: { userId: number }) {
  const user = await this.usersRepository.findOne({ where: { id: data.userId } });
  
  if (!user) {
    // Throw RPC exception (not HTTP exception)
    throw new RpcException({
      status: 404,
      message: 'User not found',
    });
  }
  
  return user;
}
```

---

## Best Practices

### 1. Use Message Patterns Consistently

```typescript
// Consistent pattern naming
@MessagePattern({ cmd: 'users.get' })
@MessagePattern({ cmd: 'users.create' })
@MessagePattern({ cmd: 'users.update' })
```

### 2. Type-Safe Payloads

```typescript
// Define DTOs for messages
class GetUserDto {
  userId: number;
}

@MessagePattern({ cmd: 'get_user' })
async getUser(@Payload() data: GetUserDto) {
  // Type-safe
}
```

### 3. Handle Timeouts

```typescript
// Client with timeout
return this.client.send(pattern, data)
  .pipe(
    timeout(5000), // 5 second timeout
    catchError(error => {
      // Handle timeout
      throw new Error('Service timeout');
    }),
  )
  .toPromise();
```

### 4. Use Connection Pooling

Reuse client connections instead of creating new ones.

### 5. Implement Retry Logic

```typescript
this.client.send(pattern, data)
  .pipe(
    retry({
      count: 3,
      delay: 1000,
    }),
  );
```

---

## Next Steps

✅ Understand NestJS microservices  
✅ Implement request-response pattern  
✅ Use event-based communication  
✅ Configure different transports  
✅ Handle errors properly  
✅ Move to Module 20: Service Communication & API Gateway  

---

*You now know how to build microservices with NestJS! 🏗️*

