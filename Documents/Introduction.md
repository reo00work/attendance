# Introduction

## Essential Configuration

### Setting of Timezone

Set time zone with this process:

```php
// config/app.php

'timezone' => 'Asia/Tokyo',
```

### Setting of Configuration Files for Login Function

#### .env

```php
# 修正前
SANCTUM_STATEFUL_DOMAINS=localhost

# 修正後
SANCTUM_STATEFUL_DOMAINS=localhost:3000,localhost
SESSION_DOMAIN=localhost  # 追加
```

### Queue Configuration

The password reset functionality uses Laravel queues for email sending to improve system responsiveness.

#### Queue Connection Setting

In `.env`:

```php
QUEUE_CONNECTION=database
```

Make sure the queue tables are migrated:

```bash
cd BackEnd
./vendor/bin/sail artisan migrate
```

#### Starting Queue Workers

**Development Environment (Laravel Sail):**

```bash
cd BackEnd
./vendor/bin/sail artisan queue:work
```

For background processing:

```bash
./vendor/bin/sail artisan queue:work --daemon
```

**Production Environment:**

Use Supervisor to manage queue workers persistently. Create `/etc/supervisor/conf.d/laravel-worker.conf`:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/attendance/BackEnd/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/path/to/attendance/BackEnd/storage/logs/worker.log
stopwaitsecs=3600
```

Then reload Supervisor:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

#### Queue Monitoring

```bash
# Check queue status
./vendor/bin/sail artisan queue:monitor

# Check failed jobs
./vendor/bin/sail artisan queue:failed

# Retry all failed jobs
./vendor/bin/sail artisan queue:retry all
```

---

## Payroll Batch (payroll:process) - How to Set Up

This section explains how the monthly payroll batch (`payroll:process`) is configured and how to run it in **development** and **production** environments.

The batch itself is implemented as a Laravel Artisan command:

- Command name: `payroll:process`
- Command class: `App\Console\Commands\ProcessMonthlyPayroll`
- Scheduled in: `App\Console\Kernel::schedule()`

```php
// App\Console\Kernel
protected function schedule(Schedule $schedule): void
{
    $schedule->command('payroll:process')
        ->timezone('Asia/Tokyo')
        ->dailyAt('01:00');
}
```

This means:

- The command will run **every day at 01:00 (Asia/Tokyo)** when the Laravel scheduler is active.

---

## Setup Guide

### 1. Prerequisites
- Windows users must use WSL2 (Ubuntu) and Docker Desktop with WSL Integration enabled.
- Start Docker Desktop before working in WSL.
- Convenience tip: add an alias to simplify Sail commands:
  ```
  echo 'alias sail="./vendor/bin/sail"' >> ~/.bashrc
  source ~/.bashrc
  ```

### 2. Items to obtain from the administrator
- Environment file (.env): the repository does NOT include .env. Obtain the latest .env contents from the administrator and create `.env` in the project root.
- Database dump (SQL): obtain the initial SQL dump from the administrator to populate the development database for realistic testing.

### 3. Backend setup (Laravel Sail) & troubleshooting
1. Place the received `.env` into the project root. Important: do not commit it to Git.
2. If you created/edited `.env` on Windows, convert CRLF → LF in WSL to avoid Sail errors:
   ```
   cd /path/to/attendance
   dos2unix .env
   ```
3. Ensure the DB host in `.env` matches the Sail MySQL container:
   ```
   DB_HOST=mysql
   ```
4. Start Sail (from BackEnd directory or project root with alias):
   ```
   cd BackEnd
   ./vendor/bin/sail up -d
   # or, if alias set:
   # sail up -d
   ```
5. Verify DB connectivity, then run migrations:
   ```
   ./vendor/bin/sail artisan db:monitor
   ./vendor/bin/sail artisan migrate
   ```
6. Import the provided database dump (example: place dump as BackEnd/dump.sql):
   ```
   # from project root
   ./vendor/bin/sail exec mysql bash -c "mysql -u${DB_USERNAME} -p${DB_PASSWORD} ${DB_DATABASE} < /var/www/html/BackEnd/dump.sql"
   ```
   - Ensure `${DB_USERNAME}`, `${DB_PASSWORD}`, and `${DB_DATABASE}` in `.env` match the dump target.
   - Alternatively, adjust paths/credentials per your environment.

### 4. Frontend setup
1. Change into the FrontEnd directory:
   ```
   cd FrontEnd
   ```
2. Install dependencies and start the dev server:
   ```
   npm install
   npm start
   ```

Notes
- Run all Docker / Sail commands from WSL (Ubuntu) to ensure integration with Docker Desktop.
- If you encounter connection or permission errors, first re-check `.env` line endings and that `DB_HOST=mysql`.
- Keep `.env` and dump files out of source control; treat them as sensitive artifacts.

---

### Email Configuration for Payslip Delivery

Make sure your email configuration is properly set in `.env`:

```php
MAIL_MAILER=smtp
MAIL_HOST=your-smtp-host
MAIL_PORT=587
MAIL_USERNAME=your-email@example.com
MAIL_PASSWORD=your-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

For development with Laravel Sail, you can use Mailpit for testing:

```php
MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
```

Mailpit interface is available at: `http://localhost:8025`

