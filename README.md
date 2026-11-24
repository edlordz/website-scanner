# AI-Assisted Website Scanner

Paste a public HTTP or HTTPS page URL into a local dashboard to check its title, meta description, main heading, and up to five links. When the scan finishes, the same tab displays the saved report: what passed, what needs review, the evidence, and a suggested first fix. A local Ollama model describes the page; JavaScript determines the check results.

## Workflows

| Workflow | Role |
| --- | --- |
| [Website Scanner v6](workflows/website-scanner-v6.json) | Accepts a dashboard submission or manual run, checks the submitted page, and saves the report. A dashboard submission returns to scan history after the save. |
| [Scan History v4](workflows/scan-history-v4.json) | Serves the URL form, groups saved scans by exact page URL, and shows the latest checklist, AI description, suggested fix, and earlier scan history. |

Dashboard submissions use the supplied domain. Enter a full HTTP or HTTPS URL on its standard port, without embedded credentials. IP literals and local hostnames are not supported. Manual runs default to Books to Scrape. Earlier workflow exports are available for reference.

## Local setup

Developed on Windows with Docker Desktop, n8n **2.42.7**, and Ollama running on the host. No paid AI API key is required.

**Existing installation:** Start Docker Desktop and Ollama. If needed, run `docker start ollama-relay n8n-scanner`, then open the [dashboard](http://localhost:5678/webhook/scan-history). Keep older n8n containers that share the volume stopped.

**New installation (Windows PowerShell):**

1. Start Ollama and run `ollama pull qwen3:4b`. Create the Docker network and write the Nginx relay configuration. Keep the configuration file in place: Docker mounts it when the relay starts.

```powershell
docker network create scanner-ai
$relayDir = Join-Path $env:USERPROFILE "scanner-ai-relay"
New-Item -ItemType Directory -Force -Path $relayDir | Out-Null
$relayConfig = Join-Path $relayDir "default.conf"
@'
server {
    listen 8080;
    location = /api/chat {
        if ($request_method != POST) { return 405; }
        proxy_set_header Host localhost:11434;
        proxy_read_timeout 120s;
        proxy_pass http://host.docker.internal:11434/api/chat;
    }
    location / { return 404; }
}
'@ | Set-Content -Encoding ascii -LiteralPath $relayConfig
```

2. In the same PowerShell session, start both containers. Only n8n publishes a host port. n8n checks outbound HTTP requests and redirect destinations against its SSRF protection settings. The hostname exception is limited to the relay; the relay accepts only chat requests.

```powershell
docker run -d --name ollama-relay --network scanner-ai --restart unless-stopped --mount "type=bind,source=$relayConfig,target=/etc/nginx/conf.d/default.conf,readonly" nginx:stable-alpine
docker run -d --name n8n-scanner --network scanner-ai --restart unless-stopped -p 127.0.0.1:5678:5678 -v n8n_data:/home/node/.n8n -e NODE_FUNCTION_ALLOW_BUILTIN=url -e N8N_SSRF_PROTECTION_ENABLED=true -e N8N_SSRF_ALLOWED_HOSTNAMES=ollama-relay docker.n8n.io/n8nio/n8n:2.42.7
```

3. Open the [local n8n editor](http://localhost:5678). In the same project as both workflows, create a Data Table named `website_scans` with these columns:

   | Column | Type |
   | --- | --- |
   | `scanned_url` | String |
   | `scan_completed_at` | Date |
   | `scan_status` | String |
   | `report_json` | String |

4. Import both current workflow exports. The exports reference a table from the original installation; they do not create it or import its rows. In **Save Scan Report** and **Load Saved Scans**, reselect your local `website_scans` table. In **Save Scan Report**, verify these field expressions if you need to remap them:

   | Column | Expression |
   | --- | --- |
   | `scanned_url` | `{{ $json.scannedUrl }}` |
   | `scan_completed_at` | `{{ $json.scanCompletedAt }}` |
   | `scan_status` | `{{ $json.scanStatus }}` |
   | `report_json` | `{{ JSON.stringify($json) }}` |

5. In **Website Scanner**, check that **Analyse Page** sends its POST request to `http://ollama-relay:8080/api/chat`. Website fetches do not use this relay. Do not rename nodes referenced by the workflow code.

## Run the demo

1. Publish **Website Scanner** and **Scan History** in n8n. The scanner receives form submissions at `POST /webhook/scan-website`; the dashboard reads history at `GET /webhook/scan-history`. Keep Docker and n8n running.
2. On a new, empty table, seed the first report: run **Website Scanner** using its manual trigger. Its default URL is the Books to Scrape homepage. Confirm **Save Scan Report** writes a row.
3. Open the [local dashboard](http://localhost:5678/webhook/scan-history). Paste this category URL into **Website URL** and click **Scan page** once:

   ```text
   https://books.toscrape.com/catalogue/category/books/mystery_3/index.html
   ```

4. Leave the tab open while the scanner runs. After the report is saved, **Return to Dashboard** redirects the browser to scan history in the same tab. Look for the Mystery card. Scan that URL again to add an entry to its history.

For everyday use, bookmark the dashboard above. The n8n editor and PowerShell are unnecessary for submitting scans. The form and redirect use absolute localhost URLs; the Docker command above binds n8n to the local machine.

## How to read the results

- A **completed** scan means the workflow reached a report. It can still contain issues or unverified links. A failed fetch produces a report marked **failed**, with page and link checks skipped.
- An unavailable or invalid AI response does not stop the page and link checks. Ollama summarizes content; the JavaScript rules determine the findings.
- A link passes on HTTP 2xx; 404 or 410 is classified as broken. Request failures, denied access, and rate limits are unverified. Other unexpected responses need review. The attempted count includes unverified links.
- The red dot currently tracks missing meta descriptions by exact scanned URL. A later explicit pass for the same URL clears it; failed scans leave the previous finding open. Older reports stay in history.

## Validation

Development runs covered the normal scan, an HTTP 404 and a connection failure at the initial fetch, and an unavailable Ollama endpoint. Separate link checks covered a 404, an empty link sample, and a failed link request. Successful and failed scans were saved and rendered in the browser. The red-dot behavior was checked with simulated failed and passing rescans for the same URL, then the temporary test data was removed. The scanner saved reports for the Travel category via webhook and the Mystery category via the dashboard. On the Mystery page, the title and H1 passed, the fetched HTML had no nonempty meta description, and all five sampled links returned HTTP 200. A later Mystery submission confirmed the same-tab return after saving. On a separate public domain, the dashboard accepted the Phan Industries homepage without a workflow edit, displayed an AI description, and reported three passing page checks and five passing sampled links. These results describe that scan only. A fresh import on a separate instance has not yet been verified.

## Scope and limitations

The scanner checks fetched HTML without rendering JavaScript. It does not crawl the whole site, measure page speed, evaluate visual quality, submit forms, or apply fixes. Link selection uses the requested page URL as its base and does not account for HTML base tags or a different final page URL after redirects. Redirects can also take a selected link off the original origin. AI summaries can be inaccurate. Keep this prototype local; the dashboard accepts public website addresses but is intended for a local operator. It does not provide a hosted, multi-user client portal.

## Roadmap

- Show scan progress and explain when a page is blocked or there is too little HTML to assess.
- Track unresolved heading, title, and link findings across rescans. A skipped check should not clear an open issue.
- Add a site thumbnail or banner and test the documented setup on a clean instance.

Keep n8n backups and real scan data out of Git. Review any workflow export for credentials, private URLs, and pinned data before committing it. No project license has been selected.
