# Deploy kairo

Follow the steps **in order**. Copy and paste each command exactly. Lines starting with `#` are notes, not commands.
If something fails, jump to **Troubleshooting** at the bottom.

## What you are deploying

| Part | This export |
|------|-------------|
| Website / app | Next.js |
| Backend | Laravel (PHP) |
| Database | SQLite (file database/arkzen/kairo.sqlite) |
| Realtime | no |
| Background jobs | no |

## Deploy without Docker (a plain Linux server)

> Want the easy way? Re-export with Docker: `node mizumi.js export kairo nextjs --docker`. This section is for people who prefer to install everything by hand.

### Step 1 — Install on the server
- PHP 8.3 or newer with these extensions: pdo_sqlite (or pdo_pgsql for PostgreSQL), mbstring, openssl, bcmath, zip, intl, gd
- Composer 2
- Node.js 22
- Caddy (free web server that handles HTTPS): see caddyserver.com/docs/install

### Step 2 — Copy the project to the server
```bash
rsync -av --exclude node_modules --exclude vendor --exclude .venv --exclude target ./ root@YOUR_SERVER_IP:/srv/kairo/
```

### Step 3 — Database
This project uses SQLite: the database is the file `database/arkzen/kairo.sqlite` inside the backend folder. Nothing to install — just keep that file (back it up by copying it).

### Step 4 — Start the backend
```bash
cd backend
composer install --no-dev --optimize-autoloader
cp .env.example .env   # skip if backend/.env already exists
php artisan key:generate --force
php artisan migrate --force
php artisan storage:link
php artisan serve --host=0.0.0.0 --port=8001
```
It listens on port 8001.
### Step 5 — Build and start the website
```bash
cd frontend
npm ci
npm run build
npm start        # listens on port 3001
```
It listens on port 3001.
### Step 6 — Make it one public website (HTTPS)
Create `/etc/caddy/Caddyfile` and replace `yourdomain.com`:
```caddyfile
yourdomain.com {
  handle /api/* { reverse_proxy localhost:8001 }
  handle /storage/* { reverse_proxy localhost:8001 }
  handle { reverse_proxy localhost:3001 }
}
```
Then: `sudo systemctl reload caddy`. Point your domain's **A record** to the server first.

### Step 7 — Keep it running after you close the terminal
Create one service file per process, for example `/etc/systemd/system/kairo-backend.service`:
```ini
[Unit]
Description=kairo backend
After=network.target

[Service]
WorkingDirectory=/srv/kairo/backend
ExecStart=THE START COMMAND FROM THE BACKEND STEP
Restart=always
User=www-data

[Install]
WantedBy=multi-user.target
```
Then: `sudo systemctl daemon-reload && sudo systemctl enable --now kairo-backend`. Repeat for the website (change the name and the command).

## Troubleshooting

| Problem | What to do |
|---------|------------|
| Site shows 502 | The process behind Caddy is not running. `sudo systemctl status kairo-backend` and `journalctl -u kairo-backend -n 50` |
| HTTPS not working | A record must point to the server, ports 80 and 443 must be open |
| Command not found | A tool from Step 1 is missing or too old. Check versions |
