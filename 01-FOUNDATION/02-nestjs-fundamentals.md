# Module 2: NestJS Fundamentals

## 📚 Table of Contents
1. [Overview](#overview)
2. [NestJS Architecture](#nestjs-architecture)
3. [Controllers](#controllers)
4. [Providers & Services](#providers--services)
5. [Modules](#modules)
6. [Dependency Injection](#dependency-injection)
7. [Pipes](#pipes)
8. [Guards](#guards)
9. [Interceptors](#interceptors)
10. [Exception Filters](#exception-filters)
11. [Middleware](#middleware)
12. [Mini Project: Todo API](#mini-project-todo-api)
13. [Best Practices](#best-practices)

---

## Overview

NestJS is a progressive Node.js framework for building efficient and scalable server-side applications. It's built with TypeScript and uses modern JavaScript, combined with elements of OOP (Object-Oriented Programming), FP (Functional Programming), and FRP (Functional Reactive Programming).

### Key Features:
- **Built with TypeScript** - Type safety out of the box
- **Modular Architecture** - Organize code into modules
- **Dependency Injection** - Built-in DI container
- **Decorators** - Express your intent with decorators
- **Exception Filters** - Handle errors elegantly
- **Pipes** - Transform and validate data
- **Guards** - Implement authentication/authorization
- **Interceptors** - Add cross-cutting concerns

---

## NestJS Architecture

### Core Concepts:

```
┌─────────────────────────────────────────┐
│            Client Request               │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│            Middleware                    │
│  (CORS, Body Parser, etc.)              │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│            Guards                        │
│  (Authentication, Authorization)         │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│        Interceptors (Before)             │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│            Pipes                         │
│  (Validation, Transformation)           │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│          Controller                     │
│  (Route Handler)                         │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│            Service                       │
│  (Business Logic)                        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│        Interceptors (After)              │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Exception Filters                │
│  (Error Handling)                        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│          Client Response                 │
└─────────────────────────────────────────┘
```

---

## Controllers

Controllers are responsible for handling incoming requests and returning responses to the client. They act as the entry point to your application, defining the routes and HTTP methods that your API will expose.

**Key Points:**
- Controllers are classes decorated with `@Controller()`
- The decorator takes a path prefix (e.g., `'cats'`) that will be prepended to all routes in this controller
- Route handlers are methods decorated with HTTP method decorators (`@Get()`, `@Post()`, etc.)
- Each route handler can access request data through parameter decorators

### Basic Controller

Here's a complete example of a controller that handles CRUD operations for cats. Notice how each HTTP method decorator corresponds to a specific operation:

```typescript
// src/cats/cats.controller.ts
import { Controller, Get, Post, Body, Param, Put, Delete } from '@nestjs/common';

// @Controller('cats') - This decorator defines the route prefix for all routes in this controller
// All routes in this controller will be accessible under /cats
@Controller('cats')
export class CatsController {
  // @Get() decorator handles GET requests
  // This route will be accessible at GET /cats
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }

  // @Get(':id') - The colon (:) indicates a route parameter
  // This route will be accessible at GET /cats/123 (where 123 is the id)
  // The @Param('id') decorator extracts the id from the URL
  @Get(':id')
  findOne(@Param('id') id: string): string {
    return `This action returns a #${id} cat`;
  }

  // @Post() handles POST requests
  // @Body() decorator extracts the request body
  // This route will be accessible at POST /cats
  @Post()
  create(@Body() createCatDto: any): string {
    return 'This action adds a new cat';
  }

  // @Put(':id') handles PUT requests for updating
  // This route will be accessible at PUT /cats/123
  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: any): string {
    return `This action updates a #${id} cat`;
  }

  // @Delete(':id') handles DELETE requests
  // This route will be accessible at DELETE /cats/123
  @Delete(':id')
  remove(@Param('id') id: string): string {
    return `This action removes a #${id} cat`;
  }
}
```

**Understanding the Pattern:**
- The `@Controller('cats')` decorator creates a route prefix, so all routes in this controller automatically start with `/cats`
- Route parameter decorators like `@Param()` and `@Body()` automatically extract and inject values from the request
- The return value of each handler method becomes the HTTP response body
- NestJS automatically serializes the response to JSON for objects

### Request Object Decorators

NestJS provides several decorators to extract data from HTTP requests. These decorators make it easy to access different parts of the request without manually parsing the request object. Here's how to use each one:

**Important Note:** When using `@Res()` decorator, you must manually send the response. NestJS won't automatically serialize the return value in this case.

```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  Query,
  Headers,
  Req,
  Res,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Controller('example')
export class ExampleController {
  // @Query() decorator extracts query string parameters
  // Example: GET /example/search?q=nestjs&page=1
  // query will be "nestjs", page will be 1
  // You can extract individual parameters by name, or get all with @Query()
  @Get('search')
  search(@Query('q') query: string, @Query('page') page: number) {
    return { query, page };
  }

  // @Headers() decorator extracts specific request headers
  // Example: GET /example/headers with header "authorization: Bearer token123"
  // auth will be "Bearer token123"
  @Get('headers')
  getHeaders(@Headers('authorization') auth: string) {
    return { auth };
  }

  // @Req() gives you access to the entire Express request object
  // Use this when you need access to multiple request properties
  // Example use cases: checking IP address, accessing cookies, etc.
  @Get('request')
  getRequest(@Req() request: Request) {
    return {
      method: request.method,
      url: request.url,
      headers: request.headers,
    };
  }

  // @Res() gives you access to the Express response object
  // WARNING: When using @Res(), you must manually call response methods
  // NestJS won't automatically handle the response for you
  // Use this when you need full control (streaming, custom headers, etc.)
  @Post('response')
  setResponse(@Res() response: Response) {
    return response.status(201).json({ message: 'Created' });
  }
}
```

**Best Practice:** Prefer using specific decorators like `@Query()` and `@Headers()` over `@Req()` for better code clarity and testability. Only use `@Req()` or `@Res()` when you need advanced functionality.

### Route Parameters

```typescript
@Controller('users')
export class UsersController {
  // Single parameter
  @Get(':id')
  findOne(@Param('id') id: string) {
    return { id };
  }

  // Multiple parameters
  @Get(':userId/posts/:postId')
  findPost(
    @Param('userId') userId: string,
    @Param('postId') postId: string,
  ) {
    return { userId, postId };
  }

  // All parameters
  @Get(':id/details')
  findDetails(@Param() params: { id: string }) {
    return params;
  }
}
```

### Status Codes

```typescript
import { HttpCode, HttpStatus } from '@nestjs/common';

@Controller('items')
export class ItemsController {
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createItemDto: any) {
    return 'Item created';
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id') id: string) {
    // No content response
  }
}
```

---

## Providers & Services

Services are providers that contain business logic. They can be injected into controllers or other services. This separation of concerns is a key principle in NestJS: controllers handle HTTP requests/responses, while services handle business logic.

**Why Use Services?**
- **Separation of Concerns**: Controllers stay thin, focusing only on HTTP handling
- **Reusability**: Business logic can be shared across multiple controllers
- **Testability**: Services can be easily unit tested without HTTP concerns
- **Dependency Injection**: Services can be injected and mocked easily

### Basic Service

The `@Injectable()` decorator tells NestJS that this class can be injected as a dependency. This is essential for NestJS's dependency injection system to work. Here's a complete service with CRUD operations:

```typescript
// src/cats/cats.service.ts
import { Injectable } from '@nestjs/common';

// Define the data structure for a Cat
// Using an interface ensures type safety throughout your application
export interface Cat {
  id: number;
  name: string;
  breed: string;
  age: number;
}

// @Injectable() decorator makes this class a provider that can be injected
// This is the foundation of NestJS dependency injection
@Injectable()
export class CatsService {
  // private readonly ensures this array cannot be reassigned
  // In a real app, this would be a database connection
  // For now, we're using an in-memory array for simplicity
  private readonly cats: Cat[] = [];

  // Create operation: adds a new cat to the array
  // Returns the created cat (with any auto-generated fields like id)
  create(cat: Cat): Cat {
    this.cats.push(cat);
    return cat;
  }

  // Read operation: retrieves all cats
  // Returns a copy of the array (consider using spread operator for immutability)
  findAll(): Cat[] {
    return this.cats;
  }

  // Read operation: finds a single cat by id
  // Uses Array.find() which returns undefined if not found
  // In a real app, you'd want to throw an exception if not found
  findOne(id: number): Cat {
    return this.cats.find((cat) => cat.id === id);
  }

  // Update operation: partially updates a cat's properties
  // Partial<Cat> means all properties are optional
  // Uses spread operator to merge existing and new properties
  update(id: number, updateCat: Partial<Cat>): Cat {
    const index = this.cats.findIndex((cat) => cat.id === id);
    if (index !== -1) {
      // Spread operator merges existing cat with updates
      this.cats[index] = { ...this.cats[index], ...updateCat };
      return this.cats[index];
    }
    throw new Error('Cat not found');
  }

  // Delete operation: removes a cat from the array
  // Uses Array.splice() to remove the element at the found index
  remove(id: number): void {
    const index = this.cats.findIndex((cat) => cat.id === id);
    if (index !== -1) {
      this.cats.splice(index, 1);
    }
  }
}
```

**Key Concepts Explained:**
- `@Injectable()`: This decorator is required for dependency injection to work
- `private readonly`: Prevents external modification while keeping data encapsulated
- `Partial<Cat>`: TypeScript utility type that makes all properties optional, perfect for updates
- The service doesn't know about HTTP - it's pure business logic, making it highly testable

### Using Service in Controller

This example demonstrates the **dependency injection pattern** in NestJS. Notice how the service is injected through the constructor, and NestJS automatically provides the instance. This is the magic of NestJS's dependency injection container.

**How Dependency Injection Works:**
1. NestJS reads the constructor parameters
2. It looks for providers registered in the module
3. It creates or retrieves an instance
4. It injects it into the controller

```typescript
// src/cats/cats.controller.ts
import { Controller, Get, Post, Body, Param, Put, Delete } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './cats.service';

@Controller('cats')
export class CatsController {
  // Constructor injection: NestJS automatically provides CatsService
  // The 'private readonly' syntax is TypeScript shorthand that:
  // 1. Declares the property
  // 2. Sets it as private (only accessible within this class)
  // 3. Marks it as readonly (cannot be reassigned)
  // 4. Injects the service instance
  constructor(private readonly catsService: CatsService) {}

  // POST /cats - Create a new cat
  // @Body() extracts the request body and validates it against the Cat type
  // The controller delegates business logic to the service
  @Post()
  create(@Body() createCatDto: Cat): Cat {
    return this.catsService.create(createCatDto);
  }

  // GET /cats - Get all cats
  // Simple delegation: controller receives HTTP request, service handles logic
  @Get()
  findAll(): Cat[] {
    return this.catsService.findAll();
  }

  // GET /cats/:id - Get a single cat
  // +id converts string to number (TypeScript unary plus operator)
  // This is necessary because URL parameters are always strings
  @Get(':id')
  findOne(@Param('id') id: string): Cat {
    return this.catsService.findOne(+id);
  }

  // PUT /cats/:id - Update a cat
  // Partial<Cat> allows updating only some properties
  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: Partial<Cat>): Cat {
    return this.catsService.update(+id, updateCatDto);
  }

  // DELETE /cats/:id - Delete a cat
  @Delete(':id')
  remove(@Param('id') id: string): void {
    return this.catsService.remove(+id);
  }
}
```

**Understanding the Pattern:**
- The controller is **thin** - it only handles HTTP concerns
- Business logic lives in the service
- The `+id` syntax converts string parameters to numbers (you could also use `parseInt(id, 10)`)
- Each method returns the result from the service, which NestJS automatically serializes to JSON
- This separation makes both the controller and service easily testable in isolation

### Custom Providers

```typescript
// src/config/database.provider.ts
import { Provider } from '@nestjs/common';

export const DATABASE_CONNECTION = 'DATABASE_CONNECTION';

export const databaseProvider: Provider = {
  provide: DATABASE_CONNECTION,
  useValue: {
    host: 'localhost',
    port: 5432,
    // ... other config
  },
};
```

```typescript
// Using in module
import { Module } from '@nestjs/common';
import { databaseProvider } from './database.provider';

@Module({
  providers: [databaseProvider],
  exports: [databaseProvider],
})
export class DatabaseModule {}
```

---

## Modules

Modules are used to organize the application into logical units. Each NestJS application has at least one root module.

### Root Module

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  imports: [],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService], // Export to be used in other modules
})
export class AppModule {}
```

### Feature Module

```typescript
// src/cats/cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

### Global Module

```typescript
// src/config/config.module.ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Global() // Makes module available everywhere without importing
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}
```

### Dynamic Module

```typescript
// src/database/database.module.ts
import { Module, DynamicModule } from '@nestjs/common';

interface DatabaseModuleOptions {
  host: string;
  port: number;
}

@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseModuleOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
      ],
      exports: ['DATABASE_OPTIONS'],
    };
  }
}

// Usage
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432,
    }),
  ],
})
export class AppModule {}
```

---

## Dependency Injection

NestJS has a built-in Dependency Injection container that manages dependencies.

### Constructor Injection

```typescript
@Controller('users')
export class UsersController {
  constructor(
    private readonly usersService: UsersService,
    private readonly emailService: EmailService,
  ) {}
}
```

### Property Injection (Less Common)

```typescript
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  @Inject('CONFIG_OPTIONS')
  private configOptions: any;
}
```

### Optional Injection

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class UsersService {
  constructor(
    @Optional()
    @Inject('HTTP_OPTIONS')
    private httpOptions: any,
  ) {}
}
```

