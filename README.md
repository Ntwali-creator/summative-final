# Country Development Explorer

A web dashboard and Python CLI tool for exploring development indicators for over 200 countries, using the World Bank Open Data API.

**Live site:** https://www.james-creator.tech
**Demo video (max 2 min):** [PASTE YOUTUBE/VIMEO LINK]
**Repository:** https://github.com/Ntwali-creator/summative-final

---

## Purpose

Researchers, students and analysts often need to look up and compare development data (such as population, GDP and life expectancy) across countries. This app lets them search for a country, pick an indicator and view the results in one place, without digging through raw API output.

## Features

**Web app (`index.html`)**
- Search for a country by full name (e.g. "Rwanda")
- Select a development indicator: [list the indicators you included]
- [Sorting: describe what can be sorted]
- [Filtering: describe what can be filtered]
- [Data visualization: chart/table, and which library if any]
- Error messages when the API is down, returns no data, or the country isn't found: [describe what the user sees]

**CLI app (`cli_app.py`)**
- Menu-driven search for a single country
- Side-by-side comparison of two countries
- [Error handling you implemented, e.g. network failure, invalid input]

## Project Structure

```
summative-final/
├── index.html      # Web dashboard (frontend)
├── cli_app.py      # Python command-line interface
├── README.md       # Documentation
└── .gitignore      # Git ignore rules
```

## API Used and Credits

- **World Bank Open Data API:** https://datahelpdesk.worldbank.org/knowledgebase/topics/125589
- **Authentication:** none. The API is public, so there are no API keys or credentials in this repository.
- **Attribution:** all data is provided by the World Bank. The API is credited in the web app footer and in the CLI code comments.
- **Other resources:** [list libraries, fonts and tools, e.g. `requests` (https://requests.readthedocs.io), any chart library, Nginx, HAProxy, Let's Encrypt]

## Running Locally

### Web app
1. Clone the repository:
   ```bash
   git clone https://github.com/Ntwali-creator/summative-final.git
   cd summative-final
   ```
2. Open `index.html` in a modern web browser.
3. Enter a country name, choose an indicator and click **Search**.

### CLI app
1. Make sure Python 3 is installed, then install the dependency:
   ```bash
   pip install requests
   ```
2. Run the app:
   ```bash
   python3 cli_app.py
   ```
3. Follow the on-screen menu to search for a country or compare two countries.

## Deployment

The app runs on three servers, with DNS records on `james-creator.tech` pointing to each:

| Server | Hostname | IP | DNS record |
|--------|----------|----|-----------| 
| web-01 | 7163-web-01 | 18.135.29.103 | web-01.james-creator.tech |
| web-02 | 7163-web-02 | 18.133.181.216 | web-02.james-creator.tech |
| lb-01  | 7163-lb-01  | 18.130.139.174 | www.james-creator.tech, lb-01.james-creator.tech |

> Server IPs can change if the machines are re-provisioned. Check the current IPs on the dashboard, and update the DNS A records and the `backend` section of `haproxy.cfg` if they ever change.

### 1. Web servers (web-01 and web-02)

On each server, install Nginx and Git, then clone the app:

```bash
sudo apt update && sudo apt install -y nginx git
git clone https://github.com/Ntwali-creator/summative-final.git /home/ubuntu/summative-final
```

Edit `/etc/nginx/sites-available/default`:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /home/ubuntu/summative-final;
    index index.html;

    server_name _;

    # Stop the repo's .git history being served publicly
    location ~ /\.git { deny all; }

    location / {
        # Identifies which server answered each request
        add_header X-Served-By $hostname;
        try_files $uri $uri/ =404;
    }
}
```

Using `$hostname` (rather than hardcoding "web-01") means the exact same config file works on both servers.

Make the app directory readable by Nginx, then test and reload:

```bash
sudo chmod 755 /home/ubuntu
sudo chmod -R 755 /home/ubuntu/summative-final
sudo nginx -t && sudo systemctl reload nginx
```

Enable the firewall, allowing only SSH, HTTP and HTTPS:

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### 2. HTTPS certificate (lb-01)

Let's Encrypt issues certificates for real hostnames, so this uses the project's own domain:

```bash
sudo apt install -y certbot
sudo systemctl stop haproxy        # certbot needs port 80 free
sudo certbot certonly --standalone -d www.james-creator.tech --agree-tos -m <your-email> --non-interactive

sudo mkdir -p /etc/haproxy/certs
sudo bash -c 'cat /etc/letsencrypt/live/www.james-creator.tech/fullchain.pem /etc/letsencrypt/live/www.james-creator.tech/privkey.pem > /etc/haproxy/certs/site.pem'
sudo chmod 600 /etc/haproxy/certs/site.pem
```

HAProxy needs the certificate and private key combined into a single PEM file. This file, and anything under `/etc/letsencrypt`, is never committed to Git.

**Certificate renewal** (certificates expire after 90 days):

```bash
sudo systemctl stop haproxy
sudo certbot renew
sudo bash -c 'cat /etc/letsencrypt/live/www.james-creator.tech/fullchain.pem /etc/letsencrypt/live/www.james-creator.tech/privkey.pem > /etc/haproxy/certs/site.pem'
sudo systemctl start haproxy
```

### 3. Load balancer (lb-01)

`/etc/haproxy/haproxy.cfg`:

```haproxy
global
    ssl-default-bind-options no-sslv3
    tune.ssl.default-dh-param 2048
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend http_front
    bind *:80
    http-request redirect scheme https code 301 unless { ssl_fc }
    default_backend http_back

frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/site.pem
    default_backend http_back

backend http_back
    balance roundrobin
    server web-01 18.135.29.103:80 check
    server web-02 18.133.181.216:80 check
```

Validate and start:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

### 4. Verification

Each server adds an `X-Served-By` header, naming which one handled a given request:

```bash
curl -s http://18.135.29.103 | grep -i "<title>"
curl -s http://18.133.181.216 | grep -i "<title>"
curl -sI http://www.james-creator.tech | head -3
for i in 1 2 3 4 5 6; do curl -sI https://www.james-creator.tech | grep -i x-served-by; done
```

Expected: both servers return the app's title, HTTP returns a 301 redirect to HTTPS, and the `x-served-by` values alternate between `7163-web-01` and `7163-web-02`.

Round-robin sends each new request to the next server in turn. The `check` option makes HAProxy stop sending traffic to a server that goes down, so the site stays up if one web server fails.

## Challenges and Solutions

1. **Country codes vs. names.** The World Bank API uses codes (e.g. `RW`), but users type names (e.g. "Rwanda"). *Solution:* the app fetches the full list of countries at startup and matches the user's input to the right code before requesting data.
2. **SSH key mismatch after server re-provisioning.** When servers were rebuilt, the new machines only trusted a different SSH key than the one used before. *Solution:* tried each available key explicitly with `ssh -i <key>`, and recorded per-server entries in `~/.ssh/config` to avoid confusion going forward.
3. **Firewall allowed everything by default.** `ufw` wasn't installed, so nothing was actually blocked even though the required ports looked "open". *Solution:* installed `ufw`, set a default-deny policy for incoming traffic, and explicitly allowed only 22, 80 and 443.
4. **Browsers rejected the first HTTPS certificate.** An initial self-signed certificate (used for testing before a domain was available) triggered security warnings. *Solution:* once `james-creator.tech` was set up with the correct A records, obtained a trusted Let's Encrypt certificate for `www.james-creator.tech`.
5. **The `.git` folder was publicly readable.** A request to `/.git/config` returned the file instead of a 403. *Solution:* added `location ~ /\.git { deny all; }` to the Nginx config on both web servers.

## Security Notes

No API keys or credentials are needed or stored. The `.gitignore` excludes [list what it excludes, e.g. `__pycache__/`, `.env`, `*.pem`].

## Credentials for Grading

- API key: none required (public API)
- Login: none (the app has no authentication)

## Author

James Ntwali, African Leadership University
