# Country Development Explorer

A web dashboard and Python CLI tool for exploring development indicators for over 200 countries, using the World Bank Open Data API.

**Live (load balancer):** http://3.87.66.241
**Demo video (max 2 min):** [PASTE YOUTUBE/VIMEO LINK]
**Repository:** https://github.com/Ntwali-creator/summative-final

---

## Purpose

Researchers, students and analysts often need to look up and compare development data (such as population, GDP and life expectancy) across countries. This app lets them search for a country, pick an indicator and view the results in one place, without digging through raw API output. [Add one or two sentences on why you chose this problem.]

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
- **Other resources:** [list libraries, fonts and tools, e.g. `requests` (https://requests.readthedocs.io), any chart library, Nginx, HAProxy]

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

The app runs on three servers:

| Server | IP | Role |
|--------|----|------|
| web-01 | 3.82.191.8 | Nginx web server |
| web-02 | 3.82.162.246 | Nginx web server |
| lb-01 | 3.87.66.241 | HAProxy load balancer |

### 1. Web servers (web-01 and web-02)

On each server I installed Nginx and copied the project files to `/home/ubuntu/country-explorer`:

```bash
sudo apt update && sudo apt install -y nginx
git clone https://github.com/Ntwali-creator/summative-final.git /home/ubuntu/country-explorer
```

Then I edited `/etc/nginx/sites-available/default`:

```nginx
server {
    listen 80 default_server;
    root /home/ubuntu/country-explorer;
    index index.html;
    server_name _;

    # Identifies which server answered (change the value on web-02)
    add_header X-Served-By web-01;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

and reloaded Nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 2. Load balancer (lb-01)

I installed HAProxy and added this to `/etc/haproxy/haproxy.cfg`:

```haproxy
frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    server web-01 3.82.191.8:80 check
    server web-02 3.82.162.246:80 check
```

Then I restarted it:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

Round-robin sends each new request to the next server in turn, and `check` makes HAProxy stop sending traffic to a server that goes down.

### 3. Testing the load balancer

Each server adds an `X-Served-By` header, so I can see which one handled a request:

```bash
for i in 1 2 3 4; do curl -sI http://3.87.66.241 | grep X-Served-By; done
```

Expected output, alternating between servers:

```
X-Served-By: web-01
X-Served-By: web-02
X-Served-By: web-01
X-Served-By: web-02
```

[Paste your real output here. Optionally also describe stopping Nginx on one server and confirming the site stays up.]

## Challenges and Solutions

1. **Country codes vs. names.** The World Bank API uses codes (e.g. `RW`), but users type names (e.g. "Rwanda"). *Solution:* the app fetches the full list of countries at startup and matches the user's input to the right code before requesting data.
2. **HAProxy could not bind to port 80.** Nginx was already running on lb-01 and using the same port. *Solution:* I stopped and disabled Nginx on lb-01 (`sudo service nginx stop`) so HAProxy could use port 80.
3. [Add any other challenge you really faced, such as CORS, missing data for some countries, or API response format.]

## Security Notes

No API keys or credentials are needed or stored. The `.gitignore` excludes [list what it excludes, e.g. `__pycache__/`, `.env`].

## Credentials for Grading

- API key: none required (public API)
- Login: none (the app has no authentication)

## Author

James Ntwali, African Leadership University
