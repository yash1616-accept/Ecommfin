# Deployment Guide

## Prerequisites

1. **Database**: A PostgreSQL instance must be running and accessible
2. **Environment Variables**: Configured per environment (see reference below)

## Environment Variables Reference

Both `.env.example` files are included in the repo. Copy them to `.env` and fill in your values:

```bash
# Backend
cp backend/.env.example backend/.env

# Frontend
cp frontend/.env.example frontend/.env
```

### Backend Environment Variables

| Variable        | Required | Dev Default     | Production       | Description                                |
| --------------- | -------- | --------------- | ---------------- | ------------------------------------------ |
| `PORT`          | No       | `3000`          | Set by platform  | Express server port                        |
| `NODE_ENV`      | No       | `development`   | `production`     | Runtime environment                        |
| `DATABASE_TYPE` | Yes      | `postgresql`    | `postgresql`     | Database driver (`sqlite` or `postgresql`) |
| `DATABASE_URL`  | Yes      | Local PG string | Supabase/RDS URL | PostgreSQL connection string               |

### Frontend Environment Variables

| Variable       | Required | Dev Default                 | Production           | Description          |
| -------------- | -------- | --------------------------- | -------------------- | -------------------- |
| `VITE_API_URL` | Yes      | `http://localhost:3000/api` | Deployed backend URL | Backend API base URL |

> **⚠️ Important**: Vite embeds `VITE_*` variables at **build time**, not runtime. Set them in your platform **before** triggering a build. Changes require a rebuild to take effect.

### Platform-Specific Setup

| Platform           | Where to set env variables                      |
| ------------------ | ----------------------------------------------- |
| **Local**          | `backend/.env` and `frontend/.env`              |
| **Vercel**         | Dashboard → Settings → Environment Variables    |
| **AWS App Runner** | Service → Configuration → Environment Variables |

## Vercel Deployment

### First-Time Setup

1. **Connect Repository**: Link GitHub repo to Vercel
2. **Configure Environment Variables**: Add `DATABASE_URL` in Vercel dashboard → Settings → Environment Variables
3. **Configure Build Settings**:
   - Framework Preset: `Other`
   - Build Command: `npm run build`
   - Output Directory: Leave default (handled by vercel.json)
   - Install Command: `npm install`

### Database Migrations

**IMPORTANT**: Migrations are NOT run automatically to prevent data loss.

#### Resolve Database Drift (First Deployment)

Your database already has the correct schema but shows drift. Run this command locally with production DATABASE_URL:

```bash
cd backend
npx prisma migrate resolve --applied 20260217201708_stealth
```

This tells Prisma the migration is already applied without running it (safe for production).

#### Apply Future Migrations

```bash
# Option 1: Run locally with production DATABASE_URL
cd backend
npx prisma migrate deploy

# Option 2: Use Vercel CLI
vercel env pull .env.production
cd backend && npx prisma migrate deploy
```

### Deployment Steps

1. Resolve database drift (first time only - see above)
2. Push code to `main` branch
3. Vercel automatically builds and deploys
4. Apply migrations if needed (see above)
5. Verify deployment at your Vercel URL

## AWS App Runner Deployment

AWS App Runner requires **separate services** for frontend and backend. You need to create two App Runner services.

### Architecture Overview

- **Backend Service**: Node.js API server (Port 3000)
- **Frontend Service**: Static file server (Port 8080)
- **Database**: Supabase PostgreSQL (external)

### Configuration Files

Two configuration files have been created for you:

