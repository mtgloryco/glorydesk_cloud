# GloryDesk Cloud API

GloryDesk Cloud is the multi-tenant backend service for **Glory Desk**, providing:
- **Cloud Database Delta Sync**: Push and pull local inventory entity changes across devices.
- **Database Backup & Restore**: Upload and download compressed gzip database snapshots.
- **Multi-Seat Licensing & Activations**: Issue, activate, and deactivate company licenses and hardware IDs.
- **Authentication & Multi-Tenancy**: Organization-scoped JWT authentication.
- **Embedded Web Portal**: Public pricing, documentation, activation, download, and admin portals in `wwwroot`.

---

## Deployment Options

### Option 1: Docker Compose (Recommended)

1. Copy the `glorydesk-cloud` directory to your server (or clone the repository).
2. Navigate into `glorydesk-cloud`:
   ```bash
   cd glorydesk-cloud
   ```
3. (Optional) Adjust `.env.example` or create a `.env` file with your custom secrets:
   ```bash
   cp .env.example .env
   # Edit .env with your JWT secret, Admin API key, and port
   ```
4. Start the backend with PostgreSQL:
   ```bash
   docker compose up -d --build
   ```
5. Check service health:
   ```bash
   curl http://localhost:8080/health
   ```

---

### Option 2: Standalone Docker (SQLite or External Postgres)

#### A. Running with embedded SQLite:
```bash
docker build -t glorydesk-cloud .
docker run -d \
  --name glorydesk-cloud \
  -p 8080:8080 \
  -v glorydesk_data:/app \
  -v glorydesk_backups:/app/backups \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e Jwt__Key="your-production-secret-at-least-32-characters" \
  -e Admin__ApiKey="your-admin-secret-key" \
  --restart unless-stopped \
  glorydesk-cloud
```

#### B. Running with an external PostgreSQL database (e.g. Supabase, Neon, AWS RDS):
```bash
docker run -d \
  --name glorydesk-cloud \
  -p 8080:8080 \
  -v glorydesk_backups:/app/backups \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e DATABASE_URL="postgresql://user:password@db-host:5432/glorydesk_cloud" \
  -e Jwt__Key="your-production-secret-at-least-32-characters" \
  -e Admin__ApiKey="your-admin-secret-key" \
  --restart unless-stopped \
  glorydesk-cloud
```

---

### Option 3: Direct Host Run (.NET 10 SDK / Runtime)

If your server has the .NET 10 SDK or ASP.NET Core Runtime installed:

1. Publish the application:
   ```bash
   dotnet publish GloryDesk.Cloud/GloryDesk.Cloud.csproj -c Release -o /var/www/glorydesk-cloud
   ```
2. Run directly or configure via `systemd`:
   ```bash
   export ASPNETCORE_URLS="http://0.0.0.0:8080"
   export ASPNETCORE_ENVIRONMENT="Production"
   export Jwt__Key="your-production-secret-at-least-32-characters"
   export DATABASE_URL="postgresql://..." # or omit for SQLite
   dotnet /var/www/glorydesk-cloud/GloryDesk.Cloud.dll
   ```

---

## Verifying & Testing the Backend

### 1. Health Check
```bash
curl -i http://<SERVER_IP>:8080/health
```
Response:
```json
{"status":"healthy","service":"Glory Desk Cloud API","utc":"..."}
```

### 2. Service Discovery Info
```bash
curl -i http://<SERVER_IP>:8080/api
```

### 3. Test Organization & User Registration
```bash
curl -i -X POST http://<SERVER_IP>:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@mycompany.com","password":"SecurePassword123!","organizationName":"My Enterprise"}'
```
Response:
```json
{"token":"<JWT_TOKEN>","userId":"...","organizationId":"...","email":"admin@mycompany.com","organizationName":"My Enterprise"}
```

### 4. Test User Login
```bash
curl -i -X POST http://<SERVER_IP>:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@mycompany.com","password":"SecurePassword123!"}'
```

### 5. Connecting the GloryDesk Desktop App
To point the GloryDesk Desktop application to your deployed backend, set the environment variable before launching the desktop client:

```bash
export GLORYDESK_CLOUD_API_URL="http://<SERVER_IP>:8080"
# (or https://your-domain.com if behind a reverse proxy like Nginx / Caddy / Cloudflare)
```

In the GloryDesk desktop application:
1. Open the sidebar and click **Cloud Sync**.
2. Enter your email, password, and organization name.
3. Click **Connect / Register**.
4. Use **Sync Now** to sync data, or **Backup** to upload an encrypted snapshot.

---

## Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `8080` | Port the API listens on |
| `DATABASE_URL` | No | _(empty -> SQLite)_ | PostgreSQL connection string |
| `Jwt__Key` | Yes (in Prod) | Dev key | Secret key for signing JWT tokens (min 32 chars) |
| `Jwt__Issuer` | No | `glorydesk-web` | Token issuer |
| `Jwt__Audience` | No | `glorydesk-clients` | Token audience |
| `Admin__ApiKey` | Recommended | `change-me-in-production` | Secret key for `X-Admin-Key` header |
| `Admin__AlertEmail` | No | `support@mtglory.com` | Alert notifications recipient |
| `License__PrivateKeyBase64`| No | _(empty)_ | Base64 RSA PKCS#8 private key for signing licenses |
| `BackupStoragePath` | No | `backups` | Directory for storing compressed backups |
| `Smtp__*` | No | _(empty)_ | SMTP email server configurations |
