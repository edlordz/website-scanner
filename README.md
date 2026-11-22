# AI-Assisted Website Scanner

Paste a Books to Scrape page URL into a local dashboard to check its title, meta description, main heading, and up to five links. When the scan finishes, the same tab displays the saved report: what passed, what needs review, the evidence, and a suggested first fix. A local Ollama model describes the page; JavaScript determines the check results.

## Workflows

| Workflow | Role |
| --- | --- |
| [Website Scanner v5](workflows/website-scanner-v5.json) | Accepts a dashboard submission or manual run, checks the submitted page, and saves the report. A dashboard submission returns to scan history after the save. |
| [Scan History v3](workflows/scan-history-v3.json) | Serves the URL form, groups saved scans by exact page URL, and shows the latest checklist, AI description, suggested fix, and earlier scan history. |

The dashboard currently accepts HTTPS URLs on `books.toscrape.com`. Different category pages work without changing a node. Support for other public domains is planned. Earlier workflow exports are available for reference.

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

Development runs covered the normal scan, an HTTP 404 and a connection failure at the initial fetch, and an unavailable Ollama endpoint. Separate link checks covered a 404, an empty link sample, and a failed link request. Successful and failed scans were saved and rendered in the browser. The red-dot behavior was checked with simulated failed and passing rescans for the same URL, then the temporary test data was removed. The scanner saved reports for the Travel category via webhook and the Mystery category via the dashboard. On the Mystery page, the title and H1 passed, the fetched HTML had no nonempty meta description, and all five sampled links returned HTTP 200. A later Mystery submission confirmed the same-tab return after saving. A fresh import on a separate instance has not yet been verified.

## Scope and limitations

The scanner checks fetched HTML without rendering JavaScript. It does not crawl the whole site, measure page speed, evaluate visual quality, submit forms, or apply fixes. Link selection uses the requested page URL as its base and does not account for HTML base tags or a different final page URL after redirects. Redirects can also take a selected link off the original origin. AI summaries can be inaccurate. Keep this prototype local; it is not designed for public URL submissions or authenticated client access.

## Roadmap

- Accept public website URLs beyond the demo domain while keeping local network services private.
- Show scan progress and explain when a page is blocked or there is too little HTML to assess.
- Track unresolved heading, title, and link findings across rescans. A skipped check should not clear an open issue.
- Add a site thumbnail or banner and test the documented setup on a clean instance.

Keep n8n backups and real scan data out of Git. Review any workflow export for credentials, private URLs, and pinned data before committing it. No project license has been selected.