### Custom Provider with useFactory

```typescript
import { Module } from '@nestjs/common';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useFactory: async () => {
        // Async initialization
        return await createConnection();
      },
    },
  ],
})
export class DatabaseModule {}
```

---

## Pipes

Pipes transform or validate input data before it reaches the route handler.

### Built-in Pipes

```typescript
import { Controller, Get, Param, Query, Body, ParseIntPipe, ParseBoolPipe, ParseEnumPipe } from '@nestjs/common';

enum Status {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
}

@Controller('example')
export class ExampleController {
  // Parse to integer
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return { id, type: typeof id }; // id is a number
  }

  // Parse to boolean
  @Get('status')
  getStatus(@Query('active', ParseBoolPipe) active: boolean) {
    return { active, type: typeof active };
  }

  // Parse enum
  @Get('filter/:status')
  filter(@Param('status', new ParseEnumPipe(Status)) status: Status) {
    return { status };
  }
}
```

### Custom Validation Pipe

```typescript
// src/common/pipes/validation.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if (!value) {
      throw new BadRequestException('Value is required');
    }
    return value;
  }
}
```

### Global Pipe

```typescript
// src/main.ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global validation pipe
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true, // Strip unknown properties
    forbidNonWhitelisted: true, // Throw error for unknown properties
    transform: true, // Auto-transform payloads
  }));
  
  await app.listen(3000);
}
bootstrap();
```

