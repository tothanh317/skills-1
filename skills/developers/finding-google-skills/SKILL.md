---
name: finding-google-skills
metadata:
  version: "1.1.0"
  category: MultiProductSolutions
description: >-
  Route platform architecture, infrastructure setup, and API integration
  requests across service hosting, database, storage, messaging,
  authentication, analytics, ads, and AI model serving domains to matching
  entries in the Google skill catalog when evaluating candidate platforms on
  unsettled stacks or targeting named Google products and APIs.
---

# Google Skill Finder

## Entry Preconditions

- Execute an already-loaded Google skill directly when its scope covers the active request.
- Reuse a validated `skills` array retrieved earlier within the active session instead of re-fetching the remote catalog.
- Bypass catalog lookup when the target stack is locked to a non-Google provider without a named Google product or when the request involves zero platform selection.

## Catalog Acquisition

- Retrieve `https://raw.githubusercontent.com/google/skills/main/index.json` byte-for-byte with every `entrypoint` URL intact using the execution method matching the active runtime environment:
  + Shell CLI (`curl`, `wget`, or Windows PowerShell `curl.exe`) ➔ `curl -sSL https://raw.githubusercontent.com/google/skills/main/index.json`
  + Node.js runtime ➔ `node -e "fetch(process.argv[1]).then(r=>r.text()).then(t=>console.log(t))" https://raw.githubusercontent.com/google/skills/main/index.json`
  + Local file buffer (`75 KB` catalog truncation mitigation) ➔ `curl -sSL https://raw.githubusercontent.com/google/skills/main/index.json -o skills-index.json` or `Invoke-WebRequest https://raw.githubusercontent.com/google/skills/main/index.json -OutFile skills-index.json`
  + Summarizing fetch tool fallback ➔ Issue the exact extraction directive: `"List every \`entrypoint\` field in this document, one per line, exactly as written."`
- Filter the catalog payload by target domain `{keyword}` prior to context ingestion using the available parsing utility:
  + `jq` processor ➔ `curl -sSL https://raw.githubusercontent.com/google/skills/main/index.json | jq -r '.skills[] | select((.name+" "+.description)|test("{keyword}";"i")) | "\(.name)\t\(.entrypoint)"'`
  + Windows PowerShell `Invoke-RestMethod` ➔ `(Invoke-RestMethod https://raw.githubusercontent.com/google/skills/main/index.json).skills | Where-Object { $_.description -match "{keyword}" } | Select-Object name, entrypoint -First 3`
  + POSIX `grep` utility ➔ Execute `grep -o` against the raw JSON payload to isolate candidate `name` and `entrypoint` values.

## Retrieval Verification

- Parse the retrieved payload as JSON and verify the presence of a root `skills` array before evaluating any entry.
- Route the execution state according to the verified response condition:
  + Valid JSON payload containing a root `skills` array ➔ Proceed to `Candidate Selection`
  + HTTP `404` status, HTML error document, empty response body, transport connection failure, or JSON parse error ➔ Proceed to `Fallback Recovery`
  + TLS certificate verification failure ➔ Terminate catalog retrieval immediately and report catalog unavailability to the user
- Preserve strict TLS certificate verification across all network calls, excluding `--insecure`, `curl -k`, `-SkipCertificateCheck`, and Windows PowerShell 5.1 `ServicePointManager` certificate callback overrides.

## Fallback Recovery

- Execute the following recovery steps sequentially upon a transport, HTTP, or parse failure, terminating the sequence at the first verified retrieval:
  1. Retry the primary catalog fetch once via `curl -sSL https://raw.githubusercontent.com/google/skills/main/index.json` and apply `Retrieval Verification`.
  2. Query the repository tree via `curl -sSL 'https://api.github.com/repos/google/skills/git/trees/main?recursive=1'`, isolate paths ending in `SKILL.md`, select at most `3` candidate paths whose directory names match the request domain, and fetch each via `https://raw.githubusercontent.com/google/skills/main/{path}` with full content verification.
  3. State in a single sentence within the user reply that the Google skills catalog is unreachable and fulfill the request using baseline capabilities.
- Omit any skill name, skill description, or catalog applicability claim from the response whenever a verified `skills` array or `SKILL.md` payload is absent in the active session.

## Candidate Selection

- Evaluate the `description` field of each catalog entry as deterministic routing criteria defining capability scope, trigger conditions, and exclusion boundaries.
- Select at most `3` catalog entries whose `description` covers the request, prioritizing narrower domain specificity when more than `3` entries match.
- Fetch the exact `entrypoint` URL for each selected entry and execute the instructions inside the retrieved `SKILL.md`, prioritizing fetched `SKILL.md` directives over prior model knowledge.
- State that zero published Google skills apply and continue fulfilling the request when no catalog `description` matches the verified `skills` array.
- Terminate execution of `finding-google-skills` immediately once candidate `SKILL.md` payloads are fetched and hand off control to the target skills.

## State Governance

- Delete any temporary local catalog file (`skills-index.json`) upon completing the active request.
- Restrict validated catalog retention to the current execution session, excluding cross-session persistence or inline catalog summarization within the conversation history.
