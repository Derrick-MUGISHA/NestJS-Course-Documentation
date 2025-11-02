# Module 6: Authentication & Authorization

## 📚 Table of Contents
1. [Overview](#overview)
2. [Password Hashing](#password-hashing)
3. [JWT Authentication](#jwt-authentication)
4. [Passport.js Integration](#passportjs-integration)
5. [Local Strategy](#local-strategy)
6. [JWT Strategy](#jwt-strategy)
7. [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
8. [Refresh Tokens](#refresh-tokens)
9. [Password Reset](#password-reset)
10. [Session Management](#session-management)
11. [Best Practices](#best-practices)
12. [Mini Project: Complete Auth System](#mini-project-complete-auth-system)

---

## Overview

Authentication and authorization are critical for securing your applications. This module covers implementing a complete authentication system using JWT tokens, Passport.js strategies, and role-based access control.

---

## Password Hashing

### Install bcrypt

```bash
npm install bcrypt
npm install -D @types/bcrypt
```

### Hash Passwords

**Why Hash Passwords?**
Passwords should NEVER be stored in plain text. If your database is compromised, attackers would have access to all user passwords. Hashing converts passwords into a fixed-length string that cannot be reversed. Even if the database is breached, attackers cannot retrieve the original passwords.

**How bcrypt Works:**
- Takes a password and a "salt rounds" value (complexity factor)
- Generates a unique salt for each password
- Hashes the password + salt combination
- The salt rounds determine how many times the hash is computed (10 = 2^10 iterations)
- Higher rounds = more secure but slower

```typescript
// src/auth/auth.service.ts
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  // saltRounds: The cost factor for bcrypt (2^10 = 1024 iterations)
  // Higher values = more secure but slower
  // 10 is a good balance (takes ~100ms, secure enough)
  // In production, consider 12-13 for better security if performance allows
  private readonly saltRounds = 10;

  // hashPassword: Converts a plain text password into a secure hash
  // This should be called when creating or updating a user password
  // The returned hash can be safely stored in the database
  async hashPassword(password: string): Promise<string> {
    // bcrypt.hash() is asynchronous because hashing is CPU-intensive
    // Always use await to ensure the hash is generated before continuing
    return await bcrypt.hash(password, this.saltRounds);
  }

  // comparePassword: Verifies if a plain text password matches a hashed password
  // This is used during login to verify the user's password
  // bcrypt automatically extracts the salt from the hash and compares
  async comparePassword(
    plainPassword: string,
    hashedPassword: string,
  ): Promise<boolean> {
    // Returns true if passwords match, false otherwise
    // This comparison is secure against timing attacks
    return await bcrypt.compare(plainPassword, hashedPassword);
  }
}
```

**Security Best Practices:**
- Always hash passwords before storing them
- Use at least 10 salt rounds (consider 12-13 for high-security applications)
- Never store plain text passwords anywhere (logs, error messages, etc.)
- bcrypt handles salt generation automatically - don't try to manually salt

---

## JWT Authentication

### Install Dependencies

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install -D @types/passport-jwt
```

### Configure JWT Module

This module configuration sets up JWT (JSON Web Token) authentication. JWT tokens are stateless authentication tokens that contain user information. When a user logs in, they receive a token that they must include in subsequent requests.

**How JWT Works:**
1. User logs in with credentials
2. Server validates credentials and generates a JWT token
3. Token is sent to client
4. Client includes token in Authorization header for future requests
5. Server validates token on each request

**Why JWT?**
- Stateless: Server doesn't need to store session data
- Scalable: Works across multiple servers
- Secure: Tokens are signed and can be verified
- Self-contained: User info is in the token itself

```typescript
// src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { LocalStrategy } from './strategies/local.strategy';

@Module({
  imports: [
    // PassportModule provides authentication strategies
    // It's a base module that strategies extend
    PassportModule,
    
    // JwtModule.registerAsync() allows async configuration
    // We use this to read JWT secret from environment variables
    JwtModule.registerAsync({
      imports: [ConfigModule], // Import to access ConfigService
      
      // useFactory creates the JWT configuration dynamically
      useFactory: async (configService: ConfigService) => ({
        // JWT_SECRET: Used to sign and verify tokens
        // MUST be kept secret and strong (min 32 characters)
        // Never commit this to version control!
        secret: configService.get<string>('JWT_SECRET'),
        
        signOptions: {
          // expiresIn: How long the token is valid
          // Formats: '1h' (1 hour), '7d' (7 days), '15m' (15 minutes)
          // Access tokens should be short-lived (15min-1h)
          expiresIn: configService.get<string>('JWT_EXPIRATION', '1h'),
        },
      }),
      
      // Inject ConfigService to use in useFactory
      inject: [ConfigService],
    }),
  ],
  
  // Strategies: Define how authentication works
  // LocalStrategy: Validates username/password
  // JwtStrategy: Validates JWT tokens
  providers: [AuthService, LocalStrategy, JwtStrategy],
  controllers: [AuthController],
  
  // Export AuthService so other modules can use it
  exports: [AuthService],
})
export class AuthModule {}
```

**Security Considerations:**
- JWT_SECRET must be strong and random (use crypto.randomBytes)
- Store secrets in environment variables, never in code
- Short expiration times reduce risk if token is compromised
- Use HTTPS in production to prevent token interception

### Auth Service

```typescript
// src/auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from '../users/entities/user.entity';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
    private jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string): Promise<any> {
    const user = await this.usersRepository.findOne({ where: { email } });
    
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);
    
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const { password: _, ...result } = user;
    return result;
  }

  async login(user: User) {
    const payload = { email: user.email, sub: user.id };
    return {
      access_token: this.jwtService.sign(payload),
      user: {
        id: user.id,
        email: user.email,
        firstName: user.firstName,
        lastName: user.lastName,
      },
    };
  }

  async register(createUserDto: CreateUserDto): Promise<User> {
    const hashedPassword = await bcrypt.hash(createUserDto.password, 10);
    
    const user = this.usersRepository.create({
      ...createUserDto,
      password: hashedPassword,
    });

    return await this.usersRepository.save(user);
  }
}
```

---

## Passport.js Integration

### Local Strategy (Email/Password)

The Local Strategy is used for traditional username/password (or email/password) authentication. It's called "local" because credentials are validated locally by your server, as opposed to OAuth strategies that validate with external providers (Google, Facebook, etc.).

**How It Works:**
1. Client sends email and password to login endpoint
2. `LocalAuthGuard` triggers this strategy
3. Strategy calls `validate()` method with credentials
4. `validate()` checks credentials against database
5. If valid, returns user object (attached to request as `req.user`)
6. If invalid, throws `UnauthorizedException`

```typescript
// src/auth/strategies/local.strategy.ts
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from '../auth.service';

// @Injectable() makes this class injectable for dependency injection
// extends PassportStrategy(Strategy) sets up the local authentication strategy
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  // Constructor injection: AuthService is provided by NestJS DI container
  constructor(private authService: AuthService) {
    // super() calls the parent class constructor with options
    // usernameField: 'email' tells Passport to look for 'email' field
    // instead of default 'username' field in the request body
    super({
      usernameField: 'email', // Use 'email' instead of 'username'
    });
  }

  // validate() is called automatically by Passport when LocalAuthGuard is used
  // Parameters match the request body fields (email, password by default)
  // This method MUST return a user object or throw UnauthorizedException
  async validate(email: string, password: string): Promise<any> {
    // Delegate to AuthService to validate credentials
    // This keeps business logic in the service, not the strategy
    const user = await this.authService.validateUser(email, password);
    
    // If user is not found or password is wrong, throw exception
    // This will result in a 401 Unauthorized response
    if (!user) {
      throw new UnauthorizedException();
    }
    
    // Return value becomes req.user in the route handler
    // This is how you access the authenticated user in controllers
    return user;
  }
}
```

**Key Points:**
- The returned user object is attached to `request.user` in controllers
- Throwing `UnauthorizedException` results in a 401 HTTP status
- The strategy name 'local' comes from Passport - it's automatically registered
- Always validate credentials in a service, not directly in the strategy

### JWT Strategy

```typescript
// src/auth/strategies/jwt.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from '../../users/entities/user.entity';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private configService: ConfigService,
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    const user = await this.usersRepository.findOne({
      where: { id: payload.sub },
    });

    if (!user) {
      throw new UnauthorizedException();
    }

    return user;
  }
}
```

---

## Authentication Controller

```typescript
// src/auth/auth.controller.ts
import {
  Controller,
  Post,
  Body,
  UseGuards,
  Request,
  Get,
} from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalAuthGuard } from './guards/local-auth.guard';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { CreateUserDto } from '../users/dto/create-user.dto';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  async register(@Body() createUserDto: CreateUserDto) {
    return await this.authService.register(createUserDto);
  }

  @UseGuards(LocalAuthGuard)
  @Post('login')
  async login(@Request() req, @Body() loginDto: LoginDto) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

### Guards

```typescript
// src/auth/guards/local-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}

// src/auth/guards/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

---

## Role-Based Access Control (RBAC)

### Role Enum

```typescript
// src/users/enums/role.enum.ts
export enum Role {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
  VENDOR = 'vendor',
}
```

### Role Decorator

```typescript
// src/auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

### Roles Guard

```typescript
// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
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

### Using RBAC

```typescript
// src/users/users.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles } from '../auth/decorators/roles.decorator';

@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)
export class UsersController {
  @Get()
  @Roles('admin', 'moderator')
  findAll() {
    return 'All users (Admin/Moderator only)';
  }

  @Get('profile')
  getProfile(@Request() req) {
    return req.user; // Any authenticated user
  }
}
```

---

## Refresh Tokens

### Token Entity

```typescript
// src/auth/entities/refresh-token.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from 'typeorm';
import { User } from '../../users/entities/user.entity';

@Entity('refresh_tokens')
export class RefreshToken {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  token: string;

  @Column()
  userId: number;

  @ManyToOne(() => User, { onDelete: 'CASCADE' })
  user: User;

  @Column({ type: 'timestamp' })
  expiresAt: Date;

  @Column({ default: false })
  isRevoked: boolean;
}
```

### Refresh Token Service

```typescript
// src/auth/auth.service.ts (add methods)
async generateRefreshToken(user: User): Promise<string> {
  const token = crypto.randomBytes(40).toString('hex');
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + 30); // 30 days

  await this.refreshTokenRepository.save({
    token,
    userId: user.id,
    expiresAt,
  });

  return token;
}

async refreshAccessToken(refreshToken: string) {
  const tokenEntity = await this.refreshTokenRepository.findOne({
    where: { token: refreshToken },
    relations: ['user'],
  });

  if (!tokenEntity || tokenEntity.isRevoked || tokenEntity.expiresAt < new Date()) {
    throw new UnauthorizedException('Invalid refresh token');
  }

  const payload = { email: tokenEntity.user.email, sub: tokenEntity.user.id };
  
  return {
    access_token: this.jwtService.sign(payload),
  };
}
```

### Refresh Token Endpoint

```typescript
@Post('refresh')
async refreshToken(@Body('refreshToken') refreshToken: string) {
  return await this.authService.refreshAccessToken(refreshToken);
}
```

---

## Password Reset

### Password Reset Token Entity

```typescript
// src/auth/entities/password-reset.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from 'typeorm';
import { User } from '../../users/entities/user.entity';

@Entity('password_resets')
export class PasswordReset {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  token: string;

  @Column()
  userId: number;

  @ManyToOne(() => User, { onDelete: 'CASCADE' })
  user: User;

  @Column({ type: 'timestamp' })
  expiresAt: Date;

  @Column({ default: false })
  isUsed: boolean;
}
```

### Password Reset Service

```typescript
// src/auth/auth.service.ts (add methods)
async requestPasswordReset(email: string): Promise<void> {
  const user = await this.usersRepository.findOne({ where: { email } });
  
  if (!user) {
    // Don't reveal if user exists or not
    return;
  }

  const token = crypto.randomBytes(32).toString('hex');
  const expiresAt = new Date();
  expiresAt.setHours(expiresAt.getHours() + 1); // 1 hour

  await this.passwordResetRepository.save({
    token,
    userId: user.id,
    expiresAt,
  });

  // Send email with reset link
  await this.emailService.sendPasswordResetEmail(user.email, token);
}

async resetPassword(token: string, newPassword: string): Promise<void> {
  const resetToken = await this.passwordResetRepository.findOne({
    where: { token },
    relations: ['user'],
  });

  if (
    !resetToken ||
    resetToken.isUsed ||
    resetToken.expiresAt < new Date()
  ) {
    throw new BadRequestException('Invalid or expired reset token');
  }

  const hashedPassword = await bcrypt.hash(newPassword, 10);
  
  await this.usersRepository.update(resetToken.userId, {
    password: hashedPassword,
  });

  await this.passwordResetRepository.update(resetToken.id, {
    isUsed: true,
  });
}
```

---

## Session Management

### Current User Decorator

```typescript
// src/auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);

// Usage
@Get('profile')
getProfile(@CurrentUser() user: User) {
  return user;
}
```

---

## Best Practices

### 1. Never Store Passwords in Plain Text

Always hash passwords:

```typescript
const hashedPassword = await bcrypt.hash(password, 10);
```

### 2. Use Strong JWT Secrets

```env
JWT_SECRET=your-super-secret-key-min-32-characters-long
```

### 3. Set Appropriate Token Expiration

```typescript
// Access tokens: short-lived (15min - 1h)
// Refresh tokens: long-lived (7-30 days)
```

### 4. Implement Token Revocation

Store refresh tokens in database for revocation capability.

### 5. Use HTTPS in Production

Always use HTTPS to protect tokens in transit.

### 6. Validate All Inputs

```typescript
@IsEmail()
email: string;

@MinLength(8)
password: string;
```

---

## Mini Project: Complete Auth System

Build a complete authentication system with:

1. User registration
2. Login with JWT
3. Refresh tokens
4. Password reset
5. Role-based access control
6. Protected routes

### Features to Implement:

- ✅ User registration with validation
- ✅ Login with email/password
- ✅ JWT access tokens
- ✅ Refresh token mechanism
- ✅ Password reset flow
- ✅ Role-based guards
- ✅ Current user decorator
- ✅ Protected endpoints

---

## Next Steps

✅ Complete authentication mini project  
✅ Understand JWT and Passport strategies  
✅ Implement role-based access control  
✅ Move to Module 7: APIs & Integration  

---

*You now have a secure authentication system! 🔒*