---

## Guards

Guards determine whether a request should be handled by the route handler.

### Authentication Guard

```typescript
// src/auth/guards/auth.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const authHeader = request.headers.authorization;
    
    if (!authHeader) {
      throw new UnauthorizedException('No authorization header');
    }
    
    // Validate token logic here
    return true;
  }
}
```

### Role-Based Guard

```typescript
// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>(
      'roles',
      context.getHandler(),
    );
    
    if (!requiredRoles) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

### Using Guards

```typescript
import { Controller, Get, UseGuards, SetMetadata } from '@nestjs/common';
import { AuthGuard } from '../auth/guards/auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';

// Custom decorator for roles
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Controller('users')
@UseGuards(AuthGuard)
export class UsersController {
  @Get()
  findAll() {
    return 'All users';
  }

  @Get('admin')
  @UseGuards(RolesGuard)
  @Roles('admin')
  adminOnly() {
    return 'Admin only';
  }
}
```

---

## Interceptors

Interceptors can add logic before and after route handler execution.

### Logging Interceptor

```typescript
// src/common/interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    const request = context.switchToHttp().getRequest();
    
    console.log(`[${request.method}] ${request.url} - Before...`);
    
    return next.handle().pipe(
      tap(() => {
        console.log(`[${request.method}] ${request.url} - After ${Date.now() - now}ms`);
      }),
    );
  }
}
```

### Transform Response Interceptor

```typescript
// src/common/interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  success: boolean;
  data: T;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### Using Interceptors

```typescript
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { LoggingInterceptor } from '../common/interceptors/logging.interceptor';
import { TransformInterceptor } from '../common/interceptors/transform.interceptor';

@Controller('cats')
@UseInterceptors(TransformInterceptor)
export class CatsController {
  @Get()
  @UseInterceptors(LoggingInterceptor)
  findAll() {
    return ['cat1', 'cat2'];
  }
}
```

---

## Exception Filters

Exception filters catch exceptions and return appropriate responses.

### Custom Exception Filter

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const errorResponse = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message:
        typeof exceptionResponse === 'string'
          ? exceptionResponse
          : (exceptionResponse as any).message,
    };

    response.status(status).json(errorResponse);
  }
}
```

### Global Exception Filter

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { HttpExceptionFilter } from './common/filters/http-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

### Custom Exceptions

```typescript
// src/common/exceptions/forbidden.exception.ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}

// Usage
throw new ForbiddenException();
```

---

## Middleware

Middleware functions have access to the request, response, and the next middleware function.

### Class-Based Middleware

```typescript
// src/common/middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`Request ${req.method} ${req.url}`);
    next();
  }
}
```

### Functional Middleware

```typescript
// src/common/middleware/logger.middleware.ts
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request ${req.method} ${req.url}`);
  next();
}
```

### Applying Middleware

```typescript
// src/app.module.ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats'); // Apply to all /cats routes
      // .forRoutes(CatsController); // Or to specific controller
      // .forRoutes({ path: 'cats', method: RequestMethod.GET }); // Or specific route
  }
}
```

### Global Middleware

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as morgan from 'morgan';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(morgan('combined'));
  await app.listen(3000);
}
bootstrap();
```

---

## Mini Project: Todo API

Build a complete RESTful API for managing todos.

### Project Structure

```
todo-api/
├── src/
│   ├── todos/
│   │   ├── dto/
│   │   │   ├── create-todo.dto.ts
│   │   │   └── update-todo.dto.ts
│   │   ├── todos.controller.ts
│   │   ├── todos.service.ts
│   │   └── todos.module.ts
│   ├── common/
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts
│   │   └── interceptors/
│   │       └── transform.interceptor.ts
│   ├── app.module.ts
│   └── main.ts
```

### DTOs

```typescript
// src/todos/dto/create-todo.dto.ts
export class CreateTodoDto {
  title: string;
  description?: string;
  completed?: boolean;
}
```

```typescript
// src/todos/dto/update-todo.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateTodoDto } from './create-todo.dto';

export class UpdateTodoDto extends PartialType(CreateTodoDto) {}
```

### Service

```typescript
// src/todos/todos.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { CreateTodoDto } from './dto/create-todo.dto';
import { UpdateTodoDto } from './dto/update-todo.dto';

