# Module 5: Advanced NestJS Features

## 📚 Table of Contents
1. [Overview](#overview)
2. [Custom Decorators](#custom-decorators)
3. [Dynamic Modules](#dynamic-modules)
4. [Configuration Management](#configuration-management)
5. [Validation](#validation)
6. [Logging](#logging)
7. [Module Reference](#module-reference)
8. [Reflection and Metadata](#reflection-and-metadata)
9. [Custom Providers](#custom-providers)
10. [Lifecycle Hooks](#lifecycle-hooks)
11. [Best Practices](#best-practices)

---

## Overview

This module covers advanced NestJS features that enable you to build more flexible, maintainable, and powerful applications. You'll learn how to extend NestJS functionality through custom decorators, dynamic modules, and advanced configuration patterns.

---

## Custom Decorators

### Method Decorators

```typescript
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Usage in controller
@Controller('users')
export class UsersController {
  @Get('admin')
  @Roles('admin', 'super-admin')
  adminOnly() {
    return 'Admin access';
  }
}
```

### Parameter Decorators

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);

// Usage
@Controller('profile')
export class ProfileController {
  @Get()
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

### Property Decorators

```typescript
// src/common/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// Usage
@Controller('auth')
export class AuthController {
  @Public()
  @Get('login')
  login() {
    return 'Login page';
  }
}
```

### Complex Parameter Decorator

```typescript
// src/common/decorators/cookie.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookie = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return data ? request.cookies?.[data] : request.cookies;
  },
);

// Usage
@Get('profile')
getProfile(@Cookie('sessionId') sessionId: string) {
  return { sessionId };
}
```

---

## Dynamic Modules

### Creating Dynamic Modules

```typescript
// src/config/config.module.ts
import { DynamicModule, Module } from '@nestjs/common';

export interface ConfigModuleOptions {
  folder: string;
}

@Module({})
export class ConfigModule {
  static forRoot(options: ConfigModuleOptions): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }

  static forRootAsync(options: {
    useFactory: (...args: any[]) => Promise<ConfigModuleOptions> | ConfigModuleOptions;
    inject?: any[];
  }): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useFactory: options.useFactory,
          inject: options.inject || [],
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}

// Usage
@Module({
  imports: [
    ConfigModule.forRoot({
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

### Async Dynamic Module

```typescript
// src/database/database.module.ts
import { DynamicModule, Module } from '@nestjs/common';
import { TypeOrmModule, TypeOrmModuleOptions } from '@nestjs/typeorm';
import { ConfigService } from '@nestjs/config';

@Module({})
export class DatabaseModule {
  static forRootAsync(): DynamicModule {
    return {
      module: DatabaseModule,
      imports: [
        TypeOrmModule.forRootAsync({
          inject: [ConfigService],
          useFactory: (configService: ConfigService): TypeOrmModuleOptions => ({
            type: 'postgres',
            host: configService.get('DB_HOST'),
            port: configService.get('DB_PORT'),
            username: configService.get('DB_USERNAME'),
            password: configService.get('DB_PASSWORD'),
            database: configService.get('DB_NAME'),
            entities: [__dirname + '/../**/*.entity{.ts,.js}'],
            synchronize: configService.get('NODE_ENV') === 'development',
          }),
        }),
      ],
      exports: [TypeOrmModule],
    };
  }
}
```

---

## Configuration Management

### Using @nestjs/config

```bash
npm install @nestjs/config
```

### Basic Configuration

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // Available in all modules
      envFilePath: '.env', // Path to .env file
      ignoreEnvFile: false, // Set to true in production
    }),
  ],
})
export class AppModule {}
```

### Configuration Service

```typescript
// src/config/config.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AppConfigService {
  constructor(private configService: ConfigService) {}

  get databaseHost(): string {
    return this.configService.get<string>('DB_HOST', 'localhost');
  }

  get databasePort(): number {
    return this.configService.get<number>('DB_PORT', 5432);
  }

  get jwtSecret(): string {
    return this.configService.get<string>('JWT_SECRET');
  }

  get isDevelopment(): boolean {
    return this.configService.get<string>('NODE_ENV') === 'development';
  }
}
```

### Custom Configuration Files

```typescript
// src/config/database.config.ts
export default () => ({
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    name: process.env.DB_NAME,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRATION || '1h',
  },
});

// In app.module.ts
ConfigModule.forRoot({
  load: [databaseConfig],
  isGlobal: true,
});
```

### Schema Validation

```typescript
// src/config/config.schema.ts
import * as Joi from 'joi';

export const configValidationSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'production', 'test')
    .default('development'),
  PORT: Joi.number().default(3000),
  DB_HOST: Joi.string().required(),
  DB_PORT: Joi.number().default(5432),
  DB_USERNAME: Joi.string().required(),
  DB_PASSWORD: Joi.string().required(),
  DB_NAME: Joi.string().required(),
  JWT_SECRET: Joi.string().required(),
});

// In app.module.ts
ConfigModule.forRoot({
  validationSchema: configValidationSchema,
});
```

---

## Validation

### Using class-validator

```bash
npm install class-validator class-transformer
```

### DTOs with Validation

```typescript
// src/users/dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  IsOptional,
  Matches,
} from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(50)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase, and number',
  })
  password: string;

  @IsString()
  @MinLength(2)
  @MaxLength(50)
  firstName: string;

  @IsString()
  @MinLength(2)
  @MaxLength(50)
  lastName: string;

  @IsOptional()
  @IsString()
  phone?: string;
}
```

### Validation Pipe Configuration

```typescript
// src/main.ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // Strip properties that don't have decorators
      forbidNonWhitelisted: true, // Throw error for unknown properties
      transform: true, // Automatically transform payloads to DTOs
      transformOptions: {
        enableImplicitConversion: true, // Enable type conversion
      },
      disableErrorMessages: false,
      validationError: {
        target: false, // Don't expose target object in error
        value: false, // Don't expose value in error
      },
    }),
  );
  
  await app.listen(3000);
}
bootstrap();
```

### Custom Validators

```typescript
// src/common/validators/match.decorator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidationArguments,
} from 'class-validator';

export function Match(property: string, validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      name: 'match',
      target: object.constructor,
      propertyName: propertyName,
      constraints: [property],
      options: validationOptions,
      validator: {
        validate(value: any, args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          const relatedValue = (args.object as any)[relatedPropertyName];
          return value === relatedValue;
        },
        defaultMessage(args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          return `${args.property} must match ${relatedPropertyName}`;
        },
      },
    });
  };
}

// Usage
export class ChangePasswordDto {
  @IsString()
  @MinLength(8)
  password: string;

  @Match('password', { message: 'Passwords do not match' })
  confirmPassword: string;
}
```

---

## Logging

### Built-in Logger

```typescript
// src/users/users.service.ts
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  create(userData: CreateUserDto) {
    this.logger.log(`Creating user with email: ${userData.email}`);
    
    try {
      // Create user logic
      this.logger.debug('User created successfully');
      return user;
    } catch (error) {
      this.logger.error(`Failed to create user: ${error.message}`, error.stack);
      throw error;
    }
  }

  findAll() {
    this.logger.verbose('Fetching all users');
    return this.users;
  }
}
```

### Custom Logger

```typescript
// src/common/logger/logger.service.ts
import { Injectable, LoggerService } from '@nestjs/common';

@Injectable()
export class MyLogger implements LoggerService {
  log(message: string) {
    console.log(`[LOG] ${new Date().toISOString()} - ${message}`);
  }

  error(message: string, trace: string) {
    console.error(`[ERROR] ${new Date().toISOString()} - ${message}`, trace);
  }

  warn(message: string) {
    console.warn(`[WARN] ${new Date().toISOString()} - ${message}`);
  }

  debug(message: string) {
    console.debug(`[DEBUG] ${new Date().toISOString()} - ${message}`);
  }

  verbose(message: string) {
    console.log(`[VERBOSE] ${new Date().toISOString()} - ${message}`);
  }
}

// Usage in app.module.ts
import { Module } from '@nestjs/common';
import { MyLogger } from './common/logger/logger.service';

@Module({
  providers: [MyLogger],
})
export class AppModule {}
```

### Global Logger

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MyLogger } from './common/logger/logger.service';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: new MyLogger(),
  });
  
  await app.listen(3000);
}
bootstrap();
```

---

## Module Reference

### Getting Module Reference

```typescript
// src/users/users.service.ts
import { Injectable, ModuleRef } from '@nestjs/core';
import { CatsService } from '../cats/cats.service';

@Injectable()
export class UsersService {
  constructor(private moduleRef: ModuleRef) {}

  async getCatsService(): Promise<CatsService> {
    // Get service dynamically
    return this.moduleRef.get(CatsService, { strict: false });
  }

  async getCatsServiceLazy(): Promise<CatsService> {
    // Get service lazily (creates instance if doesn't exist)
    return this.moduleRef.resolve(CatsService);
  }
}
```

---

## Reflection and Metadata

### Using Reflector

```typescript
// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(Roles, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

---

## Custom Providers

### Factory Provider

```typescript
// src/database/database.provider.ts
import { Provider } from '@nestjs/common';

export const DATABASE_CONNECTION = 'DATABASE_CONNECTION';

export const databaseProvider: Provider = {
  provide: DATABASE_CONNECTION,
  useFactory: async () => {
    // Create database connection
    const connection = await createConnection({
      // connection config
    });
    return connection;
  },
};

// Usage in module
@Module({
  providers: [databaseProvider],
  exports: [databaseProvider],
})
export class DatabaseModule {}
```

### Value Provider

```typescript
// src/config/constants.ts
export const API_KEY = 'API_KEY';

// In module
@Module({
  providers: [
    {
      provide: API_KEY,
      useValue: process.env.API_KEY,
    },
  ],
})
export class ConfigModule {}
```

### Class Provider

```typescript
// src/common/interfaces/config.interface.ts
export interface IConfigService {
  get(key: string): string;
}

// src/common/services/config.service.ts
@Injectable()
export class ConfigService implements IConfigService {
  get(key: string): string {
    return process.env[key];
  }
}

// Usage
@Module({
  providers: [
    {
      provide: IConfigService,
      useClass: ConfigService,
    },
  ],
})
export class ConfigModule {}
```

---

## Lifecycle Hooks

### Module Lifecycle

```typescript
import { Module, OnModuleInit, OnModuleDestroy } from '@nestjs/common';

@Module({})
export class DatabaseModule implements OnModuleInit, OnModuleDestroy {
  onModuleInit() {
    console.log('Database module initialized');
    // Initialize database connection
  }

  onModuleDestroy() {
    console.log('Database module destroyed');
    // Close database connection
  }
}
```

### Application Lifecycle

```typescript
// src/app.service.ts
import { Injectable, OnApplicationBootstrap, OnApplicationShutdown } from '@nestjs/common';

@Injectable()
export class AppService implements OnApplicationBootstrap, OnApplicationShutdown {
  onApplicationBootstrap() {
    console.log('Application bootstrapped');
    // Setup logic
  }

  onApplicationShutdown(signal?: string) {
    console.log(`Application shutting down with signal: ${signal}`);
    // Cleanup logic
  }
}
```

---

## Best Practices

### 1. Use DTOs Everywhere

Always validate incoming data:

```typescript
@Post()
create(@Body() createDto: CreateUserDto) {
  // DTO is automatically validated
}
```

### 2. Create Reusable Decorators

Build a library of custom decorators:

```typescript
// Common decorators
@CurrentUser()
@Roles('admin')
@Public()
@Cookie('sessionId')
```

### 3. Use Configuration Service

Never hardcode values:

```typescript
// Good
const port = this.configService.get<number>('PORT');

// Bad
const port = 3000;
```

### 4. Implement Proper Logging

Log at appropriate levels:

- `log`: General information
- `error`: Errors that need attention
- `warn`: Warnings
- `debug`: Debug information
- `verbose`: Verbose information

### 5. Use Dynamic Modules for Flexibility

Create reusable modules that can be configured:

```typescript
ConfigModule.forRoot({ folder: './config' })
DatabaseModule.forRootAsync({ ... })
```

---

## Next Steps

✅ Master custom decorators  
✅ Understand dynamic modules  
✅ Implement proper validation  
✅ Set up logging  
✅ Move to Module 6: Authentication & Authorization  

---

*You're now ready for advanced NestJS development! 🚀*

