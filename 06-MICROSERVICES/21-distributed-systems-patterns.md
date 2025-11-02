# Module 21: Distributed Systems Patterns

## 📚 Table of Contents
1. [Overview](#overview)
2. [Saga Pattern](#saga-pattern)
3. [CQRS Pattern](#cqrs-pattern)
4. [Event Sourcing](#event-sourcing)
5. [Saga Orchestration](#saga-orchestration)
6. [Saga Choreography](#saga-choreography)
7. [Outbox Pattern](#outbox-pattern)
8. [Idempotency](#idempotency)
9. [Distributed Locking](#distributed-locking)
10. [Best Practices](#best-practices)

---

## Overview

Distributed systems require special patterns to handle consistency, reliability, and coordination across services. This module covers essential patterns for building robust microservices.

**Key Patterns:**
- **Saga**: Manage distributed transactions
- **CQRS**: Separate read and write models
- **Event Sourcing**: Store events instead of state
- **Outbox**: Reliable event publishing
- **Idempotency**: Safe operation retries

---

## Saga Pattern

Saga manages distributed transactions without using traditional ACID transactions.

### Saga Concepts

**Problem:** In microservices, you can't use database transactions across services.

**Solution:** Saga coordinates multiple local transactions.

**Two Types:**
1. **Orchestration**: Central coordinator
2. **Choreography**: Event-driven coordination

### Saga Orchestration Example

```typescript
// order-saga.service.ts
import { Injectable } from '@nestjs/common';
import { Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';

@Injectable()
export class OrderSagaService {
  constructor(
    @Inject('PRODUCTS_SERVICE') private productsClient: ClientProxy,
    @Inject('PAYMENT_SERVICE') private paymentClient: ClientProxy,
    @Inject('INVENTORY_SERVICE') private inventoryClient: ClientProxy,
  ) {}

  async executeCreateOrder(orderData: CreateOrderDto) {
    const sagaId = uuidv4();
    const steps = [];

    try {
      // Step 1: Reserve Inventory
      const inventoryResult = await firstValueFrom(
        this.inventoryClient.send(
          { cmd: 'reserve_inventory' },
          {
            sagaId,
            productId: orderData.productId,
            quantity: orderData.quantity,
          },
        ),
      );

      if (!inventoryResult.success) {
        throw new Error('Inventory reservation failed');
      }

      steps.push({ step: 'reserve_inventory', data: inventoryResult });

      // Step 2: Process Payment
      const paymentResult = await firstValueFrom(
        this.paymentClient.send(
          { cmd: 'charge_payment' },
          {
            sagaId,
            amount: orderData.total,
            paymentMethod: orderData.paymentMethod,
          },
        ),
      );

      if (!paymentResult.success) {
        throw new Error('Payment processing failed');
      }

      steps.push({ step: 'charge_payment', data: paymentResult });

      // Step 3: Create Order
      const order = await this.ordersRepository.save({
        ...orderData,
        sagaId,
        status: 'confirmed',
      });

      return { success: true, order };
    } catch (error) {
      // Compensation: Rollback previous steps
      await this.compensate(sagaId, steps);
      throw error;
    }
  }

  private async compensate(sagaId: string, steps: any[]) {
    // Compensate in reverse order
    for (let i = steps.length - 1; i >= 0; i--) {
      const step = steps[i];

      try {
        switch (step.step) {
          case 'charge_payment':
            await firstValueFrom(
              this.paymentClient.send(
                { cmd: 'refund_payment' },
                {
                  sagaId,
                  transactionId: step.data.transactionId,
                },
              ),
            );
            break;

          case 'reserve_inventory':
            await firstValueFrom(
              this.inventoryClient.send(
                { cmd: 'release_inventory' },
                {
                  sagaId,
                  reservationId: step.data.reservationId,
                },
              ),
            );
            break;
        }
      } catch (compensationError) {
        // Log compensation failure
        console.error(`Compensation failed for step ${step.step}:`, compensationError);
      }
    }
  }
}
```

**Saga Orchestration Benefits:**
- Centralized control
- Easier to understand flow
- Better error handling
- Transaction visibility

---

## CQRS Pattern

CQRS (Command Query Responsibility Segregation) separates read and write operations.

### Why CQRS?

**Problem:** Same data model for reads and writes can be inefficient.

**Solution:** Separate models optimized for each operation.

### CQRS Implementation

```typescript
// commands (write side)
@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(
    @InjectRepository(User) private usersRepository: Repository<User>,
    private eventBus: EventBus,
  ) {}

  async execute(command: CreateUserCommand): Promise<User> {
    const user = this.usersRepository.create(command);
    const savedUser = await this.usersRepository.save(user);

    // Publish event for read model update
    this.eventBus.publish(new UserCreatedEvent(savedUser));

    return savedUser;
  }
}

// queries (read side)
@QueryHandler(GetUserQuery)
export class GetUserHandler implements IQueryHandler<GetUserQuery> {
  constructor(
    private usersReadRepository: UsersReadRepository, // Optimized for reads
  ) {}

  async execute(query: GetUserQuery): Promise<UserView> {
    // Read from optimized read model
    return this.usersReadRepository.findByEmail(query.email);
  }
}

// Read model projection
@EventsHandler(UserCreatedEvent)
export class UserProjectionHandler {
  constructor(
    private usersReadRepository: UsersReadRepository,
  ) {}

  @OnEvent(UserCreatedEvent)
  async handle(event: UserCreatedEvent) {
    // Update read model when write happens
    await this.usersReadRepository.save({
      id: event.user.id,
      email: event.user.email,
      fullName: `${event.user.firstName} ${event.user.lastName}`,
      // Denormalized data for fast reads
    });
  }
}
```

**CQRS Benefits:**
- Optimized read models
- Independent scaling
- Complex queries without affecting writes

---

## Event Sourcing

Event Sourcing stores all changes as a sequence of events.

### Event Sourcing Implementation

```typescript
// event.entity.ts
@Entity()
export class Event {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  aggregateId: string; // User ID, Order ID, etc.

  @Column()
  eventType: string; // user.created, order.placed

  @Column('jsonb')
  eventData: any; // Event payload

  @CreateDateColumn()
  timestamp: Date;

  @Column()
  version: number; // Event version for ordering
}

// event-store.service.ts
@Injectable()
export class EventStoreService {
  constructor(
    @InjectRepository(Event)
    private eventsRepository: Repository<Event>,
  ) {}

  async appendEvent(
    aggregateId: string,
    eventType: string,
    eventData: any,
  ): Promise<Event> {
    // Get current version
    const lastEvent = await this.eventsRepository.findOne({
      where: { aggregateId },
      order: { version: 'DESC' },
    });

    const version = lastEvent ? lastEvent.version + 1 : 1;

    // Append event
    const event = this.eventsRepository.create({
      aggregateId,
      eventType,
      eventData,
      version,
    });

    return this.eventsRepository.save(event);
  }

  async getEvents(aggregateId: string): Promise<Event[]> {
    return this.eventsRepository.find({
      where: { aggregateId },
      order: { version: 'ASC' },
    });
  }

  // Rebuild aggregate from events
  async rebuildAggregate<T>(aggregateId: string, initialState: T): Promise<T> {
    const events = await this.getEvents(aggregateId);

    return events.reduce((state, event) => {
      return this.applyEvent(state, event);
    }, initialState);
  }

  private applyEvent<T>(state: T, event: Event): T {
    switch (event.eventType) {
      case 'user.created':
        return { ...state, ...event.eventData, id: event.aggregateId };
      case 'user.updated':
        return { ...state, ...event.eventData };
      case 'user.deleted':
        return null;
      default:
        return state;
    }
  }
}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(private eventStore: EventStoreService) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    // Append event instead of directly saving
    const event = await this.eventStore.appendEvent(
      uuidv4(), // Generate aggregate ID
      'user.created',
      createUserDto,
    );

    // Rebuild user from events
    return this.eventStore.rebuildAggregate(event.aggregateId, {});
  }

  async update(id: string, updateDto: UpdateUserDto): Promise<User> {
    await this.eventStore.appendEvent(id, 'user.updated', updateDto);
    return this.eventStore.rebuildAggregate(id, {});
  }

  async findOne(id: string): Promise<User> {
    // Rebuild from events
    return this.eventStore.rebuildAggregate(id, {});
  }
}
```

**Event Sourcing Benefits:**
- Complete audit trail
- Time travel (replay events)
- Event replay for recovery
- Rich query capabilities

---

## Outbox Pattern

Outbox Pattern ensures reliable event publishing by storing events in database before publishing.

### Outbox Implementation

```typescript
// outbox.entity.ts
@Entity('outbox')
export class OutboxEvent {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  eventType: string;

  @Column('jsonb')
  payload: any;

  @Column({ default: false })
  published: boolean;

  @CreateDateColumn()
  createdAt: Date;
}

// outbox.service.ts
@Injectable()
export class OutboxService {
  constructor(
    @InjectRepository(OutboxEvent)
    private outboxRepository: Repository<OutboxEvent>,
    private eventBus: EventBus,
  ) {}

  // Save event to outbox within transaction
  async saveEvent(eventType: string, payload: any) {
    const outboxEvent = this.outboxRepository.create({
      eventType,
      payload,
      published: false,
    });

    return this.outboxRepository.save(outboxEvent);
  }

  // Publish events (runs periodically)
  async publishEvents() {
    const unpublishedEvents = await this.outboxRepository.find({
      where: { published: false },
      order: { createdAt: 'ASC' },
      take: 100,
    });

    for (const event of unpublishedEvents) {
      try {
        // Publish to message broker
        await this.eventBus.publish({
          type: event.eventType,
          data: event.payload,
        });

        // Mark as published
        event.published = true;
        await this.outboxRepository.save(event);
      } catch (error) {
        console.error(`Failed to publish event ${event.id}:`, error);
        // Retry later
      }
    }
  }
}

// Usage in service
@Injectable()
export class OrdersService {
  constructor(
    private ordersRepository: Repository<Order>,
    private outboxService: OutboxService,
    private dataSource: DataSource,
  ) {}

  async createOrder(orderData: CreateOrderDto): Promise<Order> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // Create order
      const order = queryRunner.manager.create(Order, orderData);
      const savedOrder = await queryRunner.manager.save(order);

      // Save event to outbox (same transaction)
      await queryRunner.manager.save(
        queryRunner.manager.create(OutboxEvent, {
          eventType: 'order.created',
          payload: { orderId: savedOrder.id, ...savedOrder },
          published: false,
        }),
      );

      await queryRunner.commitTransaction();
      return savedOrder;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}

// Scheduled job to publish events
@Injectable()
export class OutboxPublisherService {
  constructor(private outboxService: OutboxService) {}

  @Cron('*/5 * * * * *') // Every 5 seconds
  async publishPendingEvents() {
    await this.outboxService.publishEvents();
  }
}
```

**Outbox Pattern Benefits:**
- Guaranteed event delivery
- Atomic transactions
- Reliable event publishing

---

## Idempotency

Idempotency ensures operations can be safely retried.

### Idempotent Operations

```typescript
// idempotency.service.ts
@Injectable()
export class IdempotencyService {
  constructor(
    @InjectRepository(IdempotencyKey)
    private idempotencyRepository: Repository<IdempotencyKey>,
  ) {}

  async execute<T>(
    idempotencyKey: string,
    operation: () => Promise<T>,
  ): Promise<T> {
    // Check if already executed
    const existing = await this.idempotencyRepository.findOne({
      where: { key: idempotencyKey },
    });

    if (existing) {
      // Return cached result
      return existing.result as T;
    }

    // Execute operation
    const result = await operation();

    // Store result
    await this.idempotencyRepository.save({
      key: idempotencyKey,
      result: result,
      createdAt: new Date(),
    });

    return result;
  }
}

// Usage
@Post('orders')
async createOrder(
  @Body() createOrderDto: CreateOrderDto,
  @Headers('idempotency-key') idempotencyKey: string,
) {
  if (!idempotencyKey) {
    idempotencyKey = uuidv4(); // Generate if not provided
  }

  return this.idempotencyService.execute(idempotencyKey, async () => {
    return this.ordersService.create(createOrderDto);
  });
}
```

---

## Distributed Locking

Distributed locks coordinate access to shared resources.

### Redis Distributed Lock

```typescript
// distributed-lock.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class DistributedLockService {
  constructor(private redisService: RedisService) {}

  async acquireLock(
    resource: string,
    ttl: number = 30000, // 30 seconds default
  ): Promise<string | null> {
    const lockKey = `lock:${resource}`;
    const lockValue = uuidv4();

    // Try to acquire lock (SET NX EX)
    const acquired = await this.redisService.getClient().set(
      lockKey,
      lockValue,
      'EX', // Expiration in seconds
      Math.floor(ttl / 1000),
      'NX', // Only if not exists
    );

    if (acquired === 'OK') {
      return lockValue; // Return lock identifier
    }

    return null; // Lock not acquired
  }

  async releaseLock(resource: string, lockValue: string): Promise<boolean> {
    const lockKey = `lock:${resource}`;
    
    // Lua script for atomic check-and-delete
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;

    const result = await this.redisService.getClient().eval(
      script,
      1,
      lockKey,
      lockValue,
    );

    return result === 1;
  }

  async withLock<T>(
    resource: string,
    operation: () => Promise<T>,
    ttl: number = 30000,
  ): Promise<T> {
    const lockValue = await this.acquireLock(resource, ttl);

    if (!lockValue) {
      throw new Error(`Failed to acquire lock for ${resource}`);
    }

    try {
      return await operation();
    } finally {
      await this.releaseLock(resource, lockValue);
    }
  }
}

// Usage
async processOrder(orderId: number) {
  return this.lockService.withLock(`order:${orderId}`, async () => {
    // Critical section - only one process can execute at a time
    const order = await this.ordersRepository.findOne({ where: { id: orderId } });
    
    if (order.status !== 'pending') {
      throw new Error('Order already processed');
    }

    // Process order
    order.status = 'processing';
    await this.ordersRepository.save(order);

    // ... rest of processing
  });
}
```

---

## Best Practices

### 1. Make Operations Idempotent

Always design operations to be safely retryable.

### 2. Use Outbox for Event Publishing

Ensure events are published reliably.

### 3. Implement Proper Compensation

Saga compensations must be idempotent too.

### 4. Use Distributed Locks Carefully

Locks can cause deadlocks and performance issues.

### 5. Monitor Saga Progress

Track saga execution for debugging.

### 6. Handle Partial Failures

Design for graceful degradation.

---

## Next Steps

✅ Understand Saga pattern  
✅ Implement CQRS  
✅ Use Event Sourcing  
✅ Implement Outbox pattern  
✅ Ensure idempotency  
✅ Move to Module 22: Observability & Monitoring  

---

*You now know distributed systems patterns! 🎯*

