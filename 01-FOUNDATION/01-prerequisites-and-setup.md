# Module 1: Prerequisites & Setup

## 📚 Table of Contents
1. [Overview](#overview)
2. [Required Knowledge](#required-knowledge)
3. [System Requirements](#system-requirements)
4. [Installation Guide](#installation-guide)
5. [Development Environment Setup](#development-environment-setup)
6. [TypeScript Fundamentals Review](#typescript-fundamentals-review)
7. [First NestJS Application](#first-nestjs-application)
8. [IDE Configuration](#ide-configuration)
9. [Best Practices](#best-practices)
10. [Mini Project](#mini-project)

---

## Overview

Before diving into NestJS, we need to ensure you have all the necessary tools and foundational knowledge. This module will set up your development environment and refresh essential TypeScript concepts.

---

## Required Knowledge

### Prerequisites You Should Have:
- **JavaScript/ES6+**: Arrow functions, async/await, destructuring, spread operator
- **TypeScript Basics**: Types, interfaces, classes, generics
- **Node.js**: Understanding of Node.js runtime, npm/yarn
- **HTTP/REST**: Understanding of REST APIs, HTTP methods, status codes
- **Database Basics**: SQL knowledge, database relationships

### If You Need a Refresher:
- TypeScript Handbook: https://www.typescriptlang.org/docs/
- Node.js Documentation: https://nodejs.org/docs/
- MDN JavaScript Guide: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide

---

## System Requirements

### Minimum Requirements:
- **OS**: macOS 10.15+, Windows 10+, or Linux
- **Node.js**: v18.0.0 or higher (LTS version recommended)
- **npm**: v9.0.0 or higher (comes with Node.js)
- **Git**: Latest version
- **RAM**: 8GB minimum (16GB recommended)
- **Storage**: At least 5GB free space

### Recommended Tools:
- **VS Code**: Latest version with extensions
- **Postman/Insomnia**: API testing
- **Docker Desktop**: For containerization (we'll set up later)

---

## Installation Guide

### Step 1: Install Node.js and npm

#### macOS (using Homebrew):
```bash
# Install Homebrew if not installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Node.js (includes npm)
brew install node

# Verify installation
node --version  # Should show v18.x.x or higher
npm --version   # Should show v9.x.x or higher
```

#### Windows:
1. Download Node.js installer from https://nodejs.org/
2. Choose the LTS version
3. Run the installer and follow the prompts
4. Verify in Command Prompt:
```bash
node --version
npm --version
```

#### Linux (Ubuntu/Debian):
```bash
# Update package index
sudo apt update

# Install Node.js and npm
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installation
node --version
npm --version
```

### Step 2: Install Git

#### macOS:
```bash
brew install git
```

#### Windows:
Download from https://git-scm.com/download/win

#### Linux:
```bash
sudo apt install git
```

### Step 3: Install Yarn (Optional but Recommended)

```bash
npm install -g yarn
```

### Step 4: Install PostgreSQL

#### macOS:
```bash
brew install postgresql@14
brew services start postgresql@14

# Create a database user
createuser -s postgres
```

#### Windows:
1. Download from https://www.postgresql.org/download/windows/
2. Run the installer
3. Remember the password you set for the `postgres` user

#### Linux:
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create a database user
sudo -u postgres createuser -s $USER
```

### Step 5: Verify PostgreSQL Installation

```bash
# Connect to PostgreSQL
psql -U postgres

# In psql, run:
SELECT version();

# Exit psql
\q
```

---

## Development Environment Setup

### Step 1: Install NestJS CLI Globally

```bash
npm install -g @nestjs/cli

# Or with yarn
yarn global add @nestjs/cli

# Verify installation
nest --version
```

### Step 2: Create a New NestJS Project

```bash
# Create a new project
nest new my-first-app

# Navigate to the project
cd my-first-app

# Start the development server
npm run start:dev
```

The application will be available at `http://localhost:3000`

### Step 3: Project Structure Overview

```
my-first-app/
├── src/
│   ├── main.ts           # Application entry point
│   ├── app.module.ts     # Root module
│   ├── app.controller.ts # Root controller
│   └── app.service.ts    # Root service
├── test/                 # Test files
├── node_modules/         # Dependencies
├── .eslintrc.js         # ESLint configuration
├── .prettierrc          # Prettier configuration
├── nest-cli.json        # NestJS CLI configuration
├── package.json         # Project dependencies
├── tsconfig.json        # TypeScript configuration
└── README.md            # Project documentation
```

---

## TypeScript Fundamentals Review

### 1. Types and Interfaces

```typescript
// Basic Types
let name: string = "John";
let age: number = 30;
let isActive: boolean = true;
let items: string[] = ["apple", "banana"];

// Interface
interface User {
  id: number;
  name: string;
  email: string;
  isActive?: boolean; // Optional property
}

// Using interface
const user: User = {
  id: 1,
  name: "John Doe",
  email: "john@example.com"
};
```

### 2. Classes

```typescript
class Person {
  private age: number; // Private property
  
  constructor(
    public name: string,
    age: number
  ) {
    this.age = age;
  }
  
  getAge(): number {
    return this.age;
  }
  
  greet(): string {
    return `Hello, I'm ${this.name}`;
  }
}

const person = new Person("John", 30);
console.log(person.greet()); // "Hello, I'm John"
```

### 3. Generics

```typescript
// Generic function
function getFirst<T>(items: T[]): T {
  return items[0];
}

const numbers = [1, 2, 3];
const firstNumber = getFirst<number>(numbers);

const strings = ["a", "b", "c"];
const firstString = getFirst<string>(strings);
```

### 4. Async/Await

```typescript
// Promise-based function
function fetchUser(id: number): Promise<User> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({ id, name: "John", email: "john@example.com" });
    }, 1000);
  });
}

// Using async/await
async function getUser(id: number): Promise<User> {
  const user = await fetchUser(id);
  return user;
}
```

### 5. Decorators

```typescript
// Class decorator
function LogClass(target: Function) {
  console.log(`Class ${target.name} is being decorated`);
}

@LogClass
class Example {
  // ...
}

// Method decorator
function LogMethod(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  
  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${propertyKey} with arguments:`, args);
    return originalMethod.apply(this, args);
  };
}

class Example {
  @LogMethod
  greet(name: string) {
    return `Hello, ${name}`;
  }
}
```

---

## First NestJS Application

Let's create a simple "Hello World" application to understand the basics.

### Step 1: Generate a New Controller

```bash
nest generate controller hello
# Or short form: nest g co hello
```

### Step 2: Update the Controller

```typescript
// src/hello/hello.controller.ts
import { Controller, Get, Param } from '@nestjs/common';

@Controller('hello')
export class HelloController {
  @Get()
  getHello(): string {
    return 'Hello World!';
  }
  
  @Get(':name')
  getHelloWithName(@Param('name') name: string): string {
    return `Hello, ${name}!`;
  }
}
```

### Step 3: Test the Endpoint

```bash
# Start the server
npm run start:dev

# Test in browser or using curl
curl http://localhost:3000/hello
# Response: "Hello World!"

curl http://localhost:3000/hello/NestJS
# Response: "Hello, NestJS!"
```

---

## IDE Configuration

### VS Code Extensions (Recommended)

1. **ES7+ React/Redux/React-Native snippets**
2. **ESLint** - Code linting
3. **Prettier** - Code formatter
4. **TypeScript Hero** - Organize imports
5. **REST Client** - Test APIs directly in VS Code
6. **Thunder Client** - Alternative to Postman
7. **Docker** - Docker support
8. **Kubernetes** - Kubernetes support

### VS Code Settings (settings.json)

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "files.exclude": {
    "**/.git": true,
    "**/.DS_Store": true,
    "**/node_modules": true
  }
}
```

### Prettier Configuration (.prettierrc)

```json
{
  "singleQuote": true,
  "trailingComma": "es5",
  "tabWidth": 2,
  "semi": true,
  "printWidth": 80,
  "arrowParens": "always"
}
```

---

## Best Practices

### 1. Project Organization

```
src/
├── common/          # Shared utilities
│   ├── decorators/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/          # Configuration files
├── database/        # Database related
│   └── migrations/
├── modules/         # Feature modules
│   ├── users/
│   ├── products/
│   └── orders/
└── main.ts
```

### 2. Naming Conventions

- **Files**: kebab-case (user.controller.ts)
- **Classes**: PascalCase (UserController)
- **Variables/Functions**: camelCase (getUserById)
- **Constants**: UPPER_SNAKE_CASE (MAX_RETRY_COUNT)
- **Interfaces**: PascalCase (IUser) or without prefix (User)

### 3. Import Organization

```typescript
// 1. External libraries
import { Injectable } from '@nestjs/common';
import { Repository } from 'typeorm';

// 2. Internal modules
import { UserService } from '../users/user.service';

// 3. Local imports
import { CreateUserDto } from './dto/create-user.dto';
```

### 4. Environment Variables

Create a `.env` file (never commit this):

```env
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=your_password
DATABASE_NAME=myapp

# Application
PORT=3000
NODE_ENV=development

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRATION=1h
```

Install dotenv:
```bash
npm install @nestjs/config
```

---

## Mini Project: TypeScript CLI Application

Build a simple command-line task manager to practice TypeScript concepts.

### Project Requirements:
1. Create tasks (add)
2. List all tasks (list)
3. Mark task as complete (complete)
4. Delete tasks (delete)
5. Save tasks to a JSON file

### Implementation:

```typescript
// task-manager.ts
import * as fs from 'fs';
import * as path from 'path';

interface Task {
  id: number;
  description: string;
  completed: boolean;
  createdAt: Date;
}

class TaskManager {
  private tasks: Task[] = [];
  private filePath: string = path.join(__dirname, 'tasks.json');

  constructor() {
    this.loadTasks();
  }

  private loadTasks(): void {
    if (fs.existsSync(this.filePath)) {
      const data = fs.readFileSync(this.filePath, 'utf-8');
      this.tasks = JSON.parse(data).map((task: any) => ({
        ...task,
        createdAt: new Date(task.createdAt),
      }));
    }
  }

  private saveTasks(): void {
    fs.writeFileSync(this.filePath, JSON.stringify(this.tasks, null, 2));
  }

  add(description: string): void {
    const task: Task = {
      id: Date.now(),
      description,
      completed: false,
      createdAt: new Date(),
    };
    this.tasks.push(task);
    this.saveTasks();
    console.log(`Task added: ${description}`);
  }

  list(): void {
    if (this.tasks.length === 0) {
      console.log('No tasks found.');
      return;
    }
    this.tasks.forEach((task) => {
      const status = task.completed ? '✓' : '○';
      console.log(`${status} [${task.id}] ${task.description}`);
    });
  }

  complete(id: number): void {
    const task = this.tasks.find((t) => t.id === id);
    if (task) {
      task.completed = true;
      this.saveTasks();
      console.log(`Task ${id} marked as complete.`);
    } else {
      console.log(`Task ${id} not found.`);
    }
  }

  delete(id: number): void {
    const index = this.tasks.findIndex((t) => t.id === id);
    if (index !== -1) {
      this.tasks.splice(index, 1);
      this.saveTasks();
      console.log(`Task ${id} deleted.`);
    } else {
      console.log(`Task ${id} not found.`);
    }
  }
}

// CLI Interface
const args = process.argv.slice(2);
const command = args[0];
const taskManager = new TaskManager();

switch (command) {
  case 'add':
    taskManager.add(args.slice(1).join(' '));
    break;
  case 'list':
    taskManager.list();
    break;
  case 'complete':
    taskManager.complete(Number(args[1]));
    break;
  case 'delete':
    taskManager.delete(Number(args[1]));
    break;
  default:
    console.log('Usage:');
    console.log('  add <description>    - Add a new task');
    console.log('  list                - List all tasks');
    console.log('  complete <id>       - Mark task as complete');
    console.log('  delete <id>         - Delete a task');
}
```

### Usage:

```bash
# Compile TypeScript
tsc task-manager.ts

# Run the application
node task-manager.js add "Learn TypeScript"
node task-manager.js list
node task-manager.js complete 1234567890
node task-manager.js delete 1234567890
```

---

## Next Steps

✅ Verify all installations are working  
✅ Complete the mini project  
✅ Review TypeScript concepts if needed  
✅ Move to Module 2: NestJS Fundamentals  

---

## Quick Reference Commands

```bash
# Node.js version
node --version

# npm version
npm --version

# Install NestJS CLI
npm install -g @nestjs/cli

# Create new project
nest new project-name

# Generate controller
nest g controller controller-name

# Generate service
nest g service service-name

# Generate module
nest g module module-name

# Run development server
npm run start:dev

# Build for production
npm run build

# Run production build
npm run start:prod
```

---

*You're now ready to start building with NestJS! 🚀*

