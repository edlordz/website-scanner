# AI-Assisted Website Scanner

A local website audit prototype built with n8n, Ollama, and JavaScript. It combines evidence-based HTML checks with an AI-generated page summary and a small sample of HTTP link checks.

**Status: working demo, not a general-purpose website scanner or production service.** Version 1 targets the [Books to Scrape](https://books.toscrape.com/) homepage.

## What it does

- Fetches a page and extracts its title, body text, meta description, H1 headings, and links.
- Checks for a nonempty title, meta description, and H1 heading in the fetched HTML.
- Uses a local Ollama model to summarize the supplied page evidence.
- Removes duplicate eligible links and checks up to five.
- Classifies HTTP responses and produces one JSON report with evidence, timestamps, scope, and counts.

The AI writes the summary. JavaScript rules determine the presence checks and HTTP status classifications. No custom model training is involved.

## Workflow

| Node | Purpose |
| --- | --- |
| When clicking ‘Execute workflow’ | Start a manual scan |
| Scan Settings | Set the demo URL |
| Fetch Website | Download HTML |
| Extract Page Content | Extract page evidence |
| Check Page Basics | Run three presence checks |
| Analyse Page | Request a structured Ollama summary |
| Parse Analysis | Validate the summary and retain evidence |
| Prepare Links | Select five unique eligible demo links |
| Check Link Status | Request each selected URL |
| Classify Link Result | Classify its HTTP response |
| Build Scan Report | Combine the results |

## Run locally

The development setup uses Windows, Docker Desktop with a persistent n8n volume, and Ollama running on the host computer.

1. Start your local n8n container and Ollama.
2. Ensure your Ollama installation has the `qwen3:4b` model available. The tested installation supports thinking and structured JSON output; the workflow sends `think: true`.
3. Open n8n and import `workflows/website-scanner-v1.json` using its workflow import option.
4. Check the **Analyse Page** endpoint: `http://host.docker.internal:11434/api/chat`. This address connects the Docker container to Ollama on the host. Other deployments may need a different address or host networking configuration.
5. Leave **Scan Settings** at `https://books.toscrape.com/`. Version 1 deliberately rejects other addresses in **Prepare Links**.
6. Execute the workflow and inspect **Build Scan Report**.

Keep the node names unchanged unless you also update the code references. This export uses HTTP Request node version 4.5, Code version 2, and HTML version 1.2; the exact n8n application version was not recorded.

No paid AI API key is used. Running the software still requires your own hardware, internet connection, and electricity.

## Results and interpretation

- `pageChecks`: presence checks with evidence; missing elements are marked `needs_review`.
- `aiAnalysis`: page purpose and summary, which may contain model errors.
- `linkSummary`: counts of checked, passed, broken, review-needed, and unverified links.
- `linkResults`: source URL, requested URL, HTTP status, classification, reason, and timestamp.
- `scope` and `checksNotPerformed`: explicit coverage limits.

HTTP 2xx responses are marked `pass`; 404/410 are marked `broken`. Authentication failures and rate limiting are marked `unverified`. Server errors require review. A successful HTTP response does not prove that the page content or functionality is correct.

## Validation completed

Manual runs in the development environment confirmed:

- Structured Ollama output could be parsed and combined with page evidence.
- The selected five demo links returned 200 and were classified as `pass`.
- A temporary, deliberately nonexistent test address returned 404 and was classified as `broken`.
- The temporary test address was removed, and the combined report returned five passes.

These are manual observations, not a comprehensive automated test suite. The repository copy has had instance-specific metadata removed and whitespace trimmed from the Ollama URL. Its JSON and connections were validated, but this cleaned copy has not yet been re-imported and executed.

## Current limitations

- URL handling is restricted to selected relative paths on the demo homepage. Redirect destinations, HTML base tags, and general URL resolution are not handled.
- Only five eligible links are checked; no recursive crawling is performed.
- If no eligible links exist, the current workflow emits no link items and may not produce the final report.
- Network failures, timeouts, and Ollama failures can stop the workflow. HTTP error-response handling does not cover transport failures.
- Fetch failures are not yet gated before HTML checks; an error page could be analysed as content.
- JavaScript is not rendered. No visual, performance, accessibility, form, or booking tests are performed.
- There is no database, dashboard, scheduling, client authentication, or automated website repair.
- This is a local demo, not a public URL-submission service. Generalized fetching will need validation and network restrictions before public deployment.

## Planned improvements

1. General URL resolution and explicit scan limits.
2. Empty-link handling, fetch-status checks, and recoverable network errors.
3. Saved scan history and issue tracking.
4. A dashboard showing open, in-progress, resolved, and unverified findings.
5. Human-reviewed remediation suggestions and rescans.

## Export notes

The shared workflow contains no configured credentials or pinned execution data. Instance/workflow/version identifiers were removed; internal node IDs remain to preserve the workflow structure. The local Ollama hostname is configuration, not a password.

Before sharing future exports, inspect them again: credentials, private URLs, headers, and pinned data can change as the project grows. `.gitignore` cannot remove secrets embedded inside a tracked JSON file.

No project license has been selected yet. Dependencies retain their own licenses.
