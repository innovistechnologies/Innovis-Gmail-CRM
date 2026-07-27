# Innovis CRM Deployment on Oracle Cloud Free Tier

This guide will help you deploy the Innovis CRM workflow (n8n) on Oracle Cloud Free Tier, including a working webpage/dashboard and necessary integrations.

## Prerequisites

1. **Oracle Cloud Free Tier account** - Sign up at [https://www.oracle.com/cloud/free/](https://www.oracle.com/cloud/free/)
2. **Domain name** (optional but recommended for SSL) - Point your domain's A record to your VM's public IP.
3. **Supabase account** - You already have Supabase credentials in the workflow.
4. **LiveKit account** - The workflow includes hardcoded LiveKit credentials (you should replace with your own).
5. **Gmail account** - For OAuth2 credentials.

## Step 1: Create Oracle Cloud VM Instance

1. Log in to Oracle Cloud Console.
2. Navigate to **Compute > Instances**.
3. Click **Create Instance**.
4. Configure:
   - **Name**: `innovis-crm-vm`
   - **Image**: Canonical Ubuntu 22.04 LTS
   - **Shape**: Ampere A1 Flexible (choose 1 OCPU, 6 GB RAM - within free tier limits)
   - **Boot Volume**: 50 GB (enough)
   - **Networking**: Assign a public IP address.
   - **SSH Keys**: Add your SSH key for access.
5. Click **Create**.

## Step 2: Connect to VM and Install Dependencies

SSH into your VM:

```bash
ssh ubuntu@<your_vm_public_ip>
```

Update system and install Docker, Docker Compose, Nginx, and Certbot:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose nginx certbot python3-certbot-nginx
```

Start and enable Docker:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker  # or log out/in
```

## Step 3: Deploy n8n with PostgreSQL

Create a directory for the deployment:

```bash
mkdir -p ~/innovis-crm/deployment
cd ~/innovis-crm/deployment
```

Copy the `docker-compose.yml` and `dashboard` folder from your local repository (or create them manually).

### docker-compose.yml

```yaml
version: '3'

services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=
      - N8N_BASIC_AUTH_PASSWORD=
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_ENCRYPTION_KEY=your_secret_key_here_ChangeIt123!
      - N8N_DEFAULT_BASIC_AUTH_ACTIVE=true
      - N8N_DEFAULT_BASIC_AUTH_USERNAME=admin
      - N8N_DEFAULT_BASIC_AUTH_PASSWORD=password
      - N8N_LOG_LEVEL=info
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8n_password
    volumes:
      - ./n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    container_name: n8n_postgres
    restart: always
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=n8n_password
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
```

### Dashboard

Place the `index.html` file in a subfolder `dashboard/`.

Start the stack:

```bash
docker-compose up -d
```

Verify containers are running:

```bash
docker ps
```

You should see `n8n` and `n8n_postgres`.

## Step 4: Configure Nginx as Reverse Proxy

We'll set up Nginx to proxy:
- `https://yourdomain.com` → n8n (port 5678)
- `https://yourdomain.com/dashboard` → static dashboard

Create Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/innovis-crm
```

Paste the following (replace `yourdomain.com` with your domain or use IP if no domain):

```
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # n8n proxy
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }

    # Dashboard static files
    location /dashboard/ {
        alias /home/ubuntu/innovis-crm/deployment/dashboard/;
        try_files $uri $uri/ =404;
    }

    # Optional: serve dashboard root at /dashboard
    location = /dashboard {
        try_files /index.html =404;
        alias /home/ubuntu/innovis-crm/deployment/dashboard/;
    }
}
```

Enable the site and test Nginx configuration:

```bash
sudo ln -s /etc/nginx/sites-available/innovis-crm /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Step 5: Obtain SSL Certificate (if you have a domain)

If you have a domain pointing to your VM's IP, run:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow the prompts to obtain and install the certificate. Choose to redirect HTTP to HTTPS.

If you don't have a domain, you can access via HTTP using the IP address, but note that n8n will warn about insecure connection. For production, use a domain.

## Step 6: Configure n8n

1. Access n8n at `https://yourdomain.com` (or `http://<your_vm_ip>:5678` if no SSL).
2. Log in with the credentials set in `docker-compose.yml` (default: admin / password).
3. Click **Import** → **Workflow** → Upload the `Innovis Tech CRM - Gmail Bot + LiveKit Voice + Supabase Leads.json` file from your local repository.
4. After importing, you need to set up credentials:
   - **Gmail OAuth2**: Create Google Cloud project, enable Gmail API, create OAuth credentials, and add them in n8n under Credentials.
   - **Supabase**: Create a credential of type "Supabase API" with your Supabase URL and anon key (or service role key if needed).
   - **Groq API**: Create a Groq API key and add as credential.
   - **OpenAI API** (optional): If you want to use alternative LLMs.
   - **Google Gemini API** (optional): If you want to use Gemini.
5. Update the **LiveKit credentials** in the workflow: The "Generate Room Name" node has hardcoded API key and secret. Replace them with your own LiveKit credentials (or better, create a credential node for LiveKit and reference it).
6. Activate the workflow by toggling the switch on the top-right.

## Step 7: Test the Workflow

1. Send an email to the Gmail account you connected (from another address) with a query about Innovis services.
2. The workflow should:
   - Trigger on new unread email.
   - Extract sender info.
   - Fetch services from Supabase table `services`.
   - Use AI agent to generate a response, determine if voice call wanted, etc.
   - If voice call wanted, generate a LiveKit room and token, create a room via LiveKit API, and send back a join link.
   - Save lead to Supabase table `leads`.
   - Send reply email with the AI response and optionally a voice call link.

## Step 8: Verify Dashboard

Visit `https://yourdomain.com/dashboard/` (or `http://<your_vm_ip>:5678/dashboard/` if using IP) to see the leads dashboard showing leads from Supabase.

## Step 9: Maintenance

- **Backups**: Regularly snapshot the volumes `n8n_data` and `postgres_data` or export data.
- **Updates**: Pull latest n8n image and recreate container: `docker-compose pull && docker-compose up -d`.
- **Logs**: View logs with `docker logs -f n8n` and `docker logs -f postgres`.

## Notes

- Replace hardcoded secrets in the workflow with proper n8n credentials for security.
- The default n8n admin password is `password`; change it immediately after first login.
- Ensure your Supabase table `leads` exists with columns matching the workflow (customer_name, customer_email, message, service_interest, budget, urgency, lead_summary, status).
- The LiveKit HTTP endpoint in the workflow is set to `https://innovis-crm-p5b3q926.livekit.cloud`; you should change this to your own LiveKit server URL or use the default LiveKit cloud service with your own key/secret.

## Troubleshooting

- **n8n not accessible**: Check Docker container status, Nginx proxy, firewall rules (ensure ingress ports 80,443 are open in Oracle Cloud Security List).
- **Workflow not triggering**: Verify Gmail credentials and that the trigger is set to "unread".
- **Supabase connection errors**: Verify Supabase URL and anon key; ensure the table `leads` exists and is writable.
- **LiveKit errors**: Verify your LiveKit server URL and credentials.

---

### Deployment Complete

You now have a fully functional Innovis CRM with:
- Workflow automation (n8n) processing inbound emails.
- AI-powered responses using Groq (or other LLMs).
- Lead storage in Supabase.
- LiveKit-powered voice call integration.
- A public dashboard to view leads.
- Secure HTTPS access via Nginx and Let's Encrypt.

```