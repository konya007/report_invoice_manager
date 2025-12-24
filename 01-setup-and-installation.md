# 01 - Setup & Installation Guide

> Hướng dẫn cài đặt môi trường development và production cho QLHoaDonWebVer2

---

## 📋 Mục lục

- [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
- [Thiết lập phát triển](#thiết-lập-phát-triển)
- [Cấu hình môi trường](#cấu-hình-môi-trường)
- [Thiết lập cơ sở dữ liệu](#thiết-lập-cơ-sở-dữ-liệu)
- [Các dịch vụ bên thứ ba](#các-dịch-vụ-bên-thứ-ba)
- [Chạy lần đầu](#chạy-lần-đầu)
- [Triển khai production](#triển-khai-production)

---

## Yêu cầu hệ thống

### Môi trường phát triển

**Tối thiểu:**
```
PHP >= 8.3
Composer >= 2.5
Node.js >= 18.x
MySQL >= 8.0
Git
```

**Khuyến khích:**
```
PHP 8.4+
Composer 2.8+
Node.js 20.x LTS
MySQL 8.4+
Redis 7+
```

### Operating Systems

- ✅ **macOS** (12 Monterey+)
- ✅ **Linux** (Ubuntu 22.04+, Debian 12+)
- ✅ **Windows** (WSL2 recommended)

### Software Tools

```bash
# Trình soạn thảo code
VS Code (khuyến nghị) / PHPStorm

# PHP Extensions (bắt buộc)
php-cli php-fpm php-mysql php-redis php-mbstring 
php-xml php-curl php-zip php-gd php-bcmath

# Database tools
MySQL Workbench / TablePlus / DBeaver

# API testing
Postman / Insomnia
```

---

## Thiết lập phát triển

### 1. Clone Repository

```bash
# SSH (khuyến nghị)
git clone git@github.com:PhatTrise/QLHoaDonWebVer2.git
cd QLHoaDonWebVer2

# HTTPS
git clone https://github.com/PhatTrise/QLHoaDonWebVer2.git
cd QLHoaDonWebVer2
```

### 2. Install PHP Dependencies

```bash
# Cài đặt các package Composer
composer install

# Nếu gặp lỗi memory limit:
php -d memory_limit=-1 /usr/bin/composer install
```

**Expected output:**
```
Installing dependencies from lock file
Package operations: 125 installs, 0 updates, 0 removals
  - Installing doctrine/inflector (2.0.8)
  ...
Generating optimized autoload files
```

### 3. Install Node Dependencies

```bash
# Cài đặt các package npm
npm install

# Hoặc dùng yarn
yarn install
```

**Expected packages:**
```
alpinejs @tailwindcss marked highlight.js ...
```

### 4. Create Environment File

```bash
# Copy example environment
cp .env.example .env

# Tạo application key
php artisan key:generate

# Tạo JWT secret
php artisan jwt:secret
```

---

## Cấu hình môi trường

### Core Settings

Edit `.env`:

```dotenv
# Application
APP_NAME="Quản lý hoá đơn"
APP_ENV=local                    # local, staging hoặc production
APP_DEBUG=true                   # nhớ đặt FALSE khi production!
APP_URL=http://localhost:8000
APP_TIMEZONE=Asia/Ho_Chi_Minh
APP_LOCALE=vi

# Generated automatically
APP_KEY=base64:...
JWT_SECRET=...
```

### Database Configuration

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=product_management   # Tên database
DB_USERNAME=root                 # MySQL user
DB_PASSWORD=                     # MySQL password
```

**Create database:**
```sql
CREATE DATABASE product_management 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;
```

### Session & Cache (Nếu có)

```dotenv
# Session (database recommended for multi-server)
SESSION_DRIVER=database
SESSION_LIFETIME=120             # Minutes

# Cache (file for dev, redis for production)
CACHE_STORE=file                 # file | redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# Queue (sync for dev, redis/database for production)
QUEUE_CONNECTION=sync            # sync | redis | database
```

---

## Các Dịch vụ Bên Thứ Ba

### 1. Gmail API (Phục vụ quét hóa đơn qua email)

**Bước 1: Google Cloud Console**

1. Truy cập [Google Cloud Console](https://console.cloud.google.com/)
2. Tạo dự án mới: "QLHoaDon"
3. Enable APIs:
   - Gmail API
   - Cloud Pub/Sub API
4. Create OAuth 2.0 credentials:
   - Application type: Web application
   - Authorized redirect URIs: `https://your-domain.com/auth/google/callback`

**Step 2: Configure .env**

```dotenv
GOOGLE_PROJECT_ID="your-project-id"
GOOGLE_CLIENT_ID="xxx.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET="GOCSPX-xxx"
GOOGLE_REDIRECT_URI="http://localhost:8000/auth/google/callback"
GOOGLE_PUBSUB_TOPIC="mail-checker"
```

**Step 3: Service Account (for Pub/Sub)**

Xem hướng dẫn chi tiết trong [Google Cloud Pub/Sub Documentation](https://cloud.google.com/pubsub/docs/quickstart-console)

### 2. Pusher (Real-time Notifications)

**Bước 1: Tạo tài khoản Pusher**

1. Đăng ký tại [pusher.com](https://pusher.com)
2. Tạo ứng dụng mới
3. Lấy thông tin xác thực (credentials)
**Bước 2: Cấu hình .env**

```dotenv
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=your-app-id
PUSHER_APP_KEY=your-app-key
PUSHER_APP_SECRET=your-app-secret
PUSHER_APP_CLUSTER=ap1           # Asia Pacific

# For frontend
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

**Test broadcasting:**
```bash
php artisan tinker
>>> broadcast(new App\Events\TestEvent());
```

### 3. Gemini AI (Chatbot)

**Bước 1: Lấy API Key**

1. Truy cập [ai.google.dev](https://ai.google.dev)
2. Lấy API key
3. Enable Gemini Pro model

**Bước 2: Cấu hình**

```dotenv
GEMINI_API_KEY="your-api-key-here"
GEMINI_MODEL="gemini-2.0-flash-exp"  # Or gemini-pro
```

**Test AI:**
```bash
php artisan ai:test-gemini
```

### 4. Cấu hình Mail

```dotenv
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME="your-email@gmail.com"
MAIL_PASSWORD="your-app-password"      # Không phải password Gmail!
MAIL_FROM_ADDRESS="noreply@yourcompany.com"
MAIL_FROM_NAME="${APP_NAME}"
```

> [!IMPORTANT]
> **Gmail App Password:** Không dùng password Gmail thường. Phải tạo App Password:
> 1. Google Account → Security → 2-Step Verification
> 2. App Passwords → Generate

---

## Thiết lập cơ sở dữ liệu

### 1. Chạy Migrations

```bash
# Fresh migration (⚠️ Xóa toàn bộ data!)
php artisan migrate:fresh

# Normal migration (production)
php artisan migrate
```

**Mẫu kết quả sau khi chạy lệnh:**
```
Migration table created successfully.
Migrating: 2024_01_01_000001_create_companies_table
Migrated:  2024_01_01_000001_create_companies_table (45.23ms)
...
```

### 2. Seed Database

```bash
# Seed demo company + admin user
php artisan db:seed --class=CompanyAndUserFromDumpSeeder

# Seed categories (product categories)
php artisan db:seed --class=CategorySeeder

# Hoặc seed all
php artisan db:seed
```

**Thông tin admin mặc định:**
```
Email: leductoan91@gmail.com
Password: 1
```

### 3. Xác nhận dữ liệu

```sql
-- Check tables
SHOW TABLES;

-- Check demo data
SELECT * FROM companies;
SELECT * FROM users;
SELECT * FROM categories;
```

---
## Chạy lần đầu

### 1. Tạo liên kết Storage

```bash
# Tạo symbolic link cho lưu trữ file
php artisan storage:link
```

Nó sẽ tạo liên kết từ thư mục `public/storage` đến `storage/app/public`. 

### 2. Build Frontend Assets

```bash
# Development (with watch)
npm run dev

# Production build
npm run build
```

### 3. Chạy server

**Cách A: Chay từng lệnh riêng biệt**

```bash
# Terminal 1: Laravel server
php artisan serve
# Running on http://localhost:8000

# Terminal 2: Queue worker
php artisan queue:work

# Terminal 3: Vite dev server
npm run dev
```

**Cách B: Concurrently (recommended)**
```bash
composer dev
```

This runs all servers với một command!

### 4. Access Application

```
Web UI:  http://localhost:8000
API:     http://localhost:8000/api
Swagger: http://localhost:8000/api/documentation
```

### 5. Login

```
URL: http://localhost:8000/login

Thông tin đăng nhập mặc định:
- Email: leductoan91@gmail.com
- Password: 1
```

---

## Triển khai Production

### Các việc cần làm trước khi triển khai 

```bash
# 1. Cập nhật .env cho môi trường production
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

# 2. Tối ưu cấu hình
php artisan config:cache
php artisan route:cache
php artisan view:cache

# 3. Build assets cho production
npm run build

# 4. Đặt quyền thư mục
chmod -R 755 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```
---

### Cấu hình tối ưu PHP & Cache

```bash
# Enable OPcache (php.ini)
opcache.enable=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000

# Use Redis for cache/session
CACHE_STORE=redis
SESSION_DRIVER=redis

# Queue jobs
QUEUE_CONNECTION=redis
```

### Debug Mode

```bash
# Bật Debugbar cho development
composer require barryvdh/laravel-debugbar --dev

# .env
DEBUGBAR_ENABLED=true
```

---

## Next Steps

✅ Môi trường development đã sẵn sàng!

**Continue to:**
- [Architecture Overview](02-architecture-overview.md) - Hiểu về kiến trúc
- [Development Workflow](03-development-workflow.md) - Bắt đầu phát triển
- [Auth & Middleware](04-auth-and-middleware.md) - Cấu hình bảo mật

---

## Quick Reference

```bash
# Start development
composer dev                     # Tất cả servers
# hoặc
php artisan serve & php artisan queue:work & npm run dev

# Database
php artisan migrate             # Run migrations
php artisan db:seed             # Seed data
php artisan migrate:fresh --seed  # Reset everything

# Cache
php artisan cache:clear         # Clear cache
php artisan config:cache        # Cache config (production)
php artisan optimize:clear      # Clear all caches

# Queue
php artisan queue:work          # Start worker
php artisan queue:restart       # Restart workers

# Assets
npm run dev                     # Watch mode
npm run build                   # Production build

# Testing
php artisan test                # Run tests
```

---
