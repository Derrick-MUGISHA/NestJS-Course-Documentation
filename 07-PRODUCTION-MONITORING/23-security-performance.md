# Module 23: Security & Performance

## 📚 Table of Contents
1. [Overview](#overview)
2. [Security Best Practices](#security-best-practices)
3. [OWASP Top 10](#owasp-top-10)
4. [Rate Limiting](#rate-limiting)
5. [SQL Injection Prevention](#sql-injection-prevention)
6. [XSS and CSRF Protection](#xss-and-csrf-protection)
7. [Performance Optimization](#performance-optimization)
8. [Database Optimization](#database-optimization)
9. [Caching Strategies](#caching-strategies)
10. [Load Testing](#load-testing)
11. [Best Practices](#best-practices)

---

## Overview

Security and performance are critical for production applications. This module covers securing your NestJS applications and optimizing performance.

**Security Topics:**
- Authentication and authorization
- Input validation
- SQL injection prevention
- XSS/CSRF protection
- Secrets management

**Performance Topics:**
- Query optimization
- Caching strategies
- Connection pooling
- Code optimization

---

## Security Best Practices

### 1. Use HTTPS

Always use HTTPS in production:

```typescript
// Enable HTTPS
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/certificate.pem'),
};

const app = await NestFactory.create(AppModule, {
  httpsOptions,
});
```

### 2. Environment Variables

Never hardcode secrets:

```typescript
// Good: Use environment variables
const jwtSecret = process.env.JWT_SECRET;

// Bad: Hardcoded secret
const jwtSecret = 'my-secret-key'; // ❌
```

### 3. Input Validation

Always validate input:

```typescript
// users.dto.ts
export class CreateUserDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
  password: string;
}
```

---

## OWASP Top 10

### 1. Broken Access Control

**Prevention:**

```typescript
// Use guards for authorization
@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)
export class UsersController {
  @Get(':id')
  @Roles('admin', 'user')
  async findOne(@Param('id') id: string, @CurrentUser() user: User) {
    // Users can only access their own data unless admin
    if (user.role !== 'admin' && user.id !== +id) {
      throw new ForbiddenException('Access denied');
    }
    return this.usersService.findOne(+id);
  }
}
```

### 2. Cryptographic Failures

**Prevention:**

```typescript
// Always hash passwords
const hashedPassword = await bcrypt.hash(password, 12);

// Use strong encryption
const encrypted = crypto.createCipheriv(
  'aes-256-gcm',
  key,
  iv,
);

// Use secure random for tokens
const token = crypto.randomBytes(32).toString('hex');
```

### 3. Injection

**Prevention:**

```typescript
// Use parameterized queries (TypeORM does this automatically)
// Good: TypeORM handles parameterization
await this.repository.findOne({ where: { id: userId } });

// Bad: Raw SQL with concatenation
await this.repository.query(`SELECT * FROM users WHERE id = ${userId}`); // ❌
```

---

## Rate Limiting

Rate limiting prevents abuse and DoS attacks.

### Implement Rate Limiting

```typescript
// rate-limit.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { ThrottlerGuard } from '@nestjs/throttler';

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  protected getTracker(req: Record<string, any>): Promise<string> {
    // Track by IP address
    return Promise.resolve(req.ip);
  }
}

// Global rate limit
app.useGlobalGuards(new CustomThrottlerGuard());
```

### Per-Endpoint Rate Limiting

```typescript
@Post('login')
@Throttle(5, 60) // 5 requests per 60 seconds
async login(@Body() loginDto: LoginDto) {
  return this.authService.login(loginDto);
}
```

---

## SQL Injection Prevention

TypeORM protects against SQL injection by default.

### Safe Query Building

```typescript
// Good: TypeORM parameterization
const user = await this.repository.findOne({
  where: { email: userEmail }, // ✅ Safe
});

// Good: Query builder with parameters
await this.repository
  .createQueryBuilder('user')
  .where('user.email = :email', { email: userEmail }) // ✅ Safe
  .getOne();

// Bad: String concatenation
await this.repository.query(
  `SELECT * FROM users WHERE email = '${userEmail}'` // ❌ Vulnerable
);
```

---

## XSS and CSRF Protection

### Helmet Middleware

```bash
npm install helmet
```

```typescript
// main.ts
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Helmet sets security headers
  app.use(helmet());
  
  await app.listen(3000);
}
```

### CSRF Protection

```bash
npm install csurf
```

```typescript
import * as csurf from 'csurf';

app.use(csurf());
```

---

## Performance Optimization

### 1. Database Query Optimization

```typescript
// Good: Select only needed fields
await this.repository.find({
  select: ['id', 'email', 'firstName'], // Only fetch needed columns
});

// Good: Use relations efficiently
await this.repository.find({
  relations: ['profile'], // Eager load if needed
  where: { isActive: true },
});

// Good: Use indexes
// Add @Index() decorator on frequently queried columns
@Entity()
@Index(['email'])
export class User {
  @Column()
  email: string;
}
```

### 2. Connection Pooling

```typescript
// Configure connection pool
TypeOrmModule.forRoot({
  // ... other config
  extra: {
    max: 20, // Maximum connections
    min: 5, // Minimum connections
    idleTimeoutMillis: 30000,
  },
})
```

### 3. Lazy Loading Prevention

```typescript
// Load relations eagerly when needed
const users = await this.repository.find({
  relations: ['posts', 'profile'],
});
```

---

## Database Optimization

### Query Optimization

```typescript
// Use indexes on frequently queried columns
@Index(['createdAt', 'status'])
@Entity()
export class Order {
  @Column()
  createdAt: Date;

  @Column()
  status: string;
}

// Use pagination
async findAll(page: number = 1, limit: number = 10) {
  return this.repository.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
  });
}
```

### Connection Pooling

Configure appropriate pool sizes based on your database capacity.

---

## Caching Strategies

### Response Caching

```typescript
// Cache controller responses
@Controller('products')
export class ProductsController {
  @Get()
  @Header('Cache-Control', 'public, max-age=3600') // 1 hour
  async findAll() {
    return this.productsService.findAll();
  }
}
```

### Service-Level Caching

Use Redis for distributed caching (covered in Module 9).

---

## Load Testing

### Artillery Setup

```bash
npm install -D artillery
```

```yaml
# load-test.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
scenarios:
  - name: "Get products"
    flow:
      - get:
          url: "/api/products"
      - think: 1
```

### Run Load Test

```bash
artillery run load-test.yml
```

---

## Best Practices

### Security

1. **Validate all input**
2. **Use parameterized queries**
3. **Hash passwords with bcrypt**
4. **Use HTTPS**
5. **Implement rate limiting**
6. **Keep dependencies updated**
7. **Use security headers (Helmet)**
8. **Store secrets securely**

### Performance

1. **Use database indexes**
2. **Implement caching**
3. **Optimize queries**
4. **Use connection pooling**
5. **Enable compression**
6. **Use CDN for static assets**
7. **Monitor performance metrics**

---

## Next Steps

✅ Secure your applications  
✅ Optimize performance  
✅ Implement rate limiting  
✅ Use proper caching  
✅ Monitor and test performance  
✅ Move to Module 24: Production Deployment  

---

*You now know how to secure and optimize your applications! 🔒⚡*

