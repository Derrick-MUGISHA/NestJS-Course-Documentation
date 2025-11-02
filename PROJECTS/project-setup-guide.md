# Project Setup Guide

## 🚀 Main Project: E-Commerce Platform

This guide will help you set up the main project that we'll build throughout the training program.

---

## 📋 Project Overview

We'll build a comprehensive E-Commerce Platform with the following features:

- User Management (Authentication, Authorization, Profiles)
- Product Management (CRUD, Categories, Tags, Inventory)
- Shopping Cart & Checkout
- Order Management
- Vendor System (Multi-vendor support)
- Reviews & Ratings
- Notifications (Email, Real-time)
- Admin Dashboard
- Search & Recommendations
- Payment Integration

---

## 🛠️ Initial Setup

### Step 1: Create NestJS Project

```bash
# Create new NestJS project
nest new ecommerce-platform

# Navigate to project
cd ecommerce-platform

# Install additional dependencies
npm install @nestjs/config @nestjs/typeorm typeorm pg
npm install class-validator class-transformer
npm install bcrypt @nestjs/jwt @nestjs/passport passport passport-jwt
npm install --save-dev @types/passport-jwt @types/bcrypt
```

### Step 2: Project Structure

```
ecommerce-platform/
├── src/
│   ├── common/
│   │   ├── decorators/
│   │   ├── filters/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   └── pipes/
│   ├── config/
│   ├── database/
│   │   ├── migrations/
│   │   └── seeds/
│   ├── modules/
│   │   ├── auth/
│   │   ├── users/
│   │   ├── products/
│   │   ├── categories/
│   │   ├── orders/
│   │   ├── cart/
│   │   ├── vendors/
│   │   ├── reviews/
│   │   └── admin/
│   ├── app.module.ts
│   └── main.ts
├── test/
├── .env
├── .env.example
├── docker-compose.yml
├── Dockerfile
└── package.json
```

### Step 3: Environment Configuration

Create `.env` file:

```env
# Application
NODE_ENV=development
PORT=3000
API_PREFIX=api

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=your_password
DB_NAME=ecommerce_db

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRATION=7d
JWT_REFRESH_SECRET=your-refresh-secret
JWT_REFRESH_EXPIRATION=30d

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Email (for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-password

# Frontend URL (for CORS)
FRONTEND_URL=http://localhost:3001

# AWS S3 (for file uploads)
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_S3_BUCKET=your-bucket-name
```

### Step 4: Database Setup

```bash
# Create database
createdb ecommerce_db

# Or using psql
psql -U postgres
CREATE DATABASE ecommerce_db;
\q
```

### Step 5: Docker Setup (Optional but Recommended)

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    container_name: ecommerce_postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: your_password
      POSTGRES_DB: ecommerce_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: ecommerce_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

Start services:

```bash
docker-compose up -d
```

---

## 📦 Module-by-Module Implementation

### Phase 1: Foundation (Weeks 1-2)

#### Module 1: Setup ✅
- Complete project structure
- Configure TypeORM
- Set up environment variables

#### Module 2: NestJS Fundamentals ✅
- Create base controllers and services
- Implement common filters, interceptors
- Set up validation pipes

#### Module 3: PostgreSQL Basics ✅
- Design database schema
- Create initial migrations

### Phase 2: Core Development (Weeks 3-4)

#### Module 4: TypeORM Integration
- Create User entity
- Create Product entity
- Create Category entity
- Set up relationships

#### Module 5: Advanced Features
- Implement custom decorators
- Add configuration management
- Set up logging

#### Module 6: Authentication
- JWT implementation
- User registration/login
- Password reset
- Role-based access control

### Phase 3: Advanced Features (Weeks 5-6)

#### Module 7: APIs
- RESTful endpoints
- API documentation (Swagger)
- Rate limiting

#### Module 8: File Handling
- Product image upload
- AWS S3 integration

#### Module 9: Redis Caching
- Cache product listings
- Cache user sessions
- Implement cache invalidation

#### Module 10: Background Jobs
- Email notifications
- Order processing queues
- Inventory updates

---

## 🎯 Development Workflow

### Daily Workflow

1. **Morning**: Read module documentation
2. **Mid-morning**: Study code samples
3. **Afternoon**: Build mini project
4. **Evening**: Integrate into main project

### Git Workflow

```bash
# Create feature branch
git checkout -b feature/user-authentication

# Make changes and commit
git add .
git commit -m "feat: implement user authentication"

# Push to remote
git push origin feature/user-authentication

# Create pull request (if working in team)
```

### Testing Strategy

```bash
# Unit tests
npm run test

# E2E tests
npm run test:e2e

# Test coverage
npm run test:cov
```

---

## 📝 Code Style

Follow these conventions:

- **Controllers**: Thin, only handle HTTP requests
- **Services**: Business logic
- **DTOs**: Data validation and transformation
- **Entities**: Database schema
- **Guards**: Authentication/Authorization
- **Interceptors**: Cross-cutting concerns

---

## 🐛 Debugging Tips

### Database Queries

```typescript
// Enable query logging in development
{
  logging: true,
  logger: 'advanced-console',
}
```

### API Testing

Use tools like:
- Postman
- Insomnia
- VS Code REST Client extension
- Thunder Client

---

## 📚 Resources

- [NestJS Documentation](https://docs.nestjs.com/)
- [TypeORM Documentation](https://typeorm.io/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

*Happy coding! 🚀*

