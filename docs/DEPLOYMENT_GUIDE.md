# Pay365 Deployment Guide: Render (Backend + Database) & Vercel (Frontend)

This guide walks you through deploying **Pay365** to production:
- **Backend & Database**: Deployed on [Render](https://render.com) (Node.js Express + Managed PostgreSQL)
- **Frontend**: Deployed on [Vercel](https://vercel.com) (Vite + React SPA)

---

## Architecture Overview & Pre-requisites

- **GitHub Repository**: Make sure your latest code (including the changes made for production) is pushed to your GitHub repository.
- **Render Account**: Free tier available at [render.com](https://render.com).
- **Vercel Account**: Free tier available at [vercel.com](https://vercel.com).

---

## Part 1: Deploy Backend & PostgreSQL on Render

You can deploy on Render using **Option A (Blueprint / Infrastructure-as-Code - Recommended)** or **Option B (Manual Dashboard)**.

### Option A: 1-Click Blueprint Deploy (Recommended)

1. Log in to your [Render Dashboard](https://dashboard.render.com).
2. Click **New +** > **Blueprint**.
3. Connect your GitHub repository.
4. Render will detect [`render.yaml`](file:///c:/Users/Aksh/Documents/GitHub/odoo-final/render.yaml) automatically.
5. In the blueprint configuration:
   - For `WEB_ORIGIN`: You can temporarily enter `http://localhost:5173` or your planned Vercel URL (e.g., `https://your-app.vercel.app`). You can update it anytime after deploying frontend.
6. Click **Apply**.
7. Render will automatically provision:
   - Managed PostgreSQL instance (`pay365-postgres`)
   - Web Service (`pay365-api`) with all environment variables (`DATABASE_URL`, auto-generated `SESSION_SECRET`, etc.), Prisma client generation, and schema migration.

---

### Option B: Manual Setup via Render Dashboard

If you prefer setting up manually without blueprints:

#### 1. Create PostgreSQL Database
1. In Render Dashboard, click **New +** > **PostgreSQL**.
2. Settings:
   - **Name**: `pay365-db`
   - **Database**: `pay365`
   - **User**: `pay365`
   - **Region**: Choose closest to you (e.g. Frankfurt, Oregon, Singapore).
   - **Plan**: Free (or Starter).
3. Click **Create Database**.
4. Once created, copy the **Internal Database URL** (if backend is on Render) or **External Database URL**.

#### 2. Create Backend Web Service
1. In Render Dashboard, click **New +** > **Web Service**.
2. Connect your GitHub repository.
3. Configure settings:
   - **Name**: `pay365-api`
   - **Region**: Same region as your database.
   - **Branch**: `main`
   - **Root Directory**: `backend`
   - **Runtime**: `Node`
   - **Build Command**:
     ```bash
     npm install && npm run build && npm run prisma:deploy
     ```
   - **Start Command**:
     ```bash
     npm start
     ```
   - **Plan**: Free
4. Under **Environment Variables**, add:
   | Key | Value | Notes |
   |---|---|---|
   | `NODE_ENV` | `production` | Enables production security policies |
   | `PORT` | `10000` | Render default port |
   | `DATABASE_URL` | `postgresql://...` | Paste your Render PostgreSQL URL |
   | `SESSION_SECRET` | *(Random 32+ char string)* | e.g. `openssl rand -hex 32` |
   | `SESSION_MAX_AGE_DAYS` | `7` | Session expiry |
   | `WEB_ORIGIN` | `https://<your-vercel-app>.vercel.app` | Frontend origin (update after Vercel deploy) |
   | `COOKIE_SAME_SITE` | `none` | Required for cross-site cookie authentication |

5. Click **Create Web Service**.
6. Note down your backend URL (e.g., `https://pay365-api.onrender.com`).

---

### Seeding Initial Demo Data on Render

Once your database migrations succeed, seed the default users and sample data:
1. In Render, go to your `pay365-api` service.
2. Open the **Shell** tab on the left menu.
3. Run:
   ```bash
   npm run db:seed
   ```
4. Default Demo Accounts seeded:
   - **Admin**: `admin@pay365.dev` / `Password@123`
   - **HR Manager**: `hr.manager@pay365.dev` / `Password@123`
   - **Payroll Manager**: `payroll.manager@pay365.dev` / `Password@123`
   - **Employee**: `employee@pay365.dev` / `Password@123`

---

## Part 2: Deploy Frontend on Vercel

1. Log in to your [Vercel Dashboard](https://vercel.com).
2. Click **Add New...** > **Project**.
3. Import your GitHub repository.
4. In the **Configure Project** screen:
   - **Framework Preset**: `Vite` (automatically detected)
   - **Root Directory**: Click **Edit** and select `frontend`.
   - **Build and Output Settings**: Defaults are correct (`npm run build`, output: `dist`).
5. Under **Environment Variables**, add:
   | Key | Value |
   |---|---|
   | `VITE_API_URL` | `https://<your-render-backend-url>/api/v1` |
   *(Example: `https://pay365-api.onrender.com/api/v1` - do not include trailing slash)*
6. Click **Deploy**.
7. Once deployed, Vercel gives you your production URL: `https://your-app.vercel.app`.

---

## Part 3: Final Connect (CORS & Origin Sync)

1. Copy your Vercel URL (e.g., `https://pay365-app.vercel.app`).
2. Go back to Render > `pay365-api` > **Environment**.
3. Set `WEB_ORIGIN` to your Vercel URL:
   ```text
   WEB_ORIGIN=https://pay365-app.vercel.app
   ```
   *(If you also want local development or preview branches, comma-separate them: `https://pay365-app.vercel.app,http://localhost:5173`)*.
4. Click **Save Changes** (Render will automatically redeploy the backend with the new setting).

---

## Verification & Health Check

1. **Backend Health Check**:
   Visit `https://<your-render-backend-url>/api/health` in your browser.
   Expected response:
   ```json
   { "status": "ok", "service": "pay365-api", "uptime": ... }
   ```
2. **Swagger Docs**:
   Available at `https://<your-render-backend-url>/api-docs` (restricted to ADMIN in production).
3. **Frontend App**:
   Visit `https://<your-vercel-app>.vercel.app`.
   - Log in with `admin@pay365.dev` / `Password@123`.
   - Refresh pages (e.g. `/employees`, `/contracts`) to verify client-side routing works cleanly without 404s.
