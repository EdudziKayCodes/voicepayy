# Deploy VoicePay on Vultr

This is a separate deployment path. It does not change `app.py`, the Render Blueprint, or your existing Render service. Keep Render running until the Vultr copy has passed the checks below.

VoicePay is a payment **simulation**. No bank, PayPal, or Paystack transfers are connected.

## What this deployment uses

- Vultr Cloud Compute with Ubuntu 24.04 LTS.
- Nginx for public HTTPS traffic, forwarding to Gunicorn on `127.0.0.1:8000`.
- `systemd` to keep one Gunicorn worker running. One worker is required because conversation and pending-approval state are held in memory.
- SQLite at `/var/lib/voicepay/voicepay.sqlite3` on the instance's persistent disk.
- Firebase phone sign-in and the existing WebAuthn device approval. The site must use its final HTTPS hostname.

The SQLite database persists through ordinary app updates and restarts on the same instance. Back it up. Deleting or rebuilding the instance can still delete its local data. In-memory conversations and pending approvals are lost if the app process restarts.

## 1. Create the Vultr instance

In Vultr, create a Cloud Compute instance with Ubuntu 24.04, choose a region close to your users, and add your SSH **public** key during creation. Enable a limited non-root login if the deployment form offers it. Vultr says SSH keys should be attached during provisioning; reinstalling keys later can reinstall the instance.

Add a Vultr Firewall Group allowing inbound:

- TCP 22 from your own IP address, if possible.
- TCP 80 and 443 from the internet.

Do not open port 8000 publicly. Gunicorn listens only on localhost.

## 2. Point a domain at the server

Use a domain you control. Add an `A` record pointing the domain (for example, `voicepay.example.com`) to the Vultr instance's IPv4 address. Wait for DNS to resolve before requesting HTTPS.

## 3. Install the server packages

Connect over SSH, then run:

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-pip nginx certbot python3-certbot-nginx
sudo adduser --system --group --home /opt/voicepay --shell /usr/sbin/nologin voicepay
sudo install -d -o voicepay -g voicepay -m 0750 /opt/voicepay/current /opt/voicepay/venv
sudo install -d -o root -g voicepay -m 0750 /etc/voicepay
sudo install -d -o voicepay -g voicepay -m 0750 /var/lib/voicepay
```

## 4. Put the project on the server

Clone the GitHub repository into `/opt/voicepay/current`. For a private repository, configure a read-only deploy key for the `voicepay` account; do not put a personal access token in the clone URL or shell history.

```bash
sudo -u voicepay git clone YOUR_REPOSITORY_URL /opt/voicepay/current
sudo -u voicepay python3 -m venv /opt/voicepay/venv
sudo -u voicepay /opt/voicepay/venv/bin/pip install --upgrade pip
sudo -u voicepay /opt/voicepay/venv/bin/pip install -r /opt/voicepay/current/requirements.txt
```

If you update the Git checkout later, reinstall requirements if they changed, then restart the service.

## 5. Add secrets outside GitHub

Run the following from the repository checkout on the server:

```bash
cd /opt/voicepay/current
```

Create `/etc/voicepay/voicepay.env` directly on the server and enter the following values. Do not put this file in the repository:

```bash
sudoedit /etc/voicepay/voicepay.env
sudo chown root:voicepay /etc/voicepay/voicepay.env
sudo chmod 0640 /etc/voicepay/voicepay.env
```

Add `GROQ_API_KEY`, `RP_ID`, `ORIGIN`, `SETUP_TOKEN`, `FIREBASE_API_KEY`, `FIREBASE_AUTH_DOMAIN`, `FIREBASE_PROJECT_ID`, `FIREBASE_APP_ID`, `GOOGLE_APPLICATION_CREDENTIALS`, `ALLOW_RESET=0`, and `PROXY_FIX_HOPS=1`. Do not paste private values into GitHub.

Set `RP_ID` to the hostname only and `ORIGIN` to the full HTTPS origin. Keep `ALLOW_RESET=0` and `PROXY_FIX_HOPS=1`.

Place your Firebase Admin service-account JSON at `/etc/voicepay/firebase-admin.json`; set its owner and permissions:

```bash
sudo chown root:voicepay /etc/voicepay/firebase-admin.json
sudo chmod 0640 /etc/voicepay/firebase-admin.json
```

Use a service account from the same Firebase project as the Web app settings. Never copy this JSON into the repository.

## 6. Install the service and Nginx configuration

```bash
sudo install -o root -g root -m 0644 deploy/vultr/voicepay.service /etc/systemd/system/voicepay.service
sudo install -o root -g root -m 0644 deploy/vultr/nginx-voicepay.conf /etc/nginx/sites-available/voicepay
```

Edit `/etc/nginx/sites-available/voicepay` and replace `voicepay.example.com` with your actual domain. Enable the site and validate Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/voicepay /etc/nginx/sites-enabled/voicepay
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl daemon-reload
sudo systemctl enable --now voicepay
```

## 7. Enable HTTPS

After DNS points to Vultr and ports 80/443 are open, issue a Let's Encrypt certificate:

```bash
sudo certbot --nginx -d voicepay.example.com
```

Use your exact hostname. Certbot can configure HTTPS and the HTTP-to-HTTPS redirect. Confirm certificate renewal:

```bash
sudo certbot renew --dry-run
```

## 8. Configure Firebase and check the site

In Firebase Authentication, add the Vultr hostname to Authorized domains. Confirm Phone sign-in and the desired SMS region policy are enabled, and that billing is set up for live SMS.

Then visit:

- `https://YOUR_DOMAIN/` — the site.
- `https://YOUR_DOMAIN/api/health` — should return `{"ok":true}`.
- `https://YOUR_DOMAIN/api/countries` — should return a `countries` list.

Check logs if the service does not start:

```bash
sudo systemctl status voicepay --no-pager
sudo journalctl -u voicepay -n 100 --no-pager
sudo tail -n 100 /var/log/nginx/error.log
```

Register a device on the final HTTPS domain and test a simulated payment. Keep the Render deployment until sign-in, country loading, device registration, approval, and the history page work on Vultr.

## Updating the app

From the server, fetch and deploy the intended Git commit, install any changed dependencies, then restart:

```bash
sudo -u voicepay git -C /opt/voicepay/current pull --ff-only
sudo -u voicepay /opt/voicepay/venv/bin/pip install -r /opt/voicepay/current/requirements.txt
sudo systemctl restart voicepay
sudo systemctl status voicepay --no-pager
```

Back up `/var/lib/voicepay/voicepay.sqlite3` before upgrades. This single-instance setup does not support multiple Gunicorn workers or multiple Vultr servers; pending payments and conversation state are in process memory.
