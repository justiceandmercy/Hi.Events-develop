# Hi.Events Deployment Guide: Railway + Netlify (Free, No Credit Card)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Users                                  │
└──────────────────────────┬────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │     Netlify CDN         │
              │  (Frontend - React SSR) │
              └────────────┬────────────┘
                           │ API Calls
              ┌────────────▼────────────┐
              │     Railway             │
              │  ┌──────────────────┐   │
              │  │ Web Service      │   │
              │  │ (Laravel + Nginx)│   │
              │  └──────────────────┘   │
              │  ┌──────────────────┐   │
              │  │ Queue Worker     │   │
              │  │ (Background Jobs)│   │
              │  └──────────────────┘   │
              │  ┌──────────────────┐   │
              │  │ Scheduler        │   │
              │  │ (Cron Jobs)      │   │
              │  └──────────────────┘   │
              └────────────┬────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │PostgreSQL│        │  Redis  │        │  S3/    │
   │(Railway) │        │(Railway)│        │  MinIO  │
   └─────────┘        └─────────┘        └─────────┘
```

---

## Step 1: Prepare Repository

```bash
cd /Users/fr.reweisantoun/Desktop/Hi.Events-develop

# Initialize git if needed
git init
git add .
git commit -m "Hi.Events deployment ready"

# Create GitHub repo
gh repo create hi-events --push --source=. --public
# or private if preferred
```

---

## Step 2: Deploy Backend on Railway

### 2.1 Create Railway Project

1. Go to [railway.app](https://railway.app) → Sign up with GitHub
2. **New Project** → **Deploy from GitHub repo**
3. Select `hi-events` repo
4. **Root Directory**: `backend`
5. **Add PostgreSQL** → One-click add
6. **Add Redis** → One-click add

### 2.2 Configure Environment Variables

In Railway dashboard → Variables tab, add:

```bash
# Core
APP_KEY=base64:YOUR_GENERATED_KEY
JWT_SECRET=YOUR_GENERATED_JWT_SECRET
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-app-name.railway.app
APP_FRONTEND_URL=https://your-site-name.netlify.app

# Database (auto-filled from PostgreSQL service)
DB_CONNECTION=pgsql
DATABASE_URL=${{Postgres.DATABASE_URL}}

# Redis (auto-filled from Redis service)
REDIS_HOST=${{Redis.HOST}}
REDIS_PASSWORD=${{Redis.PASSWORD}}
REDIS_PORT=${{Redis.PORT}}
QUEUE_CONNECTION=redis
CACHE_DRIVER=redis
SESSION_DRIVER=redis

# Mail (use Mailgun, SendGrid, or SMTP)
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=postmaster@yourdomain.com
MAIL_PASSWORD=your-mailgun-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@yourdomain.com
MAIL_FROM_NAME=Hi.Events

# Stripe (production keys)
STRIPE_PUBLIC_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# SSO (optional)
SSO_GOOGLE_CLIENT_ID=...
SSO_GOOGLE_CLIENT_SECRET=...

# Storage (optional - use Railway volume or S3)
FILESYSTEM_PUBLIC_DISK=local
FILESYSTEM_PRIVATE_DISK=local
```

**Generate keys locally:**
```bash
# APP_KEY
php -r "echo 'base64:'.base64_encode(random_bytes(32)).PHP_EOL;"

