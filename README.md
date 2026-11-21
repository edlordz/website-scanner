# AI-Assisted Website Scanner

A local website audit prototype built with n8n, Ollama, and JavaScript. It checks one page and a small sample of its links, saves the report, and shows scan history and unresolved meta description findings in a browser. The demo target is [Books to Scrape](https://books.toscrape.com/).

| Workflow | What it does |
| --- | --- |
| [Website Scanner v4](workflows/website-scanner-v4.json) | Fetches a page, checks its title, meta description and H1, samples up to five eligible same-origin links, and saves one report per run. |
| [Scan History v2](workflows/scan-history-v2.json) | Reads saved reports, shows the newest first, and flags unresolved meta description findings. |

Older exports are kept as development milestones.

## Local setup

Developed on Windows with Docker Desktop, n8n **2.42.7**, and Ollama running on the host. No paid AI API key is required.

1. Start Ollama and make the model available with `ollama pull qwen3:4b`.
2. Start n8n. For a new Docker setup, run the following in PowerShell:

   ```powershell
   docker run -d --name n8n -p 127.0.0.1:5678:5678 -v n8n_data:/home/node/.n8n -e NODE_FUNCTION_ALLOW_BUILTIN=url docker.n8n.io/n8nio/n8n:2.42.7
   ```

   If you already have a working container, reuse it. The `url` module is used to resolve relative links. If you need to change container environment variables, preserve the data volume when recreating the container. An external task runner needs its own module configuration.

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

5. In **Website Scanner**, check that **Analyse Page** points to `http://host.docker.internal:11434/api/chat` for Docker Desktop, and leave **Scan Settings** on the demo URL for your first run. Do not rename nodes referenced by the workflow code.

## Run the demo

1. In **Website Scanner**, click **Execute workflow**. Check that **Save Scan Report** writes a `completed` row to `website_scans`.
2. In **Scan History**, click **Publish**. Open the Webhook node's **Production URL**, normally [http://localhost:5678/webhook/scan-history](http://localhost:5678/webhook/scan-history). The published URL is available while n8n is running; the Docker command above binds it to this machine only.
3. Expand a saved scan to see its findings or fetch failure reason. The test URL uses `/webhook-test/` and needs **Execute workflow** before each test request.

## How to read the results

- A **completed** scan means the workflow reached a report. It can still contain issues or unverified links. A failed fetch produces a report marked **failed**, with page and link checks skipped.
- An unavailable or invalid AI response does not stop the page and link checks. Ollama summarizes content; the JavaScript rules determine the findings.
- A link passes on HTTP 2xx; 404 or 410 is classified as broken. Request failures, denied access, and rate limits are unverified. Other unexpected responses need review. The attempted count includes unverified links.
- The red dot currently tracks missing meta descriptions by exact scanned URL. A later explicit pass for the same URL clears it; failed scans leave the previous finding open. Older reports stay in history.

## What was tested

Development runs covered the normal scan, an HTTP 404 and a connection failure at the initial fetch, and an unavailable Ollama endpoint. Separate link checks covered a 404, an empty link sample, and a failed link request. Successful and failed scans were saved and rendered in the browser. The red-dot behavior was checked with simulated failed and passing rescans for the same URL, then the temporary test data was removed. A fresh import on a separate instance has not yet been verified.

## Scope and limitations

The scanner checks fetched HTML without rendering JavaScript. It does not crawl the whole site, measure page speed, evaluate visual quality, submit forms, or apply fixes. Link selection uses the requested page URL as its base and does not account for HTML base tags or a different final page URL after redirects. Redirects can also take a selected link off the original origin. AI summaries can be inaccurate. Keep this prototype local; it is not designed for public URL submissions or authenticated client access.

Keep n8n backups and real scan data out of Git. Review any workflow export for credentials, private URLs, and pinned data before committing it. No project license has been selected.
