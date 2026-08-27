# n8n Workflows

Production-minded n8n automations I've built, exported with credentials stripped.

## radar-vacantes

A daily job radar. Pulls remote job postings from the Get on Board public API, scores each one against my profile with an LLM, and emails me only the matches above threshold.

### Why it exists

I was reading job boards manually every morning, most listings ruled out by country or seniority before I finished the first paragraph. This does the triage.

### Flow

Schedule (7am) → fetch 30 postings → split → free pre-filter → LLM scoring → output validation → merge → threshold filter → email

### Engineering decisions

**Free pre-filter before the LLM.** The API returns `remote` and `countries` as structured fields, so non-remote and country-restricted postings are dropped in a Code node before any model call. On a typical run this cuts 30 postings to 14 — roughly half the LLM quota saved at zero cost.

**Model choice: Gemini 3.6 Flash, free tier.** Scoring a posting against a fixed profile is a classification task; it does not need frontier capability. The LLM call is isolated in a single HTTP node and can be swapped for Claude or GPT by changing the endpoint and request body. Marginal cost at this volume: zero.

**Rate limiting is a design constraint, not an afterthought.** The free tier allows 5 requests per minute. Batching is set to 1 item every 15 seconds — 4/min, deliberately under the ceiling. I found this the hard way by exhausting the daily quota during development.

**Output validation.** LLMs return malformed JSON sometimes. The validation node parses the response, range-checks the score, and on failure returns a flagged record instead of throwing. A bad model response degrades one item; it does not kill the run.

**Error branch.** The LLM node uses `continueErrorOutput`. A 429 or 5xx routes the item down the error branch into the same validation node, where it lands as invalid. The workflow completes.

**Merge by key, not by position.** The scored records and the original posting data are rejoined on the posting `id`. Combining by position looked fine in testing but silently misaligns the moment one item fails — you'd get one posting's score attached to another posting's title.

### Running it

Import `workflows/radar-vacantes.json` into n8n. You'll need to add your own credentials:

- Google Gemini (PaLM) API — free tier from Google AI Studio
- Gmail OAuth2 — for the notification node

Adjust the profile text in the LLM node's request body to match your own background, and the threshold in the filter node.

### Status

Built and validated end to end.