1. **[apprunner-backend.yaml](file:///c:/Users/PRATS/Playground/GDGC/ShopEase/apprunner-backend.yaml)** - For the backend API
2. **[apprunner-frontend.yaml](file:///c:/Users/PRATS/Playground/GDGC/ShopEase/apprunner-frontend.yaml)** - For the frontend UI

---

### Backend Deployment

#### 1. Create Backend Service

In AWS App Runner Console:

1. **Source**: Choose "Source code repository" (GitHub)
2. **Repository**: Select your ShopEase repository
3. **Branch**: `main`
4. **Configuration file**: Select **Use a configuration file**
5. **Configuration file path**: `apprunner-backend.yaml`

#### 2. Configure Environment Variables

Add these in App Runner → Configuration → Environment variables:

```env
DATABASE_URL=postgresql://postgres.xxxxx:password@aws-1-ap-south-1.pooler.supabase.com:5432/postgres
NODE_ENV=production
```

#### 3. Resolve Database Drift (First Time Only)

Before deploying, run this locally with your production DATABASE_URL:

```bash
cd backend
npx prisma migrate resolve --applied 20260217201708_stealth
```

#### 4. Deploy Backend

- App Runner will automatically build and deploy
- **Start Command**: `cd backend && npm start` (from apprunner-backend.yaml)
- **Port**: 3000
- **Health Check**: `/` or `/api/health` if you have one

---

### Frontend Deployment

#### 1. Create Frontend Service

In AWS App Runner Console:

1. **Source**: Choose "Source code repository" (GitHub)
2. **Repository**: Select your ShopEase repository
3. **Branch**: `main`
4. **Configuration file**: Select **Use a configuration file**
5. **Configuration file path**: `apprunner-frontend.yaml`

#### 2. Configure Environment Variables

Add in App Runner → Configuration → Environment variables:

```env
NODE_ENV=production
VITE_API_URL=https://your-backend-service.us-east-1.awsapprunner.com
```

> **Important**: Replace `VITE_API_URL` with your actual backend App Runner URL after the backend is deployed

#### 3. Deploy Frontend

- App Runner will automatically build and deploy
- **Start Command**: `cd frontend && npm start` (runs `npx serve dist -s -l 8080`)
- **Port**: 8080
- **Static Files**: Served from `frontend/dist`

---

### Important: Start Commands Explained

#### ❌ What NOT to Use

- **`npm run preview`** - This is for local development preview only, NOT production
- **Vite dev server** - Only for development, not production

#### ✅ What to Use

**Frontend:**

```bash
npm start
# Runs: npx serve dist -s -l 8080
# Serves static files from dist/ folder
# -s = Single Page Application mode (handles client-side routing)
# -l 8080 = Listen on port 8080
```

**Backend:**

```bash
npm start
# Runs: node dist/app.js
# Starts the compiled Express server
```

---

### Deployment Checklist

**Before First Deployment:**

- [ ] Resolve database drift: `npx prisma migrate resolve --applied 20260217201708_stealth`
- [ ] Commit and push configuration files to GitHub

**Deploy Backend First:**

1. [ ] Create App Runner service with `apprunner-backend.yaml`
2. [ ] Add `DATABASE_URL` environment variable
3. [ ] Wait for deployment to complete
4. [ ] Note the backend service URL (e.g., `https://xxx.us-east-1.awsapprunner.com`)

**Then Deploy Frontend:**

1. [ ] Create App Runner service with `apprunner-frontend.yaml`
2. [ ] Add `VITE_API_URL` pointing to backend service URL
3. [ ] Wait for deployment to complete
4. [ ] Access your frontend URL

---

### Database Migration (Manual)

Migrations must be run manually before deploying:

```bash
# Set DATABASE_URL environment variable
export DATABASE_URL="your-production-database-url"

# Resolve drift (first time only)
cd backend
npx prisma migrate resolve --applied 20260217201708_stealth

# Apply future migrations
npx prisma migrate deploy
```

---

### Troubleshooting AWS App Runner

#### Service Fails to Start

**Check Logs:**

- Go to App Runner service → Logs → Application logs
- Look for errors in the startup process

**Common Issues:**

1. **Port mismatch**
   - Ensure `network.port` in yaml matches the port your app listens on
   - Backend: Port 3000
   - Frontend: Port 8080

2. **Missing DATABASE_URL**
   - Error: "DATABASE_URL environment variable is not set"
   - Solution: Add DATABASE_URL in service environment variables

3. **Build failures**
   - Check build logs in App Runner
   - Ensure dependencies are in `dependencies`, not `devDependencies`

4. **Frontend shows blank page**
   - Check browser console for API errors
   - Verify `VITE_API_URL` is set correctly
   - Ensure backend service is running first

#### Frontend Can't Connect to Backend

- Verify `VITE_API_URL` environment variable is set
- Check CORS configuration in backend
- Ensure backend service is healthy

#### Database Connection Errors

- Verify `DATABASE_URL` is correct
- Check Supabase firewall/allowlist settings
- Ensure database is accessible from AWS region

## Important Commands

```bash
# Development setup (local only)
npm run setup

# Production build (CI/CD)
npm run build

# Apply migrations (production - manual)
cd backend && npm run deploy:migrate
```

## Troubleshooting

### "Drift detected" Error

Your database schema doesn't match migration history. Options:

- **For production** (preserves data):

  ```bash
  npx prisma migrate resolve --applied 20260217201708_stealth
  ```

- **For dev/staging** (deletes all data):
  ```bash
  npx prisma migrate reset
  ```

### Build Failures

- Verify `DATABASE_URL` is set correctly
- Check Prisma schema is valid: `npx prisma validate`
- Ensure all dependencies are in `dependencies`, not `devDependencies`

### Postinstall Issues

If postinstall hook causes issues, you can disable it temporarily:

```bash
npm install --ignore-scripts
npm run prisma:generate
```

## Local Production Testing

```bash
# Build everything
npm run build

# Start backend (requires DATABASE_URL in backend/.env)
cd backend && npm start

# Serve frontend
npx serve frontend/dist
```