export interface Todo {
  id: number;
  title: string;
  description?: string;
  completed: boolean;
  createdAt: Date;
  updatedAt: Date;
}

@Injectable()
export class TodosService {
  private todos: Todo[] = [];
  private nextId = 1;

  create(createTodoDto: CreateTodoDto): Todo {
    const todo: Todo = {
      id: this.nextId++,
      ...createTodoDto,
      completed: createTodoDto.completed ?? false,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.todos.push(todo);
    return todo;
  }

  findAll(): Todo[] {
    return this.todos;
  }

  findOne(id: number): Todo {
    const todo = this.todos.find((t) => t.id === id);
    if (!todo) {
      throw new NotFoundException(`Todo with ID ${id} not found`);
    }
    return todo;
  }

  update(id: number, updateTodoDto: UpdateTodoDto): Todo {
    const todo = this.findOne(id);
    Object.assign(todo, updateTodoDto, { updatedAt: new Date() });
    return todo;
  }

  remove(id: number): void {
    const todo = this.findOne(id);
    const index = this.todos.findIndex((t) => t.id === id);
    this.todos.splice(index, 1);
  }
}
```

### Controller

```typescript
// src/todos/todos.controller.ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  ParseIntPipe,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { TodosService } from './todos.service';
import { CreateTodoDto } from './dto/create-todo.dto';
import { UpdateTodoDto } from './dto/update-todo.dto';

@Controller('todos')
export class TodosController {
  constructor(private readonly todosService: TodosService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createTodoDto: CreateTodoDto) {
    return this.todosService.create(createTodoDto);
  }

  @Get()
  findAll() {
    return this.todosService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.todosService.findOne(id);
  }

  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateTodoDto: UpdateTodoDto,
  ) {
    return this.todosService.update(id, updateTodoDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.todosService.remove(id);
  }
}
```

### Module

```typescript
// src/todos/todos.module.ts
import { Module } from '@nestjs/common';
import { TodosService } from './todos.service';
import { TodosController } from './todos.controller';

@Module({
  controllers: [TodosController],
  providers: [TodosService],
  exports: [TodosService],
})
export class TodosModule {}
```

### Main Application

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TodosModule } from './todos/todos.module';

@Module({
  imports: [TodosModule],
})
export class AppModule {}
```

---

## Best Practices

### 1. Organize Code by Features

```
src/
├── users/
│   ├── dto/
│   ├── users.controller.ts
│   ├── users.service.ts
│   └── users.module.ts
├── products/
└── orders/
```

### 2. Use DTOs for Data Transfer

Always use DTOs instead of accepting raw objects:

```typescript
// Good
@Post()
create(@Body() createUserDto: CreateUserDto) {}

// Bad
@Post()
create(@Body() user: any) {}
```

### 3. Keep Controllers Thin

Move business logic to services:

```typescript
// Good
@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}
  
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }
}

// Bad - business logic in controller
@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id') id: string) {
    // Complex business logic here - BAD!
    return users.find(u => u.id === id);
  }
}
```

### 4. Use Modules to Organize

Group related features in modules:

```typescript
@Module({
  imports: [DatabaseModule, ConfigModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

### 5. Handle Errors Properly

Use appropriate HTTP exceptions:

```typescript
import { NotFoundException, BadRequestException } from '@nestjs/common';

throw new NotFoundException('User not found');
throw new BadRequestException('Invalid input');
```

---

## Next Steps

✅ Complete the Todo API mini project  
✅ Experiment with guards, interceptors, and filters  
✅ Move to Module 3: PostgreSQL Basics  

---

*You now have a solid foundation in NestJS! 🎉*

