# Module 14: Docker Compose & Local Development

## 📚 Table of Contents
1. [Overview](#overview)
2. [What is Docker Compose?](#what-is-docker-compose)
3. [Docker Compose Installation](#docker-compose-installation)
4. [Creating docker-compose.yml](#creating-docker-composeyml)
5. [Service Configuration](#service-configuration)
6. [Networking in Compose](#networking-in-compose)
7. [Volumes in Compose](#volumes-in-compose)
8. [Environment Variables](#environment-variables)
9. [Development vs Production](#development-v-production)
10. [Common Patterns](#common-patterns)
11. [Health Checks](#health-checks)
12. [Best Practices](#best-practices)
13. [Mini Project: Complete Local Stack](#mini-project-complete-local-stack)

---

## Overview

Docker Compose allows you to define and run multi-container Docker applications. Instead of running multiple `docker run` commands, you define everything in a YAML file and start everything with one command.

**Benefits:**
- **Single Command**: Start entire stack with `docker-compose up`
- **Configuration as Code**: Everything defined in YAML
- **Service Orchestration**: Manage multiple containers together
- **Development Environment**: Consistent dev environment for team
- **Easy Cleanup**: Stop everything with one command

---

## What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications. It uses a YAML file to configure application services.

**Key Concepts:**
- **Service**: A container definition
- **Network**: Isolated network for services
- **Volume**: Persistent data storage
- **Compose File**: YAML configuration file

---

## Docker Compose Installation

Docker Compose comes bundled with Docker Desktop. For Linux:

```bash
# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker-compose --version
```

---

## Creating docker-compose.yml

Create a `docker-compose.yml` file in your project root:

### Basic Compose File

```yaml
# docker-compose.yml

# Version of Compose file format (optional in newer versions)
version: '3.8'

# Define services (containers)
services:
  # Service name: app
  # This is your NestJS application
  app:
    # Build context: where Dockerfile is located
    # . means current directory
    build:
      context: .
      dockerfile: Dockerfile
    
    # Container name (optional, defaults to project-name_service-name)
    container_name: nestjs-app
    
    # Port mapping: host:container
    # Access app at http://localhost:3000
    ports:
      - "3000:3000"
    
    # Environment variables
    environment:
      - NODE_ENV=development
      - PORT=3000
      - DATABASE_HOST=postgres
      - DATABASE_PORT=5432
      - DATABASE_USER=postgres
      - DATABASE_PASSWORD=postgres
      - DATABASE_NAME=myapp
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    
    # Dependencies: start these services first
    depends_on:
      - postgres
      - redis
    
    # Restart policy: restart container if it stops
    restart: unless-stopped
    
    # Networks: connect to custom network
    networks:
      - app-network

  # PostgreSQL database service
  postgres:
    # Use official PostgreSQL image
    image: postgres:14-alpine
    
    container_name: nestjs-postgres
    
    # Environment variables for PostgreSQL
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    
    # Port mapping (expose for local access)
    ports:
      - "5432:5432"
    
    # Named volume for data persistence
    volumes:
      - postgres-data:/var/lib/postgresql/data
    
    # Health check: ensures database is ready
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    
    networks:
      - app-network
    
    restart: unless-stopped

  # Redis service for caching
  redis:
    image: redis:7-alpine
    
    container_name: nestjs-redis
    
    # Port mapping
    ports:
      - "6379:6379"
    
    # Volume for Redis data (optional)
    volumes:
      - redis-data:/data
    
    # Health check
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    
    networks:
      - app-network
    
    restart: unless-stopped

# Define volumes (named volumes for data persistence)
volumes:
  # PostgreSQL data volume
  # Data persists even if container is removed
  postgres-data:
    driver: local
  
  # Redis data volume
  redis-data:
    driver: local

# Define networks
networks:
  # Custom network for services to communicate
  app-network:
    driver: bridge
```

**Compose File Structure Explained:**
- `version`: Compose file format version
- `services`: Container definitions
- `volumes`: Named volumes for data
- `networks`: Custom networks

---

## Service Configuration

### Build Configuration

```yaml
services:
  app:
    build:
      # Directory containing Dockerfile
      context: .
      
      # Dockerfile name (default: Dockerfile)
      dockerfile: Dockerfile
      
      # Build arguments
      args:
        NODE_ENV: production
      
      # Target stage (for multi-stage builds)
      target: production
```

### Image Configuration

```yaml
services:
  app:
    # Use existing image instead of building
    image: my-nestjs-app:latest
    
    # Or use image with build (builds if not exists)
    image: my-nestjs-app:latest
    build: .
```

### Port Configuration

```yaml
services:
  app:
    ports:
      # Format: "host-port:container-port"
      - "3000:3000"
      
      # Map to different host port
      - "8080:3000"
      
      # Bind to specific host IP
      - "127.0.0.1:3000:3000"
      
      # Expose without mapping (only accessible on network)
      - "3000"
```

### Environment Variables

```yaml
services:
  app:
    # Inline environment variables
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/myapp
    
    # Or use .env file
    env_file:
      - .env
      - .env.local
    
    # Or use environment file syntax
    environment:
      NODE_ENV: ${NODE_ENV:-development}
      DATABASE_URL: ${DATABASE_URL}
```

### Dependencies

```yaml
services:
  app:
    # Wait for dependencies to be healthy (if health checks defined)
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    # Or simple dependency (just waits for start)
    depends_on:
      - postgres
      - redis
```

---

## Networking in Compose

Services in the same Compose file automatically join the same network and can communicate using service names.

### Service Discovery

```yaml
services:
  app:
    environment:
      # Use service name as hostname
      DATABASE_HOST: postgres  # Not localhost!
      REDIS_HOST: redis       # Not localhost!
    
  postgres:
    # Accessible as "postgres" from other services
    
  redis:
    # Accessible as "redis" from other services
```

**How Service Discovery Works:**
- Docker Compose creates a network
- Services can reach each other by service name
- Service name resolves to container IP
- No need for IP addresses or ports (for internal communication)

### Custom Networks

```yaml
services:
  app:
    networks:
      - frontend
      - backend
  
  postgres:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

---

## Volumes in Compose

### Named Volumes

```yaml
services:
  postgres:
    volumes:
      # Named volume (managed by Docker)
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
    driver: local
    # Optional: specify driver options
    driver_opts:
      type: none
      o: bind
      device: /path/to/data
```

### Bind Mounts

```yaml
services:
  app:
    volumes:
      # Bind mount: host-path:container-path
      # Useful for development (live code reloading)
      - ./src:/app/src
      - ./uploads:/app/uploads
```

### Anonymous Volumes

```yaml
services:
  postgres:
    volumes:
      # Anonymous volume (temporary)
      - /var/lib/postgresql/data
```

### Volume Configuration

```yaml
volumes:
  postgres-data:
    driver: local
    # External volume (created separately)
    external: true
    name: my-postgres-data
```

---

## Environment Variables

### Using .env File

Create a `.env` file in the same directory as `docker-compose.yml`:

```env
# .env
NODE_ENV=development
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=myapp
REDIS_HOST=redis
REDIS_PORT=6379
```

Reference in Compose file:

```yaml
services:
  app:
    environment:
      NODE_ENV: ${NODE_ENV:-development}
      DATABASE_HOST: ${DATABASE_HOST:-postgres}
      DATABASE_PASSWORD: ${DATABASE_PASSWORD}
    
    # Or load entire .env file
    env_file:
      - .env
```

**Variable Substitution:**
- `${VARIABLE}`: Use environment variable
- `${VARIABLE:-default}`: Use default if not set
- `${VARIABLE?error}`: Error if not set

---

## Development vs Production

### Separate Compose Files

Create different files for different environments:

**docker-compose.yml** (Development):
```yaml
services:
  app:
    build:
      context: .
      target: development
    volumes:
      # Mount source code for hot reload
      - ./src:/app/src
      - ./node_modules:/app/node_modules
    environment:
      NODE_ENV: development
    command: npm run start:dev
```

**docker-compose.prod.yml** (Production):
```yaml
services:
  app:
    build:
      context: .
      target: production
    environment:
      NODE_ENV: production
    restart: always
```

### Multi-file Compose

```bash
# Development
docker-compose -f docker-compose.yml up

# Production (overrides dev config)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

## Common Patterns

### Development with Hot Reload

```yaml
services:
  app:
    build: .
    volumes:
      # Mount source code
      - ./src:/app/src
      # Preserve node_modules (don't override with host)
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: npm run start:dev
```

### Database Migrations

```yaml
services:
  migrate:
    build: .
    command: npm run migration:run
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/myapp
```

### Database Seeding

```yaml
services:
  seed:
    build: .
    command: npm run seed:run
    depends_on:
      - postgres
      - migrate
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/myapp
```

### Multiple Environments

```yaml
# docker-compose.dev.yml
services:
  app:
    build:
      target: development
    volumes:
      - ./src:/app/src

# docker-compose.test.yml
services:
  app:
    build:
      target: test
    environment:
      - NODE_ENV=test

# docker-compose.prod.yml
services:
  app:
    build:
      target: production
    restart: always
```

---

## Health Checks

Health checks ensure services are ready before depending services start.

```yaml
services:
  postgres:
    healthcheck:
      # Command to check health
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      
      # How often to check
      interval: 10s
      
      # Timeout for check
      timeout: 5s
      
      # Number of failures before unhealthy
      retries: 5
      
      # Time to wait before first check
      start_period: 10s
  
  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
  
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    # Wait for dependencies to be healthy
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
```

---

## Best Practices

### 1. Use Named Volumes for Data

```yaml
volumes:
  postgres-data:
    driver: local
```

### 2. Set Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

### 3. Use Health Checks

Always define health checks for services that other services depend on.

### 4. Separate Development and Production

Use different Compose files or override files for different environments.

### 5. Use .env Files

Never hardcode sensitive values in Compose files.

### 6. Version Control

Commit `docker-compose.yml` but not `.env` files with secrets.

---

## Docker Compose Commands

### Basic Commands

```bash
# Start all services
docker-compose up

# Start in detached mode (background)
docker-compose up -d

# Start specific services
docker-compose up app postgres

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# View logs
docker-compose logs

# Follow logs
docker-compose logs -f

# View logs for specific service
docker-compose logs app

# Restart a service
docker-compose restart app

# Scale a service
docker-compose up -d --scale app=3

# Build images
docker-compose build

# Rebuild without cache
docker-compose build --no-cache

# Execute command in service
docker-compose exec app sh

# View running services
docker-compose ps
```

### Advanced Commands

```bash
# Validate Compose file
docker-compose config

# Pull images
docker-compose pull

# Remove stopped containers
docker-compose rm

# Stop services
docker-compose stop

# Start stopped services
docker-compose start

# Pause services
docker-compose pause

# Unpause services
docker-compose unpause
```

---

## Mini Project: Complete Local Stack

Create a complete development stack with:

1. NestJS application
2. PostgreSQL database
3. Redis cache
4. pgAdmin (database management)
5. Health checks
6. Volume persistence
7. Network isolation

### Complete docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: nestjs-app
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_HOST=postgres
      - DATABASE_PORT=5432
      - DATABASE_USER=postgres
      - DATABASE_PASSWORD=postgres
      - DATABASE_NAME=myapp
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  postgres:
    image: postgres:14-alpine
    container_name: nestjs-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: nestjs-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: nestjs-pgadmin
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    volumes:
      - pgadmin-data:/var/lib/pgadmin
    networks:
      - app-network
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:
  pgadmin-data:

networks:
  app-network:
    driver: bridge
```

---

## Next Steps

✅ Understand Docker Compose  
✅ Create multi-service configurations  
✅ Set up development environment  
✅ Use volumes and networks  
✅ Implement health checks  
✅ Move to Module 15: Kubernetes Basics  

---

*You now know how to orchestrate multi-container applications with Docker Compose! 🐳*

