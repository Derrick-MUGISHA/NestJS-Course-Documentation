# Module 11: Comprehensive Testing

## 📚 Table of Contents
1. [Overview](#overview)
2. [Testing Setup](#testing-setup)
3. [Unit Testing](#unit-testing)
4. [Integration Testing](#integration-testing)
5. [E2E Testing](#e2e-testing)
6. [Testing Services](#testing-services)
7. [Testing Controllers](#testing-controllers)
8. [Testing Guards and Pipes](#testing-guards-and-pipes)
9. [Testing with Database](#testing-with-database)
10. [Test Coverage](#test-coverage)
11. [Mocking Strategies](#mocking-strategies)
12. [Best Practices](#best-practices)
13. [Mini Project: Test Suite](#mini-project-test-suite)

---

## Overview

Testing is crucial for maintaining code quality and preventing bugs. This module covers comprehensive testing strategies in NestJS, from unit tests to end-to-end tests.

**Testing Levels:**
1. **Unit Tests**: Test individual components in isolation
2. **Integration Tests**: Test how components work together
3. **E2E Tests**: Test complete user flows

**Why Test?**
- Catch bugs before production
- Enable confident refactoring
- Document expected behavior
- Prevent regressions
- Improve code design

---

## Testing Setup

NestJS comes with Jest configured out of the box.

### Jest Configuration

The default `package.json` includes test scripts:

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  }
}
```

### Test Files Structure

```
src/
├── users/
│   ├── users.controller.ts
│   ├── users.controller.spec.ts    # Unit tests for controller
│   ├── users.service.ts
│   └── users.service.spec.ts       # Unit tests for service
test/
├── app.e2e-spec.ts                  # E2E tests
└── jest-e2e.json                    # E2E test configuration
```

---

## Unit Testing

Unit tests test individual functions, methods, or classes in isolation.

### Testing a Service

```typescript
// src/users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';

describe('UsersService', () => {
  let service: UsersService;
  let repository: Repository<User>;

  // beforeEach runs before each test
  // This sets up a fresh test environment for each test
  beforeEach(async () => {
    // Create a testing module
    // This is like a mini NestJS application for testing
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        // Mock the repository instead of using real database
        // getRepositoryToken() gets the injection token for the repository
        {
          provide: getRepositoryToken(User),
          // useValue creates a mock object with the methods we need
          useValue: {
            find: jest.fn(),
            findOne: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
            update: jest.fn(),
            delete: jest.fn(),
          },
        },
      ],
    }).compile();

    // Get instances from the testing module
    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });

  // it() or test() defines a single test case
  it('should be defined', () => {
    // expect() is Jest's assertion function
    // This test just verifies the service exists
    expect(service).toBeDefined();
  });

  // Test the findAll() method
  describe('findAll', () => {
    it('should return an array of users', async () => {
      // Arrange: Set up test data
      const mockUsers: User[] = [
        { id: 1, email: 'user1@example.com', password: 'hash1' } as User,
        { id: 2, email: 'user2@example.com', password: 'hash2' } as User,
      ];

      // Mock repository.find() to return our test data
      // jest.fn().mockResolvedValue() makes the mock function return a Promise
      jest.spyOn(repository, 'find').mockResolvedValue(mockUsers);

      // Act: Call the method we're testing
      const result = await service.findAll();

      // Assert: Verify the result
      expect(result).toEqual(mockUsers);
      
      // Verify the repository method was called
      expect(repository.find).toHaveBeenCalledTimes(1);
    });

    it('should return empty array when no users exist', async () => {
      // Mock empty result
      jest.spyOn(repository, 'find').mockResolvedValue([]);

      const result = await service.findAll();

      expect(result).toEqual([]);
      expect(result).toHaveLength(0);
    });
  });

  describe('findOne', () => {
    it('should return a user by id', async () => {
      const mockUser: User = {
        id: 1,
        email: 'test@example.com',
        password: 'hashed',
      } as User;

      // Mock findOne with where clause
      jest.spyOn(repository, 'findOne').mockResolvedValue(mockUser);

      const result = await service.findOne(1);

      expect(result).toEqual(mockUser);
      expect(repository.findOne).toHaveBeenCalledWith({ where: { id: 1 } });
    });

    it('should throw NotFoundException when user not found', async () => {
      // Mock findOne to return null (user not found)
      jest.spyOn(repository, 'findOne').mockResolvedValue(null);

      // Expect the service to throw an exception
      await expect(service.findOne(999)).rejects.toThrow('User not found');
    });
  });

  describe('create', () => {
    it('should create a new user', async () => {
      const createUserDto = {
        email: 'new@example.com',
        password: 'password123',
      };

      const createdUser: User = {
        id: 1,
        ...createUserDto,
        password: 'hashed_password',
      } as User;

      // Mock repository methods
      jest.spyOn(repository, 'create').mockReturnValue(createdUser);
      jest.spyOn(repository, 'save').mockResolvedValue(createdUser);

      const result = await service.create(createUserDto);

      expect(result).toEqual(createdUser);
      expect(repository.create).toHaveBeenCalledWith(createUserDto);
      expect(repository.save).toHaveBeenCalledWith(createdUser);
    });
  });
});
```

**Key Testing Concepts:**
- **Arrange-Act-Assert (AAA)**: Structure your tests
- **Mocking**: Replace dependencies with fake implementations
- **Spies**: Track function calls and arguments
- **Assertions**: Verify expected behavior

---

## Integration Testing

Integration tests verify that multiple components work together correctly.

### Testing with Real Database

```typescript
// src/users/users.integration.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { Repository } from 'typeorm';
import { getRepositoryToken } from '@nestjs/typeorm';

describe('UsersService Integration', () => {
  let service: UsersService;
  let repository: Repository<User>;
  let module: TestingModule;

  beforeAll(async () => {
    // beforeAll runs once before all tests
    // Set up test database connection
    module = await Test.createTestingModule({
      imports: [
        // Use test database (separate from development)
        TypeOrmModule.forRoot({
          type: 'postgres',
          host: 'localhost',
          port: 5432,
          username: 'postgres',
          password: 'test',
          database: 'test_db',
          entities: [User],
          synchronize: true, // Only for tests
        }),
        TypeOrmModule.forFeature([User]),
      ],
      providers: [UsersService],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });

  afterEach(async () => {
    // Clean up after each test
    await repository.clear();
  });

  afterAll(async () => {
    // Close database connection
    await module.close();
  });

  it('should create and find a user', async () => {
    // Create user
    const user = await service.create({
      email: 'test@example.com',
      password: 'password123',
    });

    expect(user.id).toBeDefined();
    expect(user.email).toBe('test@example.com');

    // Find user
    const found = await service.findOne(user.id);
    expect(found.email).toBe('test@example.com');
  });
});
```

---

## E2E Testing

End-to-end tests verify complete user flows from API request to response.

### E2E Test Setup

```typescript
// test/app.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication;

  // beforeAll: Set up the entire application
  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    // Create the actual NestJS application
    app = moduleFixture.createNestApplication();
    
    // Apply same configuration as production
    app.useGlobalPipes(new ValidationPipe());
    
    // Initialize the application
    await app.init();
  });

  // afterAll: Clean up after all tests
  afterAll(async () => {
    await app.close();
  });

  // Test a complete API endpoint
  it('/ (GET)', () => {
    // request(app.getHttpServer()) creates a request to the app
    // .get('/') sends a GET request to the root path
    // .expect(200) expects HTTP status 200
    // .expect('Hello World!') expects the response body
    return request(app.getHttpServer())
      .get('/')
      .expect(200)
      .expect('Hello World!');
  });
});
```

### Complete E2E Test Example

```typescript
// test/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Users (e2e)', () => {
  let app: INestApplication;
  let authToken: string;
  let userId: number;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /users (Register)', () => {
    it('should create a new user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'e2e-test@example.com',
          password: 'password123',
          firstName: 'Test',
          lastName: 'User',
        })
        .expect(201)
        .expect((res) => {
          expect(res.body).toHaveProperty('id');
          expect(res.body.email).toBe('e2e-test@example.com');
          userId = res.body.id;
        });
    });

    it('should reject invalid email', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'invalid-email',
          password: 'password123',
        })
        .expect(400); // Bad Request
    });
  });

  describe('POST /auth/login', () => {
    it('should login and return JWT token', async () => {
      const response = await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'e2e-test@example.com',
          password: 'password123',
        })
        .expect(200);

      expect(response.body).toHaveProperty('access_token');
      authToken = response.body.access_token;
    });

    it('should reject invalid credentials', () => {
      return request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'e2e-test@example.com',
          password: 'wrongpassword',
        })
        .expect(401); // Unauthorized
    });
  });

  describe('GET /users/:id (Protected)', () => {
    it('should get user profile with valid token', () => {
      return request(app.getHttpServer())
        .get(`/users/${userId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)
        .expect((res) => {
          expect(res.body.id).toBe(userId);
        });
    });

    it('should reject request without token', () => {
      return request(app.getHttpServer())
        .get(`/users/${userId}`)
        .expect(401); // Unauthorized
    });
  });
});
```

---

## Testing Services

### Testing Service with Dependencies

```typescript
// src/orders/orders.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { OrdersService } from './orders.service';
import { UsersService } from '../users/users.service';
import { ProductsService } from '../products/products.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Order } from './entities/order.entity';

describe('OrdersService', () => {
  let service: OrdersService;
  let usersService: UsersService;
  let productsService: ProductsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrdersService,
        {
          provide: UsersService,
          // Mock UsersService
          useValue: {
            findOne: jest.fn(),
          },
        },
        {
          provide: ProductsService,
          useValue: {
            findOne: jest.fn(),
            updateStock: jest.fn(),
          },
        },
        {
          provide: getRepositoryToken(Order),
          useValue: {
            create: jest.fn(),
            save: jest.fn(),
            find: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<OrdersService>(OrdersService);
    usersService = module.get<UsersService>(UsersService);
    productsService = module.get<ProductsService>(ProductsService);
  });

  it('should create an order', async () => {
    const userId = 1;
    const productId = 2;
    const quantity = 3;

    // Mock dependencies
    jest.spyOn(usersService, 'findOne').mockResolvedValue({ id: userId } as any);
    jest.spyOn(productsService, 'findOne').mockResolvedValue({
      id: productId,
      stock: 10,
      price: 100,
    } as any);
    jest.spyOn(productsService, 'updateStock').mockResolvedValue(undefined);

    const order = await service.create({
      userId,
      productId,
      quantity,
    });

    expect(order).toBeDefined();
    expect(productsService.updateStock).toHaveBeenCalledWith(productId, -quantity);
  });
});
```

---

## Testing Controllers

```typescript
// src/users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: {
            findAll: jest.fn(),
            findOne: jest.fn(),
            create: jest.fn(),
            update: jest.fn(),
            remove: jest.fn(),
          },
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
  });

  it('should return an array of users', async () => {
    const result = [{ id: 1, email: 'test@example.com' }];
    
    jest.spyOn(service, 'findAll').mockResolvedValue(result as any);

    expect(await controller.findAll()).toBe(result);
    expect(service.findAll).toHaveBeenCalled();
  });

  it('should create a user', async () => {
    const createDto = { email: 'new@example.com', password: 'pass' };
    const created = { id: 1, ...createDto };

    jest.spyOn(service, 'create').mockResolvedValue(created as any);

    expect(await controller.create(createDto)).toBe(created);
    expect(service.create).toHaveBeenCalledWith(createDto);
  });
});
```

---

## Testing Guards and Pipes

### Testing a Guard

```typescript
// src/auth/guards/jwt-auth.guard.spec.ts
import { JwtAuthGuard } from './jwt-auth.guard';
import { ExecutionContext } from '@nestjs/common';
import { UnauthorizedException } from '@nestjs/common';

describe('JwtAuthGuard', () => {
  let guard: JwtAuthGuard;
  let executionContext: ExecutionContext;

  beforeEach(() => {
    guard = new JwtAuthGuard();
    
    // Mock execution context
    executionContext = {
      switchToHttp: jest.fn().mockReturnValue({
        getRequest: jest.fn().mockReturnValue({
          headers: {
            authorization: 'Bearer valid-token',
          },
        }),
      }),
    } as any;
  });

  it('should allow request with valid token', () => {
    // Mock canActivate to return true
    jest.spyOn(guard, 'canActivate').mockReturnValue(true);

    expect(guard.canActivate(executionContext)).toBe(true);
  });

  it('should reject request without token', () => {
    executionContext.switchToHttp().getRequest().headers.authorization = undefined;

    expect(() => guard.canActivate(executionContext)).toThrow(UnauthorizedException);
  });
});
```

---

## Testing with Database

### Using Test Database

```typescript
// test-setup.ts
import { Test } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';

export async function createTestModule(entities: any[]) {
  return Test.createTestingModule({
    imports: [
      TypeOrmModule.forRoot({
        type: 'postgres',
        host: process.env.TEST_DB_HOST || 'localhost',
        database: process.env.TEST_DB_NAME || 'test_db',
        synchronize: true, // Only for tests!
        dropSchema: true, // Drop schema before each test run
        entities,
      }),
    ],
  }).compile();
}
```

---

## Test Coverage

### Generate Coverage Report

```bash
npm run test:cov
```

This generates a coverage report showing:
- **Statements**: Percentage of code executed
- **Branches**: Percentage of branches tested
- **Functions**: Percentage of functions called
- **Lines**: Percentage of lines executed

### Coverage Thresholds

Configure minimum coverage in `package.json`:

```json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

---

## Mocking Strategies

### Mock External Services

```typescript
// Mock email service
const mockEmailService = {
  sendEmail: jest.fn().mockResolvedValue({ sent: true }),
  sendBulkEmail: jest.fn().mockResolvedValue({ sent: 5 }),
};

// Mock HTTP calls
jest.mock('axios', () => ({
  get: jest.fn(),
  post: jest.fn(),
}));
```

### Mock Modules

```typescript
const module: TestingModule = await Test.createTestingModule({
  imports: [
    // Import real module
    UsersModule,
  ],
})
.overrideModule(EmailModule)
.useModule(TestEmailModule) // Replace with test module
.compile();
```

---

## Best Practices

### 1. Follow AAA Pattern

```typescript
it('should do something', () => {
  // Arrange: Set up test data
  const input = { ... };
  
  // Act: Execute the code
  const result = service.doSomething(input);
  
  // Assert: Verify the result
  expect(result).toBe(expected);
});
```

### 2. Test Edge Cases

```typescript
it('should handle empty array', () => {});
it('should handle null input', () => {});
it('should handle invalid data', () => {});
```

### 3. Use Descriptive Test Names

```typescript
// Good
it('should return 404 when user does not exist', () => {});

// Bad
it('should work', () => {});
```

### 4. Keep Tests Independent

Each test should be able to run in isolation without depending on other tests.

### 5. Mock External Dependencies

Never test external services, databases, or APIs in unit tests.

---

## Mini Project: Test Suite

Write comprehensive tests for a complete module:

1. Unit tests for service
2. Unit tests for controller
3. Integration tests with database
4. E2E tests for API endpoints
5. Achieve 80%+ code coverage

---

## Next Steps

✅ Write unit tests for all services  
✅ Write integration tests  
✅ Write E2E tests for API flows  
✅ Achieve high test coverage  
✅ Move to Module 12: Code Quality & Best Practices  

---

*You now know how to test your NestJS applications comprehensively! 🧪*

