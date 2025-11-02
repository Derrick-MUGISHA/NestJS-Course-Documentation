# Module 12: Code Quality & Best Practices

## 📚 Table of Contents
1. [Overview](#overview)
2. [ESLint Configuration](#eslint-configuration)
3. [Prettier Setup](#prettier-setup)
4. [Pre-commit Hooks](#pre-commit-hooks)
5. [SOLID Principles in NestJS](#solid-principles-in-nestjs)
6. [Design Patterns](#design-patterns)
7. [Error Handling Best Practices](#error-handling-best-practices)
8. [Performance Optimization](#performance-optimization)
9. [Code Organization](#code-organization)
10. [Documentation](#documentation)
11. [Best Practices Summary](#best-practices-summary)

---

## Overview

Code quality is essential for maintainable, scalable applications. This module covers tools, principles, and practices to ensure your NestJS codebase is clean, consistent, and professional.

**Topics Covered:**
- Linting and formatting (ESLint, Prettier)
- Code quality tools (Husky, lint-staged)
- SOLID principles
- Design patterns
- Performance optimization
- Code organization

---

## ESLint Configuration

ESLint finds and fixes problems in your JavaScript/TypeScript code.

### Install ESLint

NestJS comes with ESLint pre-configured, but you can customize it:

```bash
npm install -D @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

### ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  parserOptions: {
    project: 'tsconfig.json',
    tsconfigRootDir: __dirname,
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint/eslint-plugin'],
  extends: [
    'plugin:@typescript-eslint/recommended',
    'plugin:prettier/recommended',
  ],
  root: true,
  env: {
    node: true,
    jest: true,
  },
  ignorePatterns: ['.eslintrc.js'],
  rules: {
    '@typescript-eslint/interface-name-prefix': 'off',
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/explicit-module-boundary-types': 'off',
    '@typescript-eslint/no-explicit-any': 'warn',
    // Custom rules
    'no-console': 'warn', // Warn about console.log
    'prefer-const': 'error', // Require const for variables that never change
    'no-unused-vars': 'off', // Turn off base rule
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
  },
};
```

**Key Rules Explained:**
- `no-console`: Prevents accidental console.log in production
- `prefer-const`: Encourages immutable variables
- `no-unused-vars`: Catches unused variables and imports
- `@typescript-eslint/no-explicit-any`: Discourages use of `any` type

---

## Prettier Setup

Prettier formats your code automatically for consistency.

### Install Prettier

```bash
npm install -D prettier eslint-config-prettier eslint-plugin-prettier
```

### Prettier Configuration

```json
// .prettierrc
{
  "singleQuote": true,
  "trailingComma": "es5",
  "tabWidth": 2,
  "semi": true,
  "printWidth": 80,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

**Configuration Explained:**
- `singleQuote`: Use single quotes instead of double
- `trailingComma`: Add trailing commas where valid
- `tabWidth`: 2 spaces for indentation
- `semi`: Use semicolons
- `printWidth`: Wrap lines at 80 characters
- `arrowParens`: Always include parens in arrow functions

### Format on Save (VS Code)

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

---

## Pre-commit Hooks

Pre-commit hooks run checks before code is committed, preventing bad code from entering the repository.

### Install Husky and lint-staged

```bash
npm install -D husky lint-staged
npx husky install
```

### Configure Husky

```bash
# Create pre-commit hook
npx husky add .husky/pre-commit "npx lint-staged"
```

### Configure lint-staged

```json
// package.json
{
  "lint-staged": {
    "*.{ts,js}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

**How It Works:**
1. Developer commits code
2. Husky runs pre-commit hook
3. lint-staged runs on staged files
4. ESLint fixes issues
5. Prettier formats code
6. If all pass, commit proceeds
7. If fails, commit is blocked

---

## SOLID Principles in NestJS

SOLID principles help write maintainable, scalable code.

### Single Responsibility Principle (SRP)

Each class should have one reason to change.

```typescript
// Good: Service handles business logic only
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    // Only business logic here
    return this.usersRepository.save(createUserDto);
  }
}

// Bad: Service doing too much
@Injectable()
export class UsersService {
  async create(createUserDto: CreateUserDto): Promise<User> {
    // ❌ Sending email (should be in EmailService)
    await this.sendEmail(createUserDto.email);
    
    // ❌ Logging to file (should be in LoggerService)
    this.writeToLogFile('User created');
    
    // ✅ Creating user (correct)
    return this.usersRepository.save(createUserDto);
  }
}
```

### Open/Closed Principle (OCP)

Open for extension, closed for modification.

```typescript
// Base strategy interface
interface PaymentStrategy {
  processPayment(amount: number): Promise<boolean>;
}

// Concrete implementations
@Injectable()
export class CreditCardStrategy implements PaymentStrategy {
  async processPayment(amount: number): Promise<boolean> {
    // Credit card processing
    return true;
  }
}

@Injectable()
export class PayPalStrategy implements PaymentStrategy {
  async processPayment(amount: number): Promise<boolean> {
    // PayPal processing
    return true;
  }
}

// Service uses strategy (open for extension via new strategies)
@Injectable()
export class PaymentService {
  constructor(
    private readonly creditCardStrategy: CreditCardStrategy,
    private readonly payPalStrategy: PayPalStrategy,
  ) {}

  async processPayment(type: 'credit-card' | 'paypal', amount: number) {
    const strategy = type === 'credit-card' 
      ? this.creditCardStrategy 
      : this.payPalStrategy;
    
    return strategy.processPayment(amount);
  }
}

// ✅ Can add new payment methods without modifying PaymentService
```

### Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types.

```typescript
// Base interface
interface Repository<T> {
  findOne(id: number): Promise<T>;
  findAll(): Promise<T[]>;
}

// PostgreSQL implementation
@Injectable()
export class PostgresUserRepository implements Repository<User> {
  async findOne(id: number): Promise<User> {
    // PostgreSQL implementation
  }
  
  async findAll(): Promise<User[]> {
    // PostgreSQL implementation
  }
}

// MongoDB implementation (can be used interchangeably)
@Injectable()
export class MongoUserRepository implements Repository<User> {
  async findOne(id: number): Promise<User> {
    // MongoDB implementation
  }
  
  async findAll(): Promise<User[]> {
    // MongoDB implementation
  }
}

// Service doesn't care which implementation is used
@Injectable()
export class UsersService {
  constructor(
    @Inject('UserRepository') private repository: Repository<User>,
  ) {}
  
  // Works with any Repository<User> implementation
}
```

### Interface Segregation Principle (ISP)

Clients shouldn't depend on interfaces they don't use.

```typescript
// Bad: One large interface
interface UserService {
  createUser(): Promise<User>;
  updateUser(): Promise<User>;
  deleteUser(): Promise<void>;
  sendEmail(): Promise<void>;
  generateReport(): Promise<Report>;
}

// Good: Segregated interfaces
interface UserRepository {
  createUser(): Promise<User>;
  updateUser(): Promise<User>;
  deleteUser(): Promise<void>;
}

interface EmailService {
  sendEmail(): Promise<void>;
}

interface ReportService {
  generateReport(): Promise<Report>;
}

// Services only depend on what they need
@Injectable()
export class UsersService {
  constructor(
    private userRepo: UserRepository, // Only needs repository
  ) {}
}
```

### Dependency Inversion Principle (DIP)

Depend on abstractions, not concretions.

```typescript
// Bad: Depends on concrete class
@Injectable()
export class OrdersService {
  private emailService = new EmailService(); // ❌ Concrete dependency

  async createOrder() {
    await this.emailService.sendOrderConfirmation();
  }
}

// Good: Depends on abstraction
interface IEmailService {
  sendOrderConfirmation(): Promise<void>;
}

@Injectable()
export class OrdersService {
  constructor(
    @Inject('IEmailService') private emailService: IEmailService, // ✅ Abstraction
  ) {}
}

// Implementation can be swapped without changing OrdersService
@Injectable()
export class SendGridEmailService implements IEmailService {
  async sendOrderConfirmation(): Promise<void> {
    // SendGrid implementation
  }
}
```

---

## Design Patterns

### Repository Pattern

Abstracts data access logic:

```typescript
// Repository interface
export interface IUserRepository {
  findById(id: number): Promise<User>;
  findByEmail(email: string): Promise<User>;
  save(user: User): Promise<User>;
}

// Implementation
@Injectable()
export class UserRepository implements IUserRepository {
  constructor(
    @InjectRepository(User)
    private repository: Repository<User>,
  ) {}

  async findById(id: number): Promise<User> {
    return this.repository.findOne({ where: { id } });
  }

  async findByEmail(email: string): Promise<User> {
    return this.repository.findOne({ where: { email } });
  }

  async save(user: User): Promise<User> {
    return this.repository.save(user);
  }
}
```

### Factory Pattern

Creates objects without specifying exact classes:

```typescript
@Injectable()
export class PaymentFactory {
  constructor(
    private creditCardStrategy: CreditCardStrategy,
    private payPalStrategy: PayPalStrategy,
  ) {}

  createStrategy(type: PaymentType): PaymentStrategy {
    switch (type) {
      case PaymentType.CREDIT_CARD:
        return this.creditCardStrategy;
      case PaymentType.PAYPAL:
        return this.payPalStrategy;
      default:
        throw new Error(`Unsupported payment type: ${type}`);
    }
  }
}
```

### Strategy Pattern

Define family of algorithms and make them interchangeable:

```typescript
// Strategy interface
interface NotificationStrategy {
  send(message: string, recipient: string): Promise<void>;
}

// Email strategy
@Injectable()
export class EmailNotificationStrategy implements NotificationStrategy {
  async send(message: string, recipient: string): Promise<void> {
    // Send email
  }
}

// SMS strategy
@Injectable()
export class SmsNotificationStrategy implements NotificationStrategy {
  async send(message: string, recipient: string): Promise<void> {
    // Send SMS
  }
}

// Context that uses strategy
@Injectable()
export class NotificationService {
  constructor(
    private emailStrategy: EmailNotificationStrategy,
    private smsStrategy: SmsNotificationStrategy,
  ) {}

  async sendNotification(
    type: 'email' | 'sms',
    message: string,
    recipient: string,
  ) {
    const strategy = type === 'email' 
      ? this.emailStrategy 
      : this.smsStrategy;
    
    await strategy.send(message, recipient);
  }
}
```

---

## Error Handling Best Practices

### Use Specific Exceptions

```typescript
// Good: Specific exceptions
if (!user) {
  throw new NotFoundException(`User with ID ${id} not found`);
}

if (user.isBanned) {
  throw new ForbiddenException('User account is banned');
}

// Bad: Generic exceptions
throw new Error('Something went wrong'); // ❌ Too vague
```

### Global Exception Filter

```typescript
// src/common/filters/http-exception.filter.ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    const status = 
      exception instanceof HttpException
        ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    const message = 
      exception instanceof HttpException
        ? exception.getMessage()
      : 'Internal server error';

    // Log error for monitoring
    console.error('Exception:', exception);

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}
```

---

## Performance Optimization

### Database Query Optimization

```typescript
// Bad: N+1 query problem
async findAll(): Promise<User[]> {
  const users = await this.usersRepository.find();
  
  // ❌ Makes additional query for each user
  for (const user of users) {
    user.posts = await this.postsRepository.find({ where: { userId: user.id } });
  }
  
  return users;
}

// Good: Single query with relations
async findAll(): Promise<User[]> {
  // ✅ Loads posts in one query
  return this.usersRepository.find({
    relations: ['posts'],
  });
}
```

### Use Pagination

```typescript
async findAll(page: number = 1, limit: number = 10): Promise<PaginatedResult<User>> {
  const [data, total] = await this.usersRepository.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
  });

  return {
    data,
    total,
    page,
    limit,
    totalPages: Math.ceil(total / limit),
  };
}
```

### Caching Expensive Operations

```typescript
@Injectable()
export class ProductsService {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private productsRepository: Repository<Product>,
  ) {}

  async findAll(): Promise<Product[]> {
    const cached = await this.cacheManager.get<Product[]>('products:all');
    if (cached) return cached;

    const products = await this.productsRepository.find();
    await this.cacheManager.set('products:all', products, { ttl: 300 });
    
    return products;
  }
}
```

---

## Code Organization

### Feature-Based Structure

```
src/
├── users/
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   ├── entities/
│   │   └── user.entity.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── users.module.ts
│   └── users.service.spec.ts
├── products/
└── orders/
```

### Shared Code Organization

```
src/
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/
└── database/
    ├── migrations/
    └── seeds/
```

---

## Documentation

### Code Comments

```typescript
/**
 * Creates a new user account
 * 
 * @param createUserDto - User registration data
 * @returns The created user object (without password)
 * @throws BadRequestException if email already exists
 * @throws ValidationException if data is invalid
 * 
 * @example
 * ```typescript
 * const user = await usersService.create({
 *   email: 'user@example.com',
 *   password: 'secure123',
 * });
 * ```
 */
async create(createUserDto: CreateUserDto): Promise<User> {
  // Implementation
}
```

### README Files

Include clear README files for each module explaining:
- Purpose
- Usage
- Examples
- Dependencies

---

## Best Practices Summary

### 1. Use TypeScript Strictly

```typescript
// Good
function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Bad
function calculateTotal(items: any): any {
  return items.reduce((sum: any, item: any) => sum + item.price, 0);
}
```

### 2. Keep Functions Small

```typescript
// Good: Small, focused function
async validateUser(userId: number): Promise<User> {
  const user = await this.usersRepository.findOne({ where: { id: userId } });
  if (!user) {
    throw new NotFoundException('User not found');
  }
  return user;
}

// Bad: Too many responsibilities
async processOrder(orderId: number): Promise<void> {
  // 50+ lines doing multiple things
}
```

### 3. Use Meaningful Names

```typescript
// Good
const activeUsers = await this.getActiveUsers();
const userEmailAddress = user.email;

// Bad
const data = await this.getUsers();
const email = user.e;
```

### 4. Avoid Deep Nesting

```typescript
// Good: Early returns
if (!user) {
  throw new NotFoundException();
}

if (user.isBanned) {
  throw new ForbiddenException();
}

return user;

// Bad: Deep nesting
if (user) {
  if (!user.isBanned) {
    return user;
  } else {
    throw new ForbiddenException();
  }
} else {
  throw new NotFoundException();
}
```

---

## Next Steps

✅ Configure ESLint and Prettier  
✅ Set up pre-commit hooks  
✅ Apply SOLID principles  
✅ Optimize performance  
✅ Organize code structure  
✅ Ready for production! 🚀

---

*You now have the tools and knowledge for professional code quality! ✨*