# JWT_SECRET
php -r "echo base64_encode(random_bytes(32)).PHP_EOL;"
```

### 2.3 Create Services

#### Web Service (Main)
- **Root**: `backend`
- **Start Command**: `php artisan migrate --force && php artisan serve --host=0.0.0.0 --port=$PORT`
- **Health Check**: `/health`

#### Worker Service
- **Root**: `backend` (same repo)
- **Start Command**: `php artisan queue:work --queue=default,webhook-queue,occurrences --sleep=3 --tries=3 --timeout=60`
- **No health check needed**

#### Scheduler Service
- **Root**: `backend` (same repo)
- **Start Command**: `php artisan schedule:work`
- **No health check needed**

---

## Step 3: Deploy Frontend on Netlify

### 3.1 Connect Repository

1. Go to [netlify.com](https://netlify.com) → Sign up with GitHub
2. **New Site** → **Import from Git** → Select `hi-events`
3. **Base Directory**: `frontend`
4. **Build Command**: `yarn install && yarn build`
5. **Publish Directory**: `dist/client`

### 3.2 Environment Variables

In Netlify dashboard → Site Settings → Environment Variables:

```bash
VITE_API_URL_CLIENT=https://your-app-name.railway.app
VITE_API_URL_SERVER=https://your-app-name.railway.app
VITE_FRONTEND_URL=https://your-site-name.netlify.app
VITE_STRIPE_PUBLISHABLE_KEY=pk_live_...
VITE_APP_NAME=Hi.Events
VITE_APP_LOGO_DARK=
VITE_APP_FAVICON=/favicon.svg
```

### 3.3 Custom Domain (Optional)

1. Netlify → Domain Settings → Add Custom Domain
2. Configure DNS (Netlify provides nameservers)
3. Enable HTTPS (automatic)

---

## Step 4: Configure Stripe Webhooks

### 4.1 Production Webhook

1. Stripe Dashboard → Developers → Webhooks → Add Endpoint
2. **URL**: `https://your-app-name.railway.app/api/public/webhooks/stripe`
3. **Events to subscribe**:
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
   - `charge.succeeded`
   - `charge.updated`
   - `charge.refunded`
   - `refund.created`
   - `refund.updated`
   - `account.updated`
   - `payout.paid`
   - `payout.updated`
   - `terminal.reader.action_failed` (for Box Office)
4. Copy **Signing Secret** → `STRIPE_WEBHOOK_SECRET` in Railway

### 4.2 Test Webhook

```bash
# From local machine with Stripe CLI
stripe listen --forward-to localhost:8000/api/public/webhooks/stripe
```

---

## Step 5: Verify Deployment

### 5.1 Health Checks

```bash
# Backend
curl https://your-app.railway.app/health
curl https://your-app.railway.app/api/public/instance

# Frontend
curl https://your-site.netlify.app
```

### 5.2 Test User Flow

1. Visit `https://your-site.netlify.app/auth/register`
2. Create account
3. Create event
4. Add free ticket (price = 0)
5. Test checkout
6. Verify email in Railway logs (MAIL_MAILER=log)

---

## Step 6: Production Optimizations

### 6.1 Enable OPcache (Railway)

Add to Railway variables:
```bash
PHP_OPCACHE_ENABLE=1
PHP_OPCACHE_MEMORY=256
```

### 6.2 Configure CDN (Optional)

For better performance, add Cloudflare:
1. Add domain to Cloudflare
2. Enable: Auto Minify (CSS, JS, HTML), Brotli, HTTP/3
3. Cache Level: Standard
4. Add Page Rule: `your-site.netlify.app/*` → Cache Level: Bypass (for dynamic content)

### 6.3 Monitoring

Railway provides:
- Metrics tab (CPU, RAM, Network)
- Logs tab (real-time)
- Alerts (configure for downtime)

Netlify provides:
- Deploy logs
- Function logs
- Analytics (enable in Site Settings)

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Build fails on `yarn install` | Ensure `yarn.lock` committed; check Node version |
| Migration fails | Check `DATABASE_URL` format; ensure PostgreSQL accessible |
| Queue worker stops | Check Redis connection; increase memory limit |
| CORS errors | Ensure `APP_FRONTEND_URL` matches Netlify URL exactly |
| Stripe webhook fails | Verify `STRIPE_WEBHOOK_SECRET`; check webhook URL |
| Assets 404 | Check `APP_CDN_URL` or ensure `storage:link` ran |
| Session lost | Ensure `SESSION_DRIVER=redis` and Redis accessible |

---

## Cost Summary (Free Tier)

| Service | Free Tier Limits |
|---------|------------------|
| Railway | $5/month credit (~500h hobby) |
| Netlify | 100GB bandwidth, 300 build mins |
| PostgreSQL | 1GB storage |
| Redis | 25MB storage |

**Total: $0/month** for small projects

---

## Next Steps

1. **Custom Domain**: Configure in Netlify + Railway
2. **Email**: Set up Mailgun/SendGrid for production emails
3. **Monitoring**: Add Sentry (free tier) for error tracking
4. **Backups**: Railway auto-backups PostgreSQL daily
5. **Scaling**: Upgrade Railway plan when needed

---

## Quick Commands Reference

```bash
# Deploy backend changes
git push origin main  # Railway auto-deploys

# Deploy frontend changes
git push origin main  # Netlify auto-deploys

# Run migration manually
railway run php artisan migrate --force

# Clear caches
railway run php artisan cache:clear && php artisan config:clear

# View logs
railway logs --service web
railway logs --service worker

# SSH into container (debugging)
railway shell --service web
```