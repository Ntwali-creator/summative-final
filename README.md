# Country Development Explorer

This is a website and a small program. Both help you find facts about countries. The facts come from the World Bank. For example: population, GDP, life expectancy.

**Website:** https://www.james-creator.tech
**Demo video:** [PUT YOUR YOUTUBE OR VIMEO LINK HERE]
**Code:** https://github.com/Ntwali-creator/summative-final

---

## Why I made this

Students, teachers, and researchers often need to check facts about countries. They need to compare countries too. This app lets them type a country name, pick what they want to know, and see the answer fast. They do not need to search many websites.

## What the app can do

**Website (`index.html`)**
- You type a country name, like "Rwanda".
- You pick what you want to see (an indicator), like population or GDP.
- The app shows the data for that country.
- If something goes wrong (no internet, wrong name, API is down), the app shows a message. It does not just stay blank or crash.

**Command-line program (`cli_app.py`)**
- You run it in a terminal.
- It shows a menu.
- You can search one country.
- You can also compare two countries side by side.

## Files in this project

```
summative-final/
├── index.html      # The website
├── cli_app.py      # The terminal program
├── README.md       # This file
└── .gitignore      # Tells Git which files to ignore
```

## Where the data comes from

- **API:** World Bank Open Data — https://datahelpdesk.worldbank.org/knowledgebase/topics/125589
- **Do you need a password or key?** No. This API is free and open. Anyone can use it.
- All the data belongs to the World Bank. We only show it in a nicer way.

## How to run it on your own computer

### The website
1. Download the code:
   ```bash
   git clone https://github.com/Ntwali-creator/summative-final.git
   cd summative-final
   ```
2. Open the file `index.html` in any web browser (Chrome, Firefox, etc.).
3. Type a country name, choose what you want to see, and press Search.

### The terminal program
1. Make sure Python 3 is on your computer.
2. Install one small tool it needs:
   ```bash
   pip install requests
   ```
3. Run it:
   ```bash
   python3 cli_app.py
   ```
4. Follow the menu on the screen.

## How the website is hosted (the servers)

The website does not live on one computer. It lives on **three** computers, working together:

| Computer | What it does | Its address on the internet |
|---|---|---|
| web-01 | Shows the website | web-01.james-creator.tech |
| web-02 | Shows the website (a second copy) | web-02.james-creator.tech |
| lb-01 | Sends visitors to web-01 or web-02 | www.james-creator.tech |

**Why two website computers?** If one computer breaks or is too busy, the other one still works. Visitors never notice.

**Important:** these computers can change their IP address if they are restarted or rebuilt. Always check the IP on your server dashboard before you connect. If the IP changed, update:
1. The DNS records (where your domain name points), and
2. The `backend` part of the HAProxy file on lb-01 (shown further down).

### Step 1 — Set up web-01 and web-02

Do this on **both** computers, one at a time.

Install the web server (Nginx) and Git:
```bash
sudo apt update && sudo apt install -y nginx git
```

Download the website code:
```bash
git clone https://github.com/Ntwali-creator/summative-final.git /home/ubuntu/summative-final
```

Tell Nginx where to find the website files. Open this file:
```bash
sudo nano /etc/nginx/sites-available/default
```
and make it look like this:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /home/ubuntu/summative-final;
    index index.html;

    server_name _;

    # Do not show our Git history to the public
    location ~ /\.git { deny all; }

    location / {
        # This shows which computer answered. Helpful for testing.
        add_header X-Served-By $hostname;
        try_files $uri $uri/ =404;
    }
}
```

Let Nginx read the website files:
```bash
sudo chmod 755 /home/ubuntu
sudo chmod -R 755 /home/ubuntu/summative-final
sudo nginx -t && sudo systemctl reload nginx
```

Turn on the firewall. This blocks every door into the computer except the three we need:
```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### Step 2 — Get a free HTTPS certificate (on lb-01)

HTTPS means the connection is locked, like a locked envelope. Without it, anyone on the same Wi-Fi could read what visitors send. We use Let's Encrypt, which gives free certificates.

```bash
sudo apt install -y certbot

# Certbot needs port 80 free for a moment, so we stop HAProxy
sudo systemctl stop haproxy

sudo certbot certonly --standalone -d www.james-creator.tech --agree-tos -m YOUR_EMAIL_HERE --non-interactive

sudo mkdir -p /etc/haproxy/certs
sudo bash -c 'cat /etc/letsencrypt/live/www.james-creator.tech/fullchain.pem /etc/letsencrypt/live/www.james-creator.tech/privkey.pem > /etc/haproxy/certs/site.pem'
sudo chmod 600 /etc/haproxy/certs/site.pem
```

**Note:** change `YOUR_EMAIL_HERE` to your real email.

**This certificate is not forever.** It stops working after 90 days. To get a new one:
```bash
sudo systemctl stop haproxy
sudo certbot renew
sudo bash -c 'cat /etc/letsencrypt/live/www.james-creator.tech/fullchain.pem /etc/letsencrypt/live/www.james-creator.tech/privkey.pem > /etc/haproxy/certs/site.pem'
sudo systemctl start haproxy
```

### Step 3 — Set up the load balancer (lb-01)

This computer decides which website computer (web-01 or web-02) answers each visitor. The program that does this is called **HAProxy**.

File: `/etc/haproxy/haproxy.cfg`
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
    # If someone visits with plain http://, send them to https:// instead
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

Check the file is correct, then start HAProxy:
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

Turn on the firewall here too:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### Step 4 — Check that it all works

```bash
# Each web computer should show the website title
curl -s http://18.135.29.103 | grep -i "<title>"
curl -s http://18.133.181.216 | grep -i "<title>"

# Plain http:// should redirect to https://
curl -sI http://www.james-creator.tech | head -3

# Visiting many times should alternate between web-01 and web-02
for i in 1 2 3 4 5 6; do curl -sI https://www.james-creator.tech | grep -i x-served-by; done
```

What you should see: both website computers show the title, HTTP sends you to HTTPS, and the last test shows `web-01`, `web-02`, `web-01`, `web-02`... one after another.

"Round-robin" means HAProxy takes turns: first visitor to web-01, next to web-02, next to web-01, and so on. The word `check` in the config means HAProxy keeps testing each server. If one server stops answering, HAProxy stops sending visitors to it, until it comes back.

## Problems I had, and how I solved them

1. **Country names vs. country codes.** The World Bank API wants short codes, like `RW` for Rwanda, but people type full names. *Fix:* the app downloads the full list of countries first, then matches the name you typed to the right code.
2. **My SSH key stopped working after a server was rebuilt.** The new computer trusted a different key than before. *Fix:* I tried each key I had until one worked, and I wrote down which key goes with which server.
3. **The firewall did nothing at first.** The tool (`ufw`) was not even installed, so nothing was actually blocked, even though it looked "open" in tests. *Fix:* I installed it, blocked everything by default, then opened only ports 22, 80 and 443.
4. **Browsers did not trust my first HTTPS certificate.** I made my own certificate before I had a real domain name, so browsers showed a warning. *Fix:* once my domain `james-creator.tech` was ready, I got a free, trusted certificate from Let's Encrypt for it.
5. **My Git history was visible to everyone.** Anyone could open `/.git/config` on my website and see it. *Fix:* I added a rule in Nginx to block any request for `/.git/`.
6. **Server IP addresses kept changing.** Some servers got new IP addresses after a restart. *Fix:* I always double-check the IP on the dashboard before trusting old notes, and I update DNS and HAProxy together whenever an IP changes.

## About safety

- No password or API key is needed for this project. The World Bank API is open to everyone.
- There is no login on the website or the terminal program.
- The `.gitignore` file stops some files (like Python cache files) from being uploaded to GitHub by mistake.

## For the people grading this

- API key needed: **none**
- Login needed: **none**

## Made by

James Ntwali, African Leadership University (ALU)
