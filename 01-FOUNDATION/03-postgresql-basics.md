# Module 3: PostgreSQL Basics

## 📚 Table of Contents
1. [Overview](#overview)
2. [Database Design Principles](#database-design-principles)
3. [PostgreSQL Installation & Setup](#postgresql-installation--setup)
4. [SQL Fundamentals](#sql-fundamentals)
5. [Database Relationships](#database-relationships)
6. [Indexes and Performance](#indexes-and-performance)
7. [Transactions and ACID](#transactions-and-acid)
8. [Advanced Queries](#advanced-queries)
9. [Best Practices](#best-practices)
10. [Mini Project: Blog Database](#mini-project-blog-database)

---

## Overview

PostgreSQL is a powerful, open-source relational database system. It's known for its reliability, feature richness, and performance. Understanding PostgreSQL is crucial for building robust backend applications.

### Key Features:
- **ACID Compliance** - Guarantees data integrity
- **Extensible** - Supports custom data types and functions
- **JSON Support** - Native JSON and JSONB data types
- **Full-Text Search** - Built-in text search capabilities
- **Concurrent Access** - Handles multiple users efficiently
- **Extensible** - Custom functions, operators, and data types

---

## Database Design Principles

### 1. Normalization

**First Normal Form (1NF)**: Each column contains atomic values, no repeating groups.

**Example - Before 1NF:**
```
| UserID | Name    | Skills                    |
|--------|---------|---------------------------|
| 1      | John    | Python, Java, JavaScript  |
```

**After 1NF:**
```
| UserID | Name    | Skill      |
|--------|---------|------------|
| 1      | John    | Python     |
| 1      | John    | Java       |
| 1      | John    | JavaScript |
```

**Second Normal Form (2NF)**: Must be in 1NF and all non-key attributes must depend on the entire primary key.

**Third Normal Form (3NF)**: Must be in 2NF and no transitive dependencies.

### 2. Primary Keys

Every table should have a primary key that uniquely identifies each row.

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,  -- Auto-incrementing integer
  email VARCHAR(255) UNIQUE NOT NULL
);
```

### 3. Foreign Keys

Foreign keys maintain referential integrity between tables.

```sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL
);
```

### 4. Data Types Selection

Choose appropriate data types for efficiency and correctness:

- **INTEGER** / **BIGINT**: For whole numbers
- **DECIMAL** / **NUMERIC**: For precise decimal numbers
- **VARCHAR(n)**: Variable-length strings
- **TEXT**: Unlimited length strings
- **BOOLEAN**: True/false values
- **TIMESTAMP**: Date and time
- **JSON** / **JSONB**: JSON data (JSONB is binary, faster)
- **UUID**: Universally unique identifiers

---

## PostgreSQL Installation & Setup

### macOS (Homebrew)

```bash
# Install PostgreSQL
brew install postgresql@14

# Start PostgreSQL service
brew services start postgresql@14

# Create database
createdb myapp

# Connect to database
psql myapp
```

### Linux (Ubuntu/Debian)

```bash
# Install PostgreSQL
sudo apt update
sudo apt install postgresql postgresql-contrib

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Switch to postgres user and create database
sudo -u postgres psql
CREATE DATABASE myapp;
CREATE USER myuser WITH PASSWORD 'mypassword';
GRANT ALL PRIVILEGES ON DATABASE myapp TO myuser;
\q
```

### Windows

1. Download installer from https://www.postgresql.org/download/windows/
2. Run installer and follow prompts
3. Remember the password for `postgres` user
4. Use pgAdmin or psql command line

### Verify Installation

```bash
# Check PostgreSQL version
psql --version

# Connect to PostgreSQL
psql -U postgres -d postgres

# In psql prompt
SELECT version();
\q
```

---

## SQL Fundamentals

### Creating Tables

```sql
-- Create a users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create an index on email for faster lookups
CREATE INDEX idx_users_email ON users(email);
```

### Inserting Data

```sql
-- Insert single row
INSERT INTO users (username, email, password_hash)
VALUES ('john_doe', 'john@example.com', 'hashed_password');

-- Insert multiple rows
INSERT INTO users (username, email, password_hash)
VALUES
  ('jane_doe', 'jane@example.com', 'hashed_password2'),
  ('bob_smith', 'bob@example.com', 'hashed_password3');

-- Insert with returning clause
INSERT INTO users (username, email, password_hash)
VALUES ('alice', 'alice@example.com', 'hashed_password4')
RETURNING id, username, created_at;
```

### Querying Data

```sql
-- Select all columns
SELECT * FROM users;

-- Select specific columns
SELECT id, username, email FROM users;

-- Select with WHERE clause
SELECT * FROM users WHERE email = 'john@example.com';

-- Select with multiple conditions
SELECT * FROM users 
WHERE created_at > '2024-01-01' 
  AND email LIKE '%@example.com';

-- Select with ORDER BY
SELECT * FROM users ORDER BY created_at DESC;

-- Select with LIMIT and OFFSET
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;

-- Select distinct values
SELECT DISTINCT email FROM users;

-- Aggregate functions
SELECT 
  COUNT(*) as total_users,
  MIN(created_at) as first_user,
  MAX(created_at) as latest_user
FROM users;
```

### Updating Data

```sql
-- Update single row
UPDATE users
SET email = 'newemail@example.com'
WHERE id = 1;

-- Update multiple rows
UPDATE users
SET updated_at = CURRENT_TIMESTAMP
WHERE created_at < '2024-01-01';

-- Update with returning clause
UPDATE users
SET username = 'john_updated'
WHERE id = 1
RETURNING id, username;
```

### Deleting Data

```sql
-- Delete specific row
DELETE FROM users WHERE id = 1;

-- Delete with conditions
DELETE FROM users WHERE created_at < '2020-01-01';

-- Delete all rows (be careful!)
TRUNCATE TABLE users;
```

### Altering Tables

```sql
-- Add a column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Modify column type
ALTER TABLE users ALTER COLUMN phone TYPE VARCHAR(50);

-- Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- Drop a column
ALTER TABLE users DROP COLUMN phone;

-- Add a constraint
ALTER TABLE users ADD CONSTRAINT check_username_length 
CHECK (char_length(username) >= 3);

-- Rename a column
ALTER TABLE users RENAME COLUMN username TO user_name;

-- Rename table
ALTER TABLE users RENAME TO app_users;
```

---

## Database Relationships

### One-to-One (1:1)

Each row in table A relates to exactly one row in table B.

```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL
);

-- User profiles table (1:1 with users)
CREATE TABLE user_profiles (
  id SERIAL PRIMARY KEY,
  user_id INTEGER UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  bio TEXT,
  avatar_url VARCHAR(255)
);
```

### One-to-Many (1:N)

Each row in table A relates to many rows in table B.

```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL
);

-- Posts table (many posts per user)
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Query with JOIN
SELECT u.email, p.title, p.created_at
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.id = 1;
```

### Many-to-Many (N:M)

Many rows in table A relate to many rows in table B. Requires a junction table.

```sql
-- Posts table
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  content TEXT
);

-- Tags table
CREATE TABLE tags (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL
);

-- Junction table (Posts and Tags - many-to-many)
CREATE TABLE post_tags (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

-- Query many-to-many relationship
SELECT p.title, t.name as tag
FROM posts p
JOIN post_tags pt ON p.id = pt.post_id
JOIN tags t ON pt.tag_id = t.id
WHERE p.id = 1;
```

### Self-Referencing Relationships

```sql
-- Comments table (comments can have replies)
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  parent_id INTEGER REFERENCES comments(id) ON DELETE CASCADE, -- Self-reference
  user_id INTEGER NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Query with self-join (get comment with its parent)
SELECT 
  c.id,
  c.content,
  p.content as parent_content
FROM comments c
LEFT JOIN comments p ON c.parent_id = p.id
WHERE c.post_id = 1;
```

---

## Indexes and Performance

### Types of Indexes

#### B-Tree Index (Default)

```sql
-- Create index on single column
CREATE INDEX idx_posts_user_id ON posts(user_id);

-- Create index on multiple columns (composite index)
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);

-- Create unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

#### Partial Index

```sql
-- Index only active posts
CREATE INDEX idx_active_posts ON posts(user_id, created_at)
WHERE status = 'active';
```

#### Full-Text Search Index

```sql
-- Add full-text search column
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- Create GIN index for full-text search
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);

-- Update search vector
UPDATE posts 
SET search_vector = 
  to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''));
```

### Query Performance

```sql
-- Explain analyze query plan
EXPLAIN ANALYZE
SELECT * FROM posts WHERE user_id = 1 ORDER BY created_at DESC;

-- Use EXPLAIN to see query plan without executing
EXPLAIN
SELECT * FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.email = 'john@example.com';
```

### Performance Tips

1. **Index frequently queried columns**
2. **Use appropriate data types** (smaller = faster)
3. **Avoid SELECT *** (select only needed columns)
4. **Use LIMIT for large result sets**
5. **Analyze query plans with EXPLAIN**
6. **Keep statistics updated**: `ANALYZE table_name;`

---

## Transactions and ACID

### ACID Properties

- **Atomicity**: All operations succeed or all fail
- **Consistency**: Database remains in valid state
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed changes persist

### Using Transactions

```sql
-- Begin transaction
BEGIN;

-- Perform operations
INSERT INTO users (email, username) VALUES ('user1@example.com', 'user1');
INSERT INTO posts (user_id, title) VALUES (1, 'First Post');
UPDATE users SET email = 'updated@example.com' WHERE id = 1;

-- Commit transaction (save changes)
COMMIT;

-- Or rollback (discard changes)
ROLLBACK;
```

### Transaction Isolation Levels

```sql
-- Set isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Isolation levels:
-- READ UNCOMMITTED - Can read uncommitted data
-- READ COMMITTED - Can only read committed data (default)
-- REPEATABLE READ - Same query returns same results
-- SERIALIZABLE - Highest isolation, prevents all anomalies
```

---

## Advanced Queries

### JOINs

```sql
-- INNER JOIN (default JOIN)
SELECT u.email, p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id;

-- LEFT JOIN (include all left table rows)
SELECT u.email, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.email;

-- RIGHT JOIN (include all right table rows)
SELECT u.email, p.title
FROM posts p
RIGHT JOIN users u ON p.user_id = u.id;

-- FULL OUTER JOIN (include all rows from both tables)
SELECT u.email, p.title
FROM users u
FULL OUTER JOIN posts p ON u.id = p.user_id;
```

### Subqueries

```sql
-- Subquery in WHERE clause
SELECT * FROM posts
WHERE user_id IN (
  SELECT id FROM users WHERE created_at > '2024-01-01'
);

-- Subquery in SELECT clause
SELECT 
  email,
  (SELECT COUNT(*) FROM posts WHERE posts.user_id = users.id) as post_count
FROM users;

-- EXISTS subquery
SELECT * FROM users u
WHERE EXISTS (
  SELECT 1 FROM posts p WHERE p.user_id = u.id
);
```

### Window Functions

```sql
-- ROW_NUMBER
SELECT 
  id,
  title,
  user_id,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as post_rank
FROM posts;

-- RANK and DENSE_RANK
SELECT 
  user_id,
  COUNT(*) as post_count,
  RANK() OVER (ORDER BY COUNT(*) DESC) as rank,
  DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) as dense_rank
FROM posts
GROUP BY user_id;
```

### Common Table Expressions (CTEs)

```sql
-- Simple CTE
WITH recent_posts AS (
  SELECT * FROM posts 
  WHERE created_at > CURRENT_DATE - INTERVAL '7 days'
)
SELECT * FROM recent_posts ORDER BY created_at DESC;

-- Recursive CTE (hierarchical data)
WITH RECURSIVE comment_tree AS (
  -- Base case
  SELECT id, content, parent_id, 1 as level
  FROM comments
  WHERE parent_id IS NULL
  
  UNION ALL
  
  -- Recursive case
  SELECT c.id, c.content, c.parent_id, ct.level + 1
  FROM comments c
  INNER JOIN comment_tree ct ON c.parent_id = ct.id
)
SELECT * FROM comment_tree;
```

### Aggregation and GROUP BY

```sql
-- Basic aggregation
SELECT 
  user_id,
  COUNT(*) as total_posts,
  MIN(created_at) as first_post,
  MAX(created_at) as latest_post
FROM posts
GROUP BY user_id;

-- HAVING clause (filter after grouping)
SELECT 
  user_id,
  COUNT(*) as post_count
FROM posts
GROUP BY user_id
HAVING COUNT(*) > 5;

-- Multiple aggregations
SELECT 
  DATE_TRUNC('month', created_at) as month,
  COUNT(*) as posts_count,
  COUNT(DISTINCT user_id) as unique_authors
FROM posts
GROUP BY month
ORDER BY month DESC;
```

---

## Best Practices

### 1. Naming Conventions

```sql
-- Tables: plural, lowercase with underscores
CREATE TABLE user_profiles (...);

-- Columns: lowercase with underscores
CREATE TABLE users (
  user_id SERIAL,
  email_address VARCHAR(255)
);

-- Indexes: prefix with idx_
CREATE INDEX idx_users_email ON users(email);
```

### 2. Use Constraints

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  age INTEGER CHECK (age >= 0 AND age <= 150),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 3. Use Appropriate Data Types

```sql
-- Good
created_at TIMESTAMP
age SMALLINT  -- Use SMALLINT for small numbers
is_active BOOLEAN

-- Bad
created_at VARCHAR(255)
age TEXT
is_active INTEGER  -- Use BOOLEAN instead
```

### 4. Backups

```bash
# Create backup
pg_dump -U postgres myapp > backup.sql

# Restore backup
psql -U postgres myapp < backup.sql
```

### 5. Security

```sql
-- Create user with limited privileges
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Grant specific privileges
GRANT SELECT, INSERT, UPDATE ON users TO app_user;
GRANT USAGE, SELECT ON SEQUENCE users_id_seq TO app_user;

-- Revoke privileges
REVOKE DELETE ON users FROM app_user;
```

---

## Mini Project: Blog Database

Design and create a complete database schema for a blog application.

### Requirements:
1. Users can create accounts
2. Users can write posts
3. Posts can have categories
4. Posts can have multiple tags
5. Users can comment on posts
6. Comments can have replies (nested comments)
7. Users can like posts and comments

### Schema Design:

```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  avatar_url VARCHAR(255),
  bio TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Categories table
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) UNIQUE NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  description TEXT
);

-- Posts table
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  category_id INTEGER REFERENCES categories(id) ON DELETE SET NULL,
  title VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE NOT NULL,
  excerpt TEXT,
  content TEXT NOT NULL,
  featured_image VARCHAR(255),
  status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived')),
  views INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  published_at TIMESTAMP
);

-- Tags table
CREATE TABLE tags (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL,
  slug VARCHAR(50) UNIQUE NOT NULL
);

-- Post-Tags junction table
CREATE TABLE post_tags (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

-- Comments table (self-referencing for replies)
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  parent_id INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Likes table (polymorphic - can like posts or comments)
CREATE TABLE likes (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  likeable_type VARCHAR(20) NOT NULL CHECK (likeable_type IN ('post', 'comment')),
  likeable_id INTEGER NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, likeable_type, likeable_id)
);

-- Create indexes
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_category_id ON posts(category_id);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_published_at ON posts(published_at);
CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);
CREATE INDEX idx_likes_user_id ON likes(user_id);
CREATE INDEX idx_likes_likeable ON likes(likeable_type, likeable_id);

-- Sample queries

-- Get all published posts with author and category
SELECT 
  p.id,
  p.title,
  p.slug,
  p.excerpt,
  u.username as author,
  c.name as category,
  p.created_at,
  COUNT(DISTINCT cm.id) as comment_count,
  COUNT(DISTINCT l.id) as like_count
FROM posts p
JOIN users u ON p.user_id = u.id
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN comments cm ON p.id = cm.post_id
LEFT JOIN likes l ON l.likeable_type = 'post' AND l.likeable_id = p.id
WHERE p.status = 'published'
GROUP BY p.id, p.title, p.slug, p.excerpt, u.username, c.name, p.created_at
ORDER BY p.created_at DESC;

-- Get post with all tags
SELECT 
  p.title,
  STRING_AGG(t.name, ', ') as tags
FROM posts p
LEFT JOIN post_tags pt ON p.id = pt.post_id
LEFT JOIN tags t ON pt.tag_id = t.id
WHERE p.id = 1
GROUP BY p.title;

-- Get nested comments for a post
WITH RECURSIVE comment_tree AS (
  SELECT 
    id,
    user_id,
    content,
    parent_id,
    0 as level,
    ARRAY[id] as path
  FROM comments
  WHERE post_id = 1 AND parent_id IS NULL
  
  UNION ALL
  
  SELECT 
    c.id,
    c.user_id,
    c.content,
    c.parent_id,
    ct.level + 1,
    ct.path || c.id
  FROM comments c
  JOIN comment_tree ct ON c.parent_id = ct.id
  WHERE c.post_id = 1
)
SELECT 
  ct.*,
  u.username
FROM comment_tree ct
JOIN users u ON ct.user_id = u.id
ORDER BY ct.path;
```

---

## Next Steps

✅ Complete the blog database mini project  
✅ Practice writing complex queries  
✅ Understand database relationships  
✅ Move to Module 4: TypeORM Deep Dive  

---

*You now have a solid foundation in PostgreSQL! 🗄️*

