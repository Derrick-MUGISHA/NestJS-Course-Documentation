# Module 4: TypeORM Deep Dive

## 📚 Table of Contents
1. [Overview](#overview)
2. [Installation and Setup](#installation-and-setup)
3. [Entities and Decorators](#entities-and-decorators)
4. [Repository Pattern](#repository-pattern)
5. [Relationships](#relationships)
6. [Migrations](#migrations)
7. [Query Builder](#query-builder)
8. [Transactions](#transactions)
9. [Seeding](#seeding)
10. [Advanced Features](#advanced-features)
11. [Mini Project: E-Commerce Product Catalog](#mini-project-e-commerce-product-catalog)
12. [Best Practices](#best-practices)

---

## Overview

TypeORM is an Object-Relational Mapping (ORM) library for TypeScript and JavaScript. It allows you to work with databases using object-oriented programming instead of writing raw SQL queries.

### Key Features:
- **Entity-Based**: Define database schema using classes
- **TypeScript Support**: Full type safety
- **Multiple Databases**: PostgreSQL, MySQL, SQLite, MongoDB, etc.
- **Migrations**: Version control for database schema
- **Relationships**: Easy handling of database relationships
- **Query Builder**: Fluent query building API
- **Active Record & Data Mapper**: Two patterns supported

---

## Installation and Setup

### Step 1: Install Dependencies

```bash
npm install @nestjs/typeorm typeorm pg
npm install -D @types/pg
```

### Step 2: Configure TypeORM Module

This configuration sets up TypeORM to connect to your PostgreSQL database. We use `forRootAsync()` instead of `forRoot()` to enable asynchronous configuration loading, which is essential for reading from environment variables.

**Important Configuration Options Explained:**
- `type: 'postgres'`: Specifies the database type (can be 'mysql', 'sqlite', 'mongodb', etc.)
- `entities`: Pattern that tells TypeORM where to find your entity files
- `synchronize`: **CRITICAL** - Set to `true` only in development. In production, use migrations instead!
- `logging`: Enables SQL query logging for debugging

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    // ConfigModule must be loaded first to provide ConfigService
    // isGlobal: true makes ConfigService available in all modules
    ConfigModule.forRoot({ isGlobal: true }),
    
    // TypeOrmModule.forRootAsync() allows async configuration
    // This is necessary to read from environment variables
    TypeOrmModule.forRootAsync({
      // Import ConfigModule to access ConfigService
      imports: [ConfigModule],
      
      // useFactory is a function that returns the configuration object
      // ConfigService is injected automatically
      useFactory: (configService: ConfigService) => ({
        type: 'postgres', // Database type
        host: configService.get('DB_HOST', 'localhost'), // Default to localhost if not set
        port: configService.get('DB_PORT', 5432), // Default PostgreSQL port
        username: configService.get('DB_USERNAME', 'postgres'),
        password: configService.get('DB_PASSWORD'), // No default - must be set
        database: configService.get('DB_NAME', 'myapp'),
        
        // __dirname is the current directory (src/ when compiled)
        // This pattern finds all .entity.ts files recursively
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        
        // WARNING: synchronize should NEVER be true in production!
        // It auto-creates/drops tables based on entities, which can cause data loss
        // Use migrations instead for production
        synchronize: configService.get('NODE_ENV') === 'development',
        
        // Log SQL queries in development for debugging
        logging: configService.get('NODE_ENV') === 'development',
      }),
      
      // Inject ConfigService so it can be used in useFactory
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

**Security Best Practice:** Never hardcode database credentials. Always use environment variables, especially for passwords and secrets.

### Step 3: Environment Variables

```env
# .env
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=your_password
DB_NAME=myapp
NODE_ENV=development
```

---

## Entities and Decorators

### Basic Entity

Entities are TypeScript classes that map to database tables. Each property decorated with `@Column()` becomes a column in the table. This is the Object-Relational Mapping (ORM) in action - you write TypeScript classes, and TypeORM translates them to SQL tables.

**Key Concepts:**
- `@Entity()`: Marks the class as a database entity (table)
- `@PrimaryGeneratedColumn()`: Creates an auto-incrementing primary key
- `@Column()`: Maps a property to a database column
- `@CreateDateColumn()` and `@UpdateDateColumn()`: Special columns that TypeORM manages automatically

```typescript
// src/users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

// @Entity('users') - The string parameter specifies the table name
// If omitted, TypeORM uses the class name (User -> "user")
// It's a best practice to use plural, lowercase table names
@Entity('users')
export class User {
  // @PrimaryGeneratedColumn() creates an auto-incrementing integer primary key
  // This is equivalent to SERIAL PRIMARY KEY in PostgreSQL
  // TypeORM will automatically generate IDs: 1, 2, 3, etc.
  @PrimaryGeneratedColumn()
  id: number;

  // @Column() with options: defines column type, length, and constraints
  // type: 'varchar' - Variable length string
  // length: 255 - Maximum characters (common email length)
  // unique: true - Creates a unique constraint (no duplicate emails)
  @Column({ type: 'varchar', length: 255, unique: true })
  email: string;

  // Password column (should be hashed before storing!)
  // No unique constraint here - multiple users can theoretically have same password hash
  @Column({ type: 'varchar', length: 255 })
  password: string;

  // nullable: true means this column can have NULL values
  // Useful for optional fields like first/last names
  @Column({ type: 'varchar', length: 100, nullable: true })
  firstName: string;

  @Column({ type: 'varchar', length: 100, nullable: true })
  lastName: string;

  // default: true sets the default value when creating a new record
  // If not specified during creation, isActive will be true
  @Column({ type: 'boolean', default: true })
  isActive: boolean;

  // @CreateDateColumn() is a special decorator that automatically sets
  // the value to the current timestamp when a record is first created
  // You don't need to set this manually - TypeORM handles it
  @CreateDateColumn()
  createdAt: Date;

  // @UpdateDateColumn() automatically updates whenever the record is saved
  // This is perfect for tracking when data was last modified
  @UpdateDateColumn()
  updatedAt: Date;
}
```

**Important Notes:**
- Entity classes become database tables automatically when `synchronize: true` (development only)
- In production, you'll use migrations to create/modify tables
- The class name (User) is PascalCase, but the table name (users) is lowercase plural - a common convention
- These decorators provide compile-time type safety - TypeScript will catch errors before runtime

### Column Decorators

```typescript
import { Column } from 'typeorm';

export class Example {
  // Basic column
  @Column()
  name: string;

  // With type specified
  @Column({ type: 'varchar', length: 255 })
  title: string;

  // With options
  @Column({
    type: 'decimal',
    precision: 10, // Total number of digits
    scale: 2,      // Number of digits after decimal
  })
  price: number;

  // Nullable column
  @Column({ nullable: true })
  description: string;

  // Default value
  @Column({ default: 0 })
  views: number;

  // Unique constraint
  @Column({ unique: true })
  slug: string;

  // Array column (PostgreSQL)
  @Column('simple-array')
  tags: string[];

  // JSON column
  @Column('jsonb')
  metadata: Record<string, any>;

  // Enum column
  @Column({
    type: 'enum',
    enum: ['draft', 'published', 'archived'],
    default: 'draft',
  })
  status: string;
}
```

### Primary Keys

```typescript
import { Entity, PrimaryGeneratedColumn, PrimaryColumn } from 'typeorm';

// Auto-incrementing integer
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
}

// UUID
@Entity()
export class Product {
  @PrimaryGeneratedColumn('uuid')
  id: string;
}

// Custom primary key
@Entity()
export class Session {
  @PrimaryColumn()
  sessionId: string;
}

// Composite primary key
import { Entity, PrimaryColumn } from 'typeorm';

@Entity()
export class UserRole {
  @PrimaryColumn()
  userId: number;

  @PrimaryColumn()
  roleId: number;
}
```

### Indexes

```typescript
import { Entity, Column, Index } from 'typeorm';

@Entity()
@Index(['email']) // Single column index
export class User {
  @Column()
  email: string;
}

@Entity()
@Index(['firstName', 'lastName']) // Composite index
export class Person {
  @Column()
  firstName: string;

  @Column()
  lastName: string;
}

@Entity()
@Index(['email'], { unique: true }) // Unique index
export class Account {
  @Column()
  email: string;
}

// Full-text search index (PostgreSQL)
@Entity()
@Index(['title', 'content'], { fulltext: true })
export class Article {
  @Column()
  title: string;

  @Column('text')
  content: string;
}
```

---

## Repository Pattern

### Using Repository in Service

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const user = this.usersRepository.create(createUserDto);
    return await this.usersRepository.save(user);
  }

  async findAll(): Promise<User[]> {
    return await this.usersRepository.find();
  }

  async findOne(id: number): Promise<User> {
    const user = await this.usersRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return await this.usersRepository.findOne({ where: { email } });
  }

  async update(id: number, updateUserDto: Partial<CreateUserDto>): Promise<User> {
    await this.usersRepository.update(id, updateUserDto);
    return this.findOne(id);
  }

  async remove(id: number): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
```

### Register Repository in Module

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

### Repository Methods

```typescript
// Find methods
await repository.find(); // Find all
await repository.findOne({ where: { id: 1 } });
await repository.findBy({ email: 'user@example.com' });
await repository.findOneBy({ id: 1 });

// Find with relations
await repository.find({ relations: ['posts'] });
await repository.findOne({ 
  where: { id: 1 },
  relations: ['posts', 'profile']
});

// Find with conditions
await repository.find({
  where: { isActive: true },
  order: { createdAt: 'DESC' },
  take: 10,
  skip: 0,
});

// Save (insert or update)
await repository.save(user);
await repository.save([user1, user2]);

// Update
await repository.update(1, { isActive: false });
await repository.update({ email: 'old@example.com' }, { email: 'new@example.com' });

// Delete
await repository.delete(1);
await repository.remove(user); // Removes entity instance
await repository.softDelete(1); // Soft delete (requires DeleteDateColumn)

// Count
await repository.count();
await repository.count({ where: { isActive: true } });

// Exist
await repository.exist({ where: { email: 'user@example.com' } });
```

---

## Relationships

### One-to-One

```typescript
// User entity
import { Entity, OneToOne, JoinColumn } from 'typeorm';
import { Profile } from './profile.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @OneToOne(() => Profile, profile => profile.user, { cascade: true })
  @JoinColumn() // Foreign key in User table
  profile: Profile;
}

// Profile entity
import { Entity, Column, OneToOne } from 'typeorm';
import { User } from './user.entity';

@Entity()
export class Profile {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  bio: string;

  @OneToOne(() => User, user => user.profile)
  user: User;
}
```

### One-to-Many / Many-to-One

```typescript
// User entity (One-to-Many)
import { Entity, OneToMany } from 'typeorm';
import { Post } from './post.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @OneToMany(() => Post, post => post.author)
  posts: Post[];
}

// Post entity (Many-to-One)
import { Entity, Column, ManyToOne, JoinColumn } from 'typeorm';
import { User } from './user.entity';

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @ManyToOne(() => User, user => user.posts, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'author_id' })
  author: User;

  @Column()
  author_id: number;
}
```

### Many-to-Many

```typescript
// Post entity
import { Entity, ManyToMany, JoinTable } from 'typeorm';
import { Tag } from './tag.entity';

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @ManyToMany(() => Tag, tag => tag.posts)
  @JoinTable({ // Owner side of relationship
    name: 'post_tags', // Junction table name
    joinColumn: { name: 'post_id', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'tag_id', referencedColumnName: 'id' },
  })
  tags: Tag[];
}

// Tag entity
import { Entity, Column, ManyToMany } from 'typeorm';
import { Post } from './post.entity';

@Entity()
export class Tag {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany(() => Post, post => post.tags)
  posts: Post[];
}
```

### Working with Relationships

```typescript
// Create with relations
const user = await usersRepository.create({
  email: 'user@example.com',
  profile: {
    bio: 'My bio',
  },
});
await usersRepository.save(user);

// Load relations
const user = await usersRepository.findOne({
  where: { id: 1 },
  relations: ['profile', 'posts'],
});

// Using query builder for relations
const user = await usersRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .where('user.id = :id', { id: 1 })
  .getOne();

// Add relations
const user = await usersRepository.findOne({ where: { id: 1 } });
const tag = await tagsRepository.findOne({ where: { id: 1 } });
const post = await postsRepository.findOne({ 
  where: { id: 1 },
  relations: ['tags']
});

post.tags.push(tag);
await postsRepository.save(post);
```

---

## Migrations

### Configuration

```typescript
// ormconfig.ts or data-source.ts
import { DataSource } from 'typeorm';
import { ConfigService } from '@nestjs/config';

export default new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT) || 5432,
  username: process.env.DB_USERNAME || 'postgres',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'myapp',
  entities: ['src/**/*.entity{.ts,.js}'],
  migrations: ['src/database/migrations/*{.ts,.js}'],
});
```

### Creating Migrations

```bash
# Generate migration (auto-generate from entities)
npm run typeorm migration:generate src/database/migrations/CreateUserTable

# Create empty migration
npm run typeorm migration:create src/database/migrations/AddUserEmailIndex

# Run migrations
npm run typeorm migration:run

# Revert migration
npm run typeorm migration:revert
```

### Migration Example

```typescript
// src/database/migrations/1234567890-CreateUserTable.ts
import { MigrationInterface, QueryRunner, Table } from 'typeorm';

export class CreateUserTable1234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          {
            name: 'id',
            type: 'int',
            isPrimary: true,
            isGenerated: true,
            generationStrategy: 'increment',
          },
          {
            name: 'email',
            type: 'varchar',
            length: '255',
            isUnique: true,
          },
          {
            name: 'password',
            type: 'varchar',
            length: '255',
          },
          {
            name: 'created_at',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP',
          },
          {
            name: 'updated_at',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP',
            onUpdate: 'CURRENT_TIMESTAMP',
          },
        ],
      }),
      true,
    );

    await queryRunner.createIndex(
      'users',
      new TableIndex({
        name: 'IDX_USERS_EMAIL',
        columnNames: ['email'],
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
  }
}
```

---

## Query Builder

### Basic Queries

```typescript
// Select
const users = await usersRepository
  .createQueryBuilder('user')
  .where('user.isActive = :isActive', { isActive: true })
  .orderBy('user.createdAt', 'DESC')
  .getMany();

// Select with relations
const users = await usersRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .getMany();

// Select specific fields
const users = await usersRepository
  .createQueryBuilder('user')
  .select(['user.id', 'user.email', 'user.firstName'])
  .getMany();
```

### Joins

```typescript
// Inner join
const posts = await postsRepository
  .createQueryBuilder('post')
  .innerJoin('post.author', 'author')
  .where('author.id = :authorId', { authorId: 1 })
  .getMany();

// Left join and select
const posts = await postsRepository
  .createQueryBuilder('post')
  .leftJoinAndSelect('post.author', 'author')
  .leftJoinAndSelect('post.tags', 'tag')
  .getMany();

// Join with conditions
const posts = await postsRepository
  .createQueryBuilder('post')
  .leftJoin('post.comments', 'comment', 'comment.isApproved = :approved', { approved: true })
  .getMany();
```

### Aggregations

```typescript
// Count
const count = await usersRepository
  .createQueryBuilder('user')
  .where('user.isActive = :isActive', { isActive: true })
  .getCount();

// Sum, Avg, Max, Min
const stats = await postsRepository
  .createQueryBuilder('post')
  .select('post.author_id', 'authorId')
  .addSelect('COUNT(post.id)', 'postCount')
  .addSelect('SUM(post.views)', 'totalViews')
  .addSelect('AVG(post.views)', 'avgViews')
  .groupBy('post.author_id')
  .getRawMany();
```

### Subqueries

```typescript
const activeUsers = await usersRepository
  .createQueryBuilder('user')
  .where((qb) => {
    const subQuery = qb
      .subQuery()
      .select('post.author_id')
      .from('posts', 'post')
      .where('post.status = :status', { status: 'published' })
      .getQuery();
    return 'user.id IN ' + subQuery;
  })
  .getMany();
```

---

## Transactions

### Using Transaction in Service

```typescript
import { DataSource } from 'typeorm';

@Injectable()
export class OrdersService {
  constructor(
    private dataSource: DataSource,
    @InjectRepository(Order)
    private ordersRepository: Repository<Order>,
    @InjectRepository(OrderItem)
    private orderItemsRepository: Repository<OrderItem>,
  ) {}

  async createOrder(orderData: CreateOrderDto): Promise<Order> {
    const queryRunner = this.dataSource.createQueryRunner();

    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // Create order
      const order = queryRunner.manager.create(Order, orderData);
      await queryRunner.manager.save(order);

      // Create order items
      for (const item of orderData.items) {
        const orderItem = queryRunner.manager.create(OrderItem, {
          ...item,
          order_id: order.id,
        });
        await queryRunner.manager.save(orderItem);
      }

      await queryRunner.commitTransaction();
      return order;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```

---

## Seeding

### Seed Example

```typescript
// src/database/seeds/users.seed.ts
import { DataSource } from 'typeorm';
import { User } from '../../users/entities/user.entity';

export async function seedUsers(dataSource: DataSource): Promise<void> {
  const usersRepository = dataSource.getRepository(User);

  const users = [
    {
      email: 'admin@example.com',
      password: 'hashed_password',
      firstName: 'Admin',
      lastName: 'User',
    },
    {
      email: 'user@example.com',
      password: 'hashed_password',
      firstName: 'Regular',
      lastName: 'User',
    },
  ];

  for (const userData of users) {
    const existingUser = await usersRepository.findOne({
      where: { email: userData.email },
    });

    if (!existingUser) {
      const user = usersRepository.create(userData);
      await usersRepository.save(user);
      console.log(`Created user: ${userData.email}`);
    }
  }
}
```

---

## Advanced Features

### Soft Delete

```typescript
import { Entity, DeleteDateColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @DeleteDateColumn()
  deletedAt: Date;
}

// Usage
await repository.softDelete(1);
await repository.restore(1);
await repository.find({ withDeleted: true }); // Include soft-deleted
```

### Listeners and Subscribers

```typescript
import { EntitySubscriberInterface, EventSubscriber, InsertEvent } from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log('Before user insert:', event.entity);
  }

  afterInsert(event: InsertEvent<User>) {
    console.log('After user insert:', event.entity);
  }
}
```

---

## Mini Project: E-Commerce Product Catalog

Build a complete product catalog system with categories, tags, and inventory management.

### Entities:

```typescript
// Category entity
@Entity('categories')
export class Category {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  name: string;

  @Column({ unique: true })
  slug: string;

  @Column({ nullable: true })
  description: string;

  @ManyToOne(() => Category, category => category.children, { nullable: true })
  parent: Category;

  @OneToMany(() => Category, category => category.parent)
  children: Category[];

  @OneToMany(() => Product, product => product.category)
  products: Product[];
}

// Product entity
@Entity('products')
export class Product {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column({ unique: true })
  sku: string;

  @Column('decimal', { precision: 10, scale: 2 })
  price: number;

  @Column('text', { nullable: true })
  description: string;

  @Column({ default: 0 })
  stock: number;

  @ManyToOne(() => Category, category => category.products)
  category: Category;

  @ManyToMany(() => Tag, tag => tag.products)
  @JoinTable()
  tags: Tag[];

  @OneToMany(() => ProductImage, image => image.product)
  images: ProductImage[];
}
```

---

## Best Practices

1. **Use Entities for Schema**: Define all tables as entities
2. **Use Migrations**: Never use `synchronize: true` in production
3. **Index Frequently Queried Columns**: Add indexes for performance
4. **Use Transactions**: For operations affecting multiple tables
5. **Eager vs Lazy Loading**: Use `relations` option when needed
6. **Use Query Builder**: For complex queries
7. **Validate Data**: Use class-validator with DTOs

---

*You now understand TypeORM deeply! 🗄️*

