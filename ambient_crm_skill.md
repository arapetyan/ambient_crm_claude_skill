---
name: aicrm-llm-graph
version: "2.6"
description: >
  Ambient CRM
---

# Ambient CRM PoC — Full Mailbox Pipeline

This PoC aims to test hypothesis that behavioral sales analysis of the inbound communication enough to build agentic sales enablement and operations. 
Advanced use of this PoC leads to extraction of data about: deals, clients, users, action items, blockers, tasks, responsibilities. This PoC can be enhanced to incorporate specific sales processes of the organization, operate with the context of products and services, enable gap analysis. PoC is intentionally not optimized for token use to speed up the testing process. Author is aware of connectors, light weight claws, microgpts, autoresearch capabilities. They are in MVP version of this PoC along with multi-mailbox and multi-chanell processing.

This skill is complex and best suited for the indepth analysis of the structure for MVP. It may fail to execute on all details because the model can run out of "attention budget" by the time it finished the heavy pipeline work (email parsing, graph building, dashboard generation) and truncated the output before reaching the follow-up questions step. Long skill executions can cause the model to wrap up prematurely after the main work is done. Skill analysis itself is benefitial if you plan to create a set of skills to support your own ambient crm version. However overall execution should match 90%+ level of quality and be enough for the test run.

## Version History

| Version | Scope |
|---------|-------|
| 1.0 | Manual file input, single run, two output files |
| 2.0 | Live mailbox via Cowork connector, checkpoint system, on-demand incremental sync, merge/diff action items, dialog + file reporting. Junk/Trash excluded. |
| 2.1 | Standalone CRM paradigm — email is the system of record, not a satellite feed to an external CRM. Company and person nodes enriched with CRM-grade account fields. Relationship health scoring. CRM Query Interface module. Manual input with provenance tagging. |
| 2.2 | PoC credibility layer — deal economics (amount, close_date, probability, forecast_category, weighted_value), contact roles on deals (champion/economic buyer/technical buyer/blocker), dark period detection at account level, meeting-as-signal (CALENDAR_INVITE → deal stamp + 7-day follow-up window). |
| 2.3 | Dashboard Generator — `generate dashboard` command produces self-contained `aicrm_dashboard.html` with embedded graph data. Four tabs: Summary (KPIs + dark period alerts), Pipeline (grouped by forecast_category with flags), Action Items (filterable table), Accounts (health bar + open items). No external dependencies, no server, no separate skill. |
| 2.4 | Output cleanup — removed internal file path lines (chunk graph, action items graph, checkpoint) from run output. Dashboard auto-generated at end of every run; single dashboard link is the only file reference returned to the user. |
| 2.5 | Merge from branch: PII tokenization module (reversible token map — EMAIL_NNN / PERSON_NNN / PHONE_NNN / DOMAIN_NNN, stored in checkpoint); person alias resolution (deterministic union-find merge, no LLM); deal entity module (stage lifecycle derived from commercial events, stalled detection at 45 days, no LLM); dashboard auto-generated on every run (not on-demand only); pii_token_map + pii_counters persisted in checkpoint. |
| 2.6 | Redesigned post-run return message: ⚡ Priority 0 (COMMITMENT_OVERDUE + BLOCKED items) · 📋 Priority 1 (all other active items) · 🔄 What Changed section (incremental runs only — new items + newly resolved) · single dashboard link as final line. handle_generate_dashboard() made silent when called from pipeline (silent=True); only prints when invoked via manual 'generate dashboard' command. |

---

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  FIRST RUN  (user invokes skill; no checkpoint exists)              │
│                                                                     │
│  Step 0  Connect mailbox via Cowork email connector                 │
│          Fetch all emails — INBOX, Sent, all folders                │
│          ✗ SKIP: Junk, Trash, Deleted Items, Spam                   │
│              │                                                      │
│              ▼                                                      │
│  Step 1  CHECK FOR NEW MAILS                                        │
│          All emails are "new" on first run (no checkpoint)          │
│          Normalize → deduplicate threads                            │
│              │                                                      │
│              ▼                                                      │
│  Step 2  UPDATE CHUNK GRAPH                                         │
│          Build: thread / chunk / person / company nodes             │
│          Build: SAME_THREAD, REPLIES_TO, SENT_BY,                   │
│                 RECEIVED_BY, WORKS_AT, CROSS_REF edges              │
│          ► Write: aicrm_chunk_graph.json                            │
│              │                                                      │
│              ▼                                                      │
│  Step 3  UPDATE ACTION ITEMS                                        │
│          Traverse chunk graph oldest→newest per thread              │
│          Extract: persons (with roles), companies (with types),     │
│                   events, action items, dependency edges            │
│          ► Write: aicrm_action_items_graph.json                     │
│              │                                                      │
│              ▼                                                      │
│  Step 4  REPORT open action items                                   │
│          ► Print summary in dialog only (no file output)            │
│              │                                                      │
│              ▼                                                      │
│  Step 5  CHECKPOINT + DASHBOARD                                     │
│          ► Write: aicrm_checkpoint.json  (incl. pii_token_map)      │
│          ► Write: aicrm_dashboard.html   (auto-generated every run) │
│          ► Print: dashboard link only                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  SUBSEQUENT RUN  (user invokes skill again; checkpoint exists)      │
│                                                                     │
│  Step 1  CHECK FOR NEW MAILS                                        │
│          Read checkpoint → fetch only new emails                    │
│          (Message-ID not in checkpoint.processed_message_ids)       │
│          ✗ SKIP: Junk, Trash, Deleted Items, Spam                   │
│              │                                                      │
│              ▼  (if no new emails → report "up to date", exit)      │
│              │                                                      │
│              ▼                                                      │
│  Step 2  UPDATE CHUNK GRAPH                                         │
│          Append new chunks to existing threads                      │
│          Add new threads, persons, companies                        │
│          Update CROSS_REF edges                                     │
│          ► Overwrite: aicrm_chunk_graph.json                        │
│              │                                                      │
│              ▼                                                      │
│  Step 3  UPDATE ACTION ITEMS                                        │
│          Re-run Phase 2 ONLY for threads that received new chunks   │
│          Match items by stable_id → update status in place          │
│          Append genuinely new items                                 │
│          ► Overwrite: aicrm_action_items_graph.json                 │
│              │                                                      │
│              ▼                                                      │
│  Step 4  REPORT changes since last run                              │
│          ► Print delta in dialog only (no file output)              │
│              │                                                      │
│              ▼                                                      │
│  Step 5  CHECKPOINT + DASHBOARD                                     │
│          ► Write: aicrm_checkpoint.json  (incl. pii_token_map)      │
│          ► Write: aicrm_dashboard.html   (auto-generated every run) │
│          ► Print: dashboard link only                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Required Output Files — Always All Four

Every run (first or incremental) MUST produce or update all four files.
The action items summary is printed to the dialog only — no report file is written.

| File | Updated on | Contents |
|------|-----------|---------|
| `aicrm_chunk_graph.json` | Every run | Structural graph: threads, chunks, persons, companies, edges |
| `aicrm_action_items_graph.json` | Every run | Semantic graph: persons (with roles), companies (with types), events, action items, deals, edges |
| `aicrm_action_items_graph.json.bak` | Every run (if prior file exists) | Previous action items graph — one rollback level |
| `aicrm_checkpoint.json` | Every run | Run state: last run timestamp, processed Message-IDs, `pii_token_map`, `pii_counters` |
| `aicrm_dashboard.html` | Every run | Self-contained static HTML dashboard — auto-generated, open in any browser, no server needed |

### ⛔ present_files — Explicit Prohibition

**Claude MUST NOT call `present_files` for any output file at any point during or after a pipeline run**, unless the user explicitly requests a specific file by name in their prompt (e.g. "show me the chunk graph file", "give me the action items JSON").

This prohibition applies to all pipeline output files without exception:
- `aicrm_chunk_graph.json`
- `aicrm_action_items_graph.json`
- `aicrm_action_items_graph.json.bak`
- `aicrm_checkpoint.json`
- `aicrm_dashboard.html`

The only file reference returned to the user after a run is the plain-text dashboard link printed at the end of Step 5. Do not supplement this with `present_files` cards for any file.

---

## Step 0 — Mailbox Connection via Cowork Connector

### Connector Discovery

The skill does not hardcode a specific email provider. At runtime, Claude discovers
what email connector the user has installed by listing available MCP tools and
matching against known email connector patterns.

```python
# Pseudo-code — Claude executes this reasoning, not literal Python
def discover_email_connector():
    """
    Inspect available MCP tools to find an email connector.
    Look for tools matching these capability patterns:
      - list/search/fetch emails (list_messages, search_emails, get_emails, fetch_inbox)
      - get a single email by ID (get_message, get_email, fetch_message)
    Common connector names: gmail, outlook, microsoft-graph, imap, nylas, emailconnector
    """
    available_tools = list_mcp_tools()  # Cowork introspection
    email_tools = [t for t in available_tools if any(
        kw in t['name'].lower()
        for kw in ['mail', 'email', 'inbox', 'message', 'gmail', 'outlook']
    )]
    if not email_tools:
        raise RuntimeError(
            "No email connector found. "
            "Please connect an email connector in Cowork Settings → Connectors."
        )
    return email_tools
```

If no connector is found, stop and instruct the user to connect one via
Cowork Settings → Connectors before proceeding.

### Abstract Connector Interface

Regardless of which connector is installed, the skill expects it to provide
two capabilities:

```
list_emails(folders, since_date=None) → list of email summaries
  Each summary must contain at minimum:
    message_id   : str   — RFC 2822 Message-ID header (unique per message)
    subject      : str
    date         : str   — ISO 8601 or RFC 2822 date string
    folder       : str   — "INBOX", "Sent", folder name

get_email(message_id) → full email object
  Must contain:
    message_id   : str
    subject      : str
    date         : str
    from_name    : str
    from_email   : str
    to           : list of {name, email}
    cc           : list of {name, email}
    body_text    : str   — plain text body (preferred)
    body_html    : str   — HTML body (fallback if no body_text)
    folder       : str
```

### Normalizing Connector Output

Different connectors use different field names. Normalize before processing:

```python
import re
from email import message_from_string  # stdlib — for raw MIME parsing if needed

def normalize_email(raw: dict) -> dict:
    """
    Map connector-specific field names to the standard schema.
    Handles Gmail, Outlook/Graph, Nylas, and generic IMAP connector output.
    """
    def pick(*keys):
        for k in keys:
            v = raw.get(k)
            if v:
                return v
        return None

    def parse_address(raw_addr):
        """Parse 'Name <email>' or plain email string into {name, email}."""
        if isinstance(raw_addr, dict):
            return raw_addr  # already normalized
        if isinstance(raw_addr, str):
            m = re.match(r'^(.*?)\s*<([^>]+)>', raw_addr)
            if m:
                return {'name': m.group(1).strip().strip('"'), 'email': m.group(2).strip().lower()}
            return {'name': raw_addr.split('@')[0].replace('.', ' ').title(),
                    'email': raw_addr.lower()}
        return {'name': '', 'email': ''}

    def parse_address_list(val):
        if val is None:
            return []
        if isinstance(val, list):
            return [parse_address(a) for a in val]
        if isinstance(val, str):
            return [parse_address(a.strip()) for a in re.split(r'[;,]', val) if a.strip()]
        return []

    body = pick('body_text', 'text', 'snippet', 'body')
    if not body:
        html = pick('body_html', 'html', 'bodyPreview')
        body = re.sub(r'<[^>]+>', ' ', html or '')  # strip HTML tags, stdlib only
    body = re.sub(r'\s+', ' ', body or '').strip()

    return {
        'message_id': pick('message_id', 'id', 'internetMessageId', 'uid'),
        'subject':    pick('subject') or '',
        'date':       pick('date', 'sentDateTime', 'receivedDateTime', 'internalDate') or '',
        'from':       parse_address(pick('from', 'sender', 'from_address') or ''),
        'to':         parse_address_list(pick('to', 'toRecipients', 'recipients')),
        'cc':         parse_address_list(pick('cc', 'ccRecipients')),
        'body':       body,
        'folder':     pick('folder', 'labelIds', 'parentFolderId') or 'INBOX',
    }
```

### Folders to Fetch

Always fetch from all available folders. Priority order:
1. INBOX
2. Sent / Sent Items
3. All other folders (Archive, Custom labels, etc.)

**NEVER fetch from junk or trash folders.** These folders contain messages the user
has explicitly discarded or flagged as unwanted and must always be excluded:

| Folder names to skip (case-insensitive) |
|-----------------------------------------|
| Junk, Junk Email, Junk Mail |
| Spam |
| Trash, Deleted Items, Deleted Messages |
| Bin |

```python
EXCLUDED_FOLDERS = {
    'junk', 'junk email', 'junk mail',
    'spam',
    'trash', 'deleted items', 'deleted messages',
    'bin',
}

def is_excluded_folder(folder_name: str) -> bool:
    return folder_name.strip().lower() in EXCLUDED_FOLDERS
```

Apply this filter at the point of listing folders AND when normalizing each email's
`folder` field — if a message's folder resolves to an excluded name, skip it even
if the connector returns it unexpectedly in a bulk fetch.

---

## Step 1.1 — Identity Resolution

The skill resolves the mailbox owner's identity before processing any emails.
This determines what is "internal" (the user's own organisation) vs. "external"
(clients, vendors, partners) — the classification that every company and person
node depends on.

**No user input is required.** The connector already knows who owns the mailbox.
The skill reads that identity and derives the internal domain automatically.

### Generic provider detection

If the owner's email domain is a well-known consumer or generic provider, the user
is a solo operator — a freelancer, consultant, or single-person company. Everyone
else in the mailbox is an external contact.

```python
GENERIC_EMAIL_PROVIDERS = {
    # Consumer
    'gmail.com', 'googlemail.com',
    'yahoo.com', 'yahoo.co.uk', 'yahoo.fr', 'yahoo.de',
    'hotmail.com', 'hotmail.co.uk', 'hotmail.fr',
    'outlook.com', 'live.com', 'msn.com',
    'icloud.com', 'me.com', 'mac.com',
    'aol.com', 'protonmail.com', 'proton.me',
    'zoho.com', 'fastmail.com', 'tutanota.com',
    # Country-specific generic
    'mail.ru', 'yandex.ru', 'gmx.com', 'gmx.de', 'web.de',
    't-online.de', 'orange.fr', 'wanadoo.fr', 'free.fr',
    'libero.it', 'virgilio.it', 'tin.it',
    'bigpond.com', 'optusnet.com.au',
}

def resolve_identity(connector) -> dict:
    """
    Retrieve mailbox owner identity from connector.
    Returns an identity config dict written to aicrm_identity.json
    and loaded on every subsequent run.

    company_mode:
      'team' — custom domain, multiple people share it → classify by domain
      'solo' — generic provider → user is their own company, all others external
    """
    profile = connector.get_profile()   # returns {email, name} at minimum
    owner_email = profile.get('email', '').lower().strip()
    owner_name  = profile.get('name', '') or owner_email.split('@')[0]
    domain      = owner_email.split('@')[-1] if '@' in owner_email else ''

    if domain in GENERIC_EMAIL_PROVIDERS:
        company_mode    = 'solo'
        internal_domains = set()           # no shared domain — only owner is internal
        company_name    = owner_name       # user IS their own company
    else:
        company_mode    = 'team'
        internal_domains = {domain}        # everyone @domain is internal
        company_name    = domain.split('.')[0].capitalize()  # "acme" from acme.com

    identity = {
        'owner_email':       owner_email,
        'owner_name':        owner_name,
        'owner_domain':      domain,
        'company_name':      company_name,
        'company_mode':      company_mode,      # 'team' | 'solo'
        'internal_domains':  list(internal_domains),
        'resolved_at':       datetime.now(timezone.utc).isoformat(),
    }
    return identity
```

### How identity is used downstream

**Person nodes** — `is_internal: True` if person's domain is in `internal_domains`
OR their email matches `owner_email` exactly (solo mode).

**Company nodes** — the owner's own company is created as the first company node
with `type: 'internal'`. In solo mode its name is the owner's name. In team mode
it's derived from the domain.

**`get_or_create_person()`** and **`get_or_create_company()`** both receive
`internal_domains` as a parameter (already shown in their signatures). In solo
mode, `internal_domains` is an empty set and only `owner_email` exact-match
triggers `is_internal: True`.

### Identity config persistence

```python
IDENTITY_PATH = 'aicrm_identity.json'

def load_or_resolve_identity(connector) -> dict:
    """
    Load identity from file if it exists (fast path).
    Resolve from connector if not (first run or file deleted).
    Identity is stable across runs — re-resolve only if owner_email changes.
    """
    if os.path.exists(IDENTITY_PATH):
        with open(IDENTITY_PATH) as f:
            identity = json.load(f)
        # Validate — if connector owner changed (e.g. shared machine), re-resolve
        current_profile = connector.get_profile()
        if identity.get('owner_email') == current_profile.get('email', '').lower():
            identity['internal_domains'] = set(identity['internal_domains'])
            return identity
    # Resolve fresh
    identity = resolve_identity(connector)
    with open(IDENTITY_PATH, 'w') as f:
        json.dump({**identity,
                   'internal_domains': list(identity['internal_domains'])}, f, indent=2)
    return identity
```
---

## Step 1.2 — Checkpoint System

### Purpose

The checkpoint file is the **anchor for incremental sync**. It records which
emails have already been processed so that each subsequent run fetches and processes
only genuinely new messages, regardless of connector-reported timestamps.

### Why Message-ID, Not Timestamp

Email timestamps are unreliable as sole checkpoints:
- Forwarded emails carry the original send date, not receipt date
- Some connectors return `receivedDateTime` in different timezones
- Backdated calendar invites can appear "older" than the last run

`Message-ID` (RFC 2822) is guaranteed unique per message and immutable.
It is the authoritative deduplication key.

### Checkpoint File Schema

```json
{
  "version": "2.0",
  "last_run_utc": "2026-02-24T22:00:00Z",
  "last_run_mode": "full | incremental",
  "total_emails_processed_all_time": 56,
  "emails_processed_this_run": 8,
  "processed_message_ids": [
    "<CABc123abc@mail.gmail.com>",
    "<BN1PR07MB4321...@namprd07.prod.outlook.com>"
  ],
  "threads_updated_this_run": ["thread_af75f00118", "thread_81c29afe0f"],
  "chunk_graph_path": "aicrm_chunk_graph.json",
  "action_graph_path": "aicrm_action_items_graph.json"
}
```

### Checkpoint Read / Write

```python
import json, os
from datetime import datetime, timezone

CHECKPOINT_PATH = "aicrm_checkpoint.json"

def read_checkpoint():
    if not os.path.exists(CHECKPOINT_PATH):
        return {
            "version": "2.0",
            "last_run_utc": None,
            "processed_message_ids": [],
            "total_emails_processed_all_time": 0,
        }
    with open(CHECKPOINT_PATH) as f:
        return json.load(f)

def write_checkpoint(checkpoint: dict, new_message_ids: list,
                     threads_updated: list, mode: str):
    checkpoint['last_run_utc'] = datetime.now(timezone.utc).isoformat()
    checkpoint['last_run_mode'] = mode
    checkpoint['emails_processed_this_run'] = len(new_message_ids)
    checkpoint['total_emails_processed_all_time'] = (
        checkpoint.get('total_emails_processed_all_time', 0) + len(new_message_ids)
    )
    # Use a set to deduplicate; convert back to list for JSON serialization
    existing = set(checkpoint.get('processed_message_ids', []))
    existing.update(new_message_ids)
    checkpoint['processed_message_ids'] = list(existing)
    checkpoint['threads_updated_this_run'] = threads_updated
    with open(CHECKPOINT_PATH, 'w') as f:
        json.dump(checkpoint, f, indent=2)
```

### Incremental Fetch Logic

```python
# ── First-run fetch limits ────────────────────────────────────────────────────
# Applied on first run only. Subsequent runs fetch all new emails since
# last checkpoint with no cap (incremental volumes are naturally bounded).
# Can be overridden by the user at invocation time.
FIRST_RUN_MAX_DAYS   = 90    # look back at most 90 days
FIRST_RUN_MAX_EMAILS = 1000  # process at most 1000 emails
# When both limits are set, whichever is hit first wins.
# Priority: sort all candidates by date descending, apply 90-day cutoff,
# then truncate to 1000. This guarantees the 1000 most recent emails
# within the window — not the oldest 1000.
# ─────────────────────────────────────────────────────────────────────────────

def fetch_first_run_emails(connector,
                            max_days:   int = FIRST_RUN_MAX_DAYS,
                            max_emails: int = FIRST_RUN_MAX_EMAILS) -> list:
    """
    Fetch emails for the first run, bounded by recency and count.
    Returns at most `max_emails` emails from the last `max_days` days,
    sorted newest-first so the most recent activity is always included.
    Always skips Junk, Trash, and Spam folders.

    Override defaults by passing max_days / max_emails explicitly, e.g.:
        fetch_first_run_emails(connector, max_days=180, max_emails=2000)
    """
    from datetime import datetime, timezone, timedelta

    cutoff_date = datetime.now(timezone.utc) - timedelta(days=max_days)

    # Step 1: list all summaries (metadata only — cheap)
    summaries = connector.list_emails(folders=['INBOX', 'Sent', 'All Mail'])

    # Step 2: exclude Junk/Trash/Spam
    summaries = [s for s in summaries
                 if not is_excluded_folder(s.get('folder', ''))]

    # Step 3: apply 90-day cutoff — keep only emails within the window
    def parse_date(s):
        try:
            dt = datetime.fromisoformat(s.get('date', '')[:19])
            return dt.replace(tzinfo=timezone.utc) if dt.tzinfo is None else dt
        except (ValueError, TypeError):
            return datetime.min.replace(tzinfo=timezone.utc)

    summaries = [s for s in summaries if parse_date(s) >= cutoff_date]

    # Step 4: sort newest-first, then truncate to max_emails
    # This ensures the 1000 most recent are chosen, not the oldest 1000
    summaries.sort(key=parse_date, reverse=True)
    summaries = summaries[:max_emails]

    # Step 5: fetch full email content
    emails = [normalize_email(connector.get_email(s['message_id']))
              for s in summaries]

    # Step 6: final safety pass on folder
    return [e for e in emails if not is_excluded_folder(e.get('folder', ''))]


def fetch_new_emails(connector, checkpoint: dict) -> list:
    """
    Fetch emails not yet processed (subsequent runs only).
    Uses Message-ID as authoritative dedup key.
    No count or date cap — incremental volumes are naturally bounded
    by the time elapsed since the last run.
    Always skips Junk, Trash, and Spam folders.
    """
    processed_ids = set(checkpoint.get('processed_message_ids', []))
    since_date    = checkpoint.get('last_run_utc')  # ISO string or None

    # Step 1: list summaries (cheap — metadata only)
    summaries = connector.list_emails(
        folders=['INBOX', 'Sent', 'All Mail'],
        since=since_date  # pre-filter; connector may or may not support this
    )

    # Step 2: exclude Junk/Trash/Spam
    summaries = [s for s in summaries
                 if not is_excluded_folder(s.get('folder', ''))]

    # Step 3: filter by Message-ID (authoritative dedup)
    new_summaries = [s for s in summaries
                     if s.get('message_id') not in processed_ids]

    # Step 4: fetch full email content
    emails = [normalize_email(connector.get_email(s['message_id']))
              for s in new_summaries]

    # Step 5: final safety pass on folder
    return [e for e in emails if not is_excluded_folder(e.get('folder', ''))]
```

---

## Module: PII Tokenization

This module runs **after email normalization and before Phase 1**. It replaces
real names, email addresses, and phone numbers with stable, reversible tokens
before any LLM call. Raw PII never appears in prompts.

The detokenization map is persisted in the checkpoint so tokens remain stable
across runs — the same person always receives the same token.

### What gets tokenized

| PII type | Token format | Example |
|----------|-------------|---------|
| Email address | `EMAIL_NNN` | `EMAIL_001` |
| Person name (from/to/cc) | `PERSON_NNN` | `PERSON_001` |
| Company domain | `DOMAIN_NNN` | `DOMAIN_001` |
| Phone number | `PHONE_NNN` | `PHONE_001` |

Company *names* are **not** tokenized — they are required for entity resolution
and relationship graph labelling. Only raw contact identifiers are replaced.

### Token map schema (stored in checkpoint)

```json
{
  "pii_token_map": {
    "EMAIL_001": "alice@example.com",
    "EMAIL_002": "bob@partner.org",
    "PERSON_001": "Alice Chen",
    "PERSON_002": "Bob Martinez",
    "DOMAIN_001": "example.com",
    "PHONE_001": "+1-415-555-0100"
  },
  "pii_counters": {
    "EMAIL": 2,
    "PERSON": 2,
    "DOMAIN": 1,
    "PHONE": 1
  }
}
```

### Tokenizer

```python
import re

# Phone: E.164, US domestic, international with spaces/dashes
_PHONE_RE = re.compile(
    r'(\+?1?\s*[-.]?\s*\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4})'
    r'|(\+\d{1,3}[\s.-]\d{2,4}[\s.-]\d{2,4}[\s.-]?\d{2,4})'
)

def _get_or_create_token(value: str, prefix: str,
                          token_map: dict, counters: dict) -> str:
    """
    Return existing token for value, or create a new one.
    Lookup is case-insensitive for emails; exact for names.
    """
    lookup = value.lower() if prefix in ('EMAIL', 'DOMAIN') else value
    for token, stored in token_map.items():
        if token.startswith(prefix + '_'):
            if (stored.lower() if prefix in ('EMAIL', 'DOMAIN') else stored) == lookup:
                return token
    counters[prefix] = counters.get(prefix, 0) + 1
    token = f"{prefix}_{counters[prefix]:03d}"
    token_map[token] = value
    return token

def tokenize_email(email: dict, token_map: dict, counters: dict) -> dict:
    """
    Replace PII in a normalized email dict with stable tokens.
    Operates on a shallow copy — does not mutate the original.
    Returns the tokenized email.
    """
    def tok_person(p: dict) -> dict:
        if not p:
            return p
        name_tok  = _get_or_create_token(p['name'],  'PERSON', token_map, counters) if p.get('name')  else ''
        email_tok = _get_or_create_token(p['email'], 'EMAIL',  token_map, counters) if p.get('email') else ''
        domain    = p['email'].split('@')[-1] if '@' in p.get('email', '') else ''
        if domain:
            _get_or_create_token(domain, 'DOMAIN', token_map, counters)
        return {'name': name_tok, 'email': email_tok}

    def tok_body(text: str) -> str:
        if not text:
            return text
        def replace_phone(m):
            return _get_or_create_token(m.group(0), 'PHONE', token_map, counters)
        return _PHONE_RE.sub(replace_phone, text)

    return {
        **email,
        'from':           tok_person(email.get('from', {})),
        'to':             [tok_person(p) for p in email.get('to', [])],
        'cc':             [tok_person(p) for p in email.get('cc', [])],
        'body':           tok_body(email.get('body', '')),
        'quoted_context': tok_body(email.get('quoted_context', '')),
    }

def detokenize(text: str, token_map: dict) -> str:
    """Restore real values from tokens in any output string."""
    for token, real_value in token_map.items():
        text = text.replace(token, real_value)
    return text
```

### Where tokenization runs in the pipeline

```python
# After normalize_email(), before email_to_chunk():
checkpoint   = read_checkpoint()
token_map    = checkpoint.get('pii_token_map', {})
counters     = checkpoint.get('pii_counters',  {})

tokenized_emails = [tokenize_email(e, token_map, counters) for e in normalized_emails]

# Persist updated map back to checkpoint after processing
checkpoint['pii_token_map'] = token_map
checkpoint['pii_counters']  = counters
write_checkpoint(checkpoint, ...)
```

### Where detokenization runs

Detokenization is applied **only at output time** — in dialog reports and
CRM query interface responses. The stored graph files always contain tokens,
never raw PII. This keeps all persisted data clean.

```python
# In the dialog report renderer:
report_text = build_report(action_graph, chunk_graph)
print(detokenize(report_text, token_map))

# In the CRM query interface response:
response = handle_query(query, action_graph, chunk_graph)
print(detokenize(response, token_map))
```

---

## Phase 1 — Chunk Graph

### Thread Deduplication (v2: Message-ID based)

In v1 (RTF files), threads were deduplicated by subject + file size.
In v2 (live mailbox), deduplicate by **In-Reply-To / References headers** first,
then fall back to subject normalization.

```python
def group_into_threads(emails: list) -> dict:
    """
    Group emails into threads using two strategies:
    1. Primary: In-Reply-To / References chain (RFC 2822 thread headers)
    2. Fallback: normalized subject matching

    Returns: dict of thread_key → list of emails (all messages in that thread)
    """
    threads = {}
    msg_id_to_thread = {}

    for email in emails:
        msg_id    = email.get('message_id', '')
        reply_to  = email.get('in_reply_to', '')
        refs      = email.get('references', [])

        # Try to find existing thread via References chain
        thread_key = None
        for ref in ([reply_to] + refs):
            if ref in msg_id_to_thread:
                thread_key = msg_id_to_thread[ref]
                break

        # Fallback: subject normalization
        if not thread_key:
            subj = email.get('subject', '')
            subj_clean = re.sub(r'^(re|fwd|fw|aw|sv)[\s:\|]+', '', subj, flags=re.I)
            subj_clean = re.sub(r'^(re|fwd|fw|aw|sv)[\s:\|]+', '', subj_clean, flags=re.I)
            subj_clean = re.sub(r'\s+', ' ', subj_clean).strip().lower()
            thread_key = subj_clean or msg_id

        threads.setdefault(thread_key, []).append(email)
        msg_id_to_thread[msg_id] = thread_key

    # Sort each thread chronologically (oldest first)
    for key in threads:
        threads[key].sort(key=lambda e: e.get('date', ''))

    return threads
```

### Chunk Node Creation (v2: from connector email, not RTF)

In v2, each email from the connector becomes one chunk directly. Before building
the chunk node, two pre-processing steps run: **quote stripping** and **email type
classification**. Both run on the raw body before any chunking logic.

#### Quote Stripping

Most email bodies contain quoted reply text from earlier messages. That content is
already captured in earlier chunks, so including it again inflates body size and
causes Phase 2 to see the same sentences multiple times, sometimes attributing
action items to the wrong person. Strip quoted blocks and store them separately.

```python
import re

# Patterns that mark the start of a quoted reply block
_QUOTE_PATTERNS = [
    re.compile(r'^>+\s?', re.MULTILINE),                          # > quoted line
    re.compile(r'^On .{10,80}wrote:\s*$', re.MULTILINE),          # On [date], [person] wrote:
    re.compile(r'^-{3,}\s*Original Message\s*-{3,}', re.MULTILINE | re.IGNORECASE),
    re.compile(r'^From:\s+\S', re.MULTILINE),                     # From: ... block header
    re.compile(r'^Sent:\s+\S', re.MULTILINE),                     # Sent: ... block header
]

def strip_quoted_reply(body: str) -> tuple[str, str]:
    """
    Separate new content from quoted reply context.

    Returns:
        (body_clean, quoted_context)
        body_clean     — only the new content written by this sender
        quoted_context — the quoted portion (for context, not primary analysis)
    """
    if not body:
        return '', ''

    lines = body.splitlines()
    split_at = len(lines)  # default: no quote found

    for i, line in enumerate(lines):
        # Check if this line starts a quoted block
        for pattern in _QUOTE_PATTERNS:
            if pattern.match(line):
                # Confirm at least 2 of the next 5 lines also look quoted
                surrounding = lines[i:i+5]
                quoted_count = sum(
                    1 for l in surrounding
                    if any(p.match(l) for p in _QUOTE_PATTERNS) or l.startswith('>')
                )
                if quoted_count >= 2 or line.startswith('>'):
                    split_at = i
                    break
        if split_at < len(lines):
            break

    body_clean     = '\n'.join(lines[:split_at]).strip()
    quoted_context = '\n'.join(lines[split_at:]).strip()
    return body_clean, quoted_context
```

#### Email Type Classification

Not all emails deserve the same treatment in Phase 2. Classify each email's
`content_type` before chunking so Phase 2 can apply the right extraction strategy:

```python
def classify_email_type(email: dict) -> str:
    """
    Classify the email content type. Used by Phase 2 to decide how to process.

    Returns one of:
      DIRECT_MESSAGE   — a genuine human-to-human conversation message
      FORWARD          — a forwarded message (may contain nested attribution chains)
      CALENDAR_INVITE  — a meeting invite or calendar event notification
      SYSTEM_NOTIFICATION — automated CRM, ticketing, or workflow notification
      AUTO_REPLY       — out-of-office or automatic acknowledgment
      MARKETING        — newsletter, promotional, or mass email

    Phase 2 behaviour by type:
      DIRECT_MESSAGE   → full action item + event extraction
      FORWARD          → extract from body_clean; quoted_context may contain older context
      CALENDAR_INVITE  → create MEETING action item from structured fields; skip body NLP
      SYSTEM_NOTIFICATION → extract SYSTEM_TRIGGER action items only
      AUTO_REPLY       → SKIP entirely (no action items possible)
      MARKETING        → SKIP entirely (no action items possible)
    """
    subject = (email.get('subject') or '').lower()
    body    = (email.get('body') or '').lower()
    sender  = (email.get('from', {}).get('email') or '').lower()
    to_list = email.get('to', [])

    # Auto-reply detection (highest priority — skip these entirely)
    auto_reply_signals = [
        'out of office', 'automatic reply', 'auto-reply', 'autoreply',
        'i am away', 'i am out', 'i\'m out of the office', 'on leave until',
        'will be back', 'i\'m currently out', 'auto response',
    ]
    if any(sig in subject or sig in body[:300] for sig in auto_reply_signals):
        return 'AUTO_REPLY'

    # Marketing / newsletter detection
    marketing_signals = [
        'unsubscribe', 'view in browser', 'email preferences', 'mailing list',
        'newsletter', 'promotional', 'no-reply@', 'noreply@', 'donotreply@',
    ]
    if (any(sig in body for sig in marketing_signals) or
            any(sig in sender for sig in ['no-reply', 'noreply', 'donotreply'])):
        return 'MARKETING'

    # Calendar / meeting invite detection
    calendar_signals = [
        'has invited you', 'you\'ve been invited', 'calendar invite',
        'when:', 'where:', 'join microsoft teams meeting',
        'join zoom meeting', 'google meet', 'ics', '.ical',
        'accepted:', 'declined:', 'tentative:', 'canceled:',
    ]
    if any(sig in subject or sig in body[:500] for sig in calendar_signals):
        return 'CALENDAR_INVITE'

    # System / automated notification detection
    system_signals = [
        'jira', 'confluence', 'servicenow', 'salesforce', 'hubspot',
        'github', 'gitlab', 'bitbucket', 'jenkins', 'automated notification',
        'ticket created', 'ticket updated', 'issue assigned', 'build failed',
        'deployment', 'alert:', 'notification:', 'do not reply',
    ]
    if any(sig in subject or sig in body[:300] for sig in system_signals):
        return 'SYSTEM_NOTIFICATION'

    # Forward detection
    forward_signals = ['fwd:', 'fw:', 'forwarded message', '---------- forwarded message']
    if any(sig in subject or sig in body[:200] for sig in forward_signals):
        return 'FORWARD'

    # Default: treat as a genuine direct message
    return 'DIRECT_MESSAGE'
```

#### Building the Chunk Node

```python
import hashlib, re

def stable_id(node_type, *parts):
    key = node_type + "::" + "::".join(str(p).lower().strip() for p in parts)
    return node_type + "_" + hashlib.md5(key.encode()).hexdigest()[:10]

def email_to_chunk(email: dict, thread_id: str, position: int) -> dict:
    """Convert a normalized connector email into a chunk node."""
    sender       = email.get('from', {'name': '', 'email': ''})
    raw_body     = email.get('body', '')
    content_type = classify_email_type(email)

    # Quote stripping: separate new content from quoted reply context
    body_clean, quoted_context = strip_quoted_reply(raw_body)

    return {
        'id':              stable_id('chunk', thread_id, str(position),
                                     sender.get('email', '')),
        'message_id':      email.get('message_id', ''),
        'thread_id':       thread_id,
        'thread_subject':  email.get('subject', ''),
        'position':        position,                        # 0 = oldest
        'position_desc':   'oldest' if position == 0 else 'newest',
        'sender':          sender,
        'recipients':      email.get('to', []),
        'cc':              email.get('cc', []),
        'date':            email.get('date', ''),
        'subject':         email.get('subject', ''),
        'content_type':    content_type,                    # classification
        'body':            body_clean[:2000],               # new content only
        'body_len':        len(body_clean),
        'quoted_context':  quoted_context[:1000],           # prior chain (capped)
        'folder':          email.get('folder', 'INBOX'),
    }
```

### Person and Company Resolution Helpers

These helpers are called during chunk graph building for every sender and recipient.
They handle the `email_evidence` transition — when a new email chunk is linked
to a manually-created entity, the entity is promoted from `email_evidence: false`
to `email_evidence: true`.

```python
def get_or_create_person(address: dict,
                          chunk_graph: dict,
                          internal_domains: set) -> dict:
    """
    Find or create a person node for an email address.
    If a manually-created person exists with matching email, promote email_evidence.
    Returns the person node (new or existing).
    """
    email_addr = address.get('email', '').lower().strip()
    name       = address.get('name', '') or email_addr

    persons = chunk_graph['nodes'].setdefault('persons', [])
    persons_by_email = {p['email'].lower(): p for p in persons}

    if email_addr in persons_by_email:
        person = persons_by_email[email_addr]
        # Promote email_evidence if this was a manually-created person
        if not person.get('email_evidence', True):
            person['email_evidence'] = True
        return person

    # Not found — create new
    domain     = email_addr.split('@')[-1] if '@' in email_addr else ''
    is_internal = domain in internal_domains
    person = {
        'id':            stable_id('person', email_addr),
        'name':          name,
        'email':         email_addr,
        'domain':        domain,
        'is_internal':   is_internal,
        'company_id':    None,   # resolved via WORKS_AT edge once company is known
        'company_name':  None,
        'appears_in_threads': [],
        'email_evidence': True,  # created from real email — always true
    }
    persons.append(person)
    return person


def get_or_create_company(domain: str,
                           name: str,
                           chunk_graph: dict,
                           internal_domains: set) -> dict:
    """
    Find or create a company node for an email domain.
    If a manually-created company exists with matching domain, promote email_evidence.
    Returns the company node (new or existing).
    """
    domain_clean = domain.lower().strip()
    companies = chunk_graph['nodes'].setdefault('companies', [])
    companies_by_domain = {c.get('domain', '').lower(): c for c in companies
                           if c.get('domain')}

    if domain_clean in companies_by_domain:
        company = companies_by_domain[domain_clean]
        # Promote email_evidence if this was a manually-created company
        if not company.get('email_evidence', True):
            company['email_evidence'] = True
        return company

    # Not found — create new (from email, so email_evidence = True)
    is_internal = domain_clean in internal_domains
    company = {
        'id':              stable_id('company', domain_clean),
        'name':            name or domain_clean,
        'domain':          domain_clean,
        'is_internal':     is_internal,
        'type':            'internal' if is_internal else 'external_unknown',
        'type_source':     'pipeline',
        'type_confidence': 1.0 if is_internal else None,  # internal is certain; external needs LLM
        'email_evidence':  True,
    }
    companies.append(company)
    return company
```

**`email_evidence` promotion rule:** When `get_or_create_person()` or
`get_or_create_company()` finds a node with `email_evidence: false`, it sets
it to `true` in-place before returning. The next `safe_overwrite()` call
persists the change. This is the only code path that transitions
`email_evidence` from `false` to `true` — never set it manually.

**Position convention (v2):** Unlike v1 where `position=0` was the newest message
(RTF files list newest first), in v2 emails are sorted chronologically before
building chunks, so `position=0` is the **oldest** message. This is more natural
for reading and eliminates the need to reverse-sort during Phase 2 traversal.

### Chunk Graph Node and Edge Schemas

These are unchanged from v1. See the full schemas below.

#### thread node
```json
{
  "id": "thread_af75f00118",
  "thread_key": "lunch with xyz",
  "subject": "Re: Lunch with XYZ",
  "chunk_ids": ["chunk_2978ac80da", "chunk_8a2806310a"],
  "message_ids": ["<msg1@domain>", "<msg2@domain>"],
  "first_date": "2025-10-19",
  "last_date": "2026-02-23"
}
```

#### chunk node
```json
{
  "id": "chunk_2978ac80da",
  "message_id": "<CABc123@mail.gmail.com>",
  "thread_id": "thread_af75f00118",
  "thread_subject": "Re: Lunch with XYZ",
  "position": 11,
  "position_desc": "newest",
  "sender": {"name": "Emily Blunt", "email": "emily.blunt@xyz.net"},
  "recipients": [{"name": "Joe", "email": "joe.lucky@legal.com"}],
  "cc": [{"name": "James Smith", "email": "james.smith@xyz.net"}],
  "date": "2026-02-23T10:00:00Z",
  "subject": "Re: Lunch with XYZ",
  "content_type": "DIRECT_MESSAGE",
  "body": "Hi Joe, Completely understand — happy to reschedule for next week...",
  "body_len": 89,
  "quoted_context": "On Feb 22, Joe Lucky wrote: Can we move our call?...",
  "folder": "Sent"
}
```

`content_type` drives Phase 2 processing:
- `AUTO_REPLY` / `MARKETING` → chunk is stored but **Phase 2 skips it entirely**
- `CALENDAR_INVITE` → Phase 2 creates a MEETING action item from structured fields; skips body NLP
- `SYSTEM_NOTIFICATION` → Phase 2 extracts SYSTEM_TRIGGER items only
- `FORWARD` / `DIRECT_MESSAGE` → full extraction against `body` (not `quoted_context`)

#### person node
```json
{
  "id": "person_a3f9c21d7e",
  "name": "Emily Blunt",
  "email": "emily.blunt@xyz.net",
  "company_id": "company_b71d3a0f12",
  "company_name": "XYZ",
  "appears_in_threads": ["thread_af75f00118"]
}
```

#### company node
```json
{
  "id": "company_b71d3a0f12",
  "name": "XYZ",
  "domain": "xyz.net"
}
```

### Chunk Graph Edge Types

| Relation | From → To | When created |
|----------|-----------|--------------|
| `SAME_THREAD` | chunk → thread | Every chunk in a thread |
| `REPLIES_TO` | chunk[pos N] → chunk[pos N-1] | Sequential messages (chronological) |
| `SENT_BY` | chunk → person | `sender.email` |
| `RECEIVED_BY` | chunk → person | Each address in `to` + `cc` |
| `WORKS_AT` | person → company | Derived from email domain |
| `CROSS_REF` | thread → thread | Two threads share a person |

### Chunk Graph Top-Level JSON Structure

```json
{
  "meta": {
    "version": "2.0",
    "phase": "chunk_graph",
    "last_updated": "2026-02-24T22:00:00Z",
    "thread_count": 18,
    "chunk_count": 97,
    "person_count": 36,
    "company_count": 11,
    "edge_count": 694
  },
  "nodes": {
    "threads":   [ { ...thread node... } ],
    "chunks":    [ { ...chunk node... } ],
    "persons":   [ { ...person node... } ],
    "companies": [ { ...company node... } ]
  },
  "edges": [
    { "from": "chunk_id", "relation": "SAME_THREAD|REPLIES_TO|SENT_BY|RECEIVED_BY|WORKS_AT|CROSS_REF", "to": "...", "meta": {} }
  ]
}
```

### Stable ID Generation

```python
import hashlib

def stable_id(node_type: str, *parts: str) -> str:
    """
    Deterministic ID — same inputs always produce the same ID.
    Ensures the same person/company deduplicates across threads and across runs.
    """
    key = node_type + "::" + "::".join(str(p).lower().strip() for p in parts)
    return node_type + "_" + hashlib.md5(key.encode()).hexdigest()[:10]
```

### Module: Person Alias Resolution

This module runs **after all person nodes are created**, merging nodes that
refer to the same individual. It is fully deterministic — no LLM required.

The core insight: email local parts carry strong identity signal. `john`,
`john.doe`, `j.doe`, and `jdoe` are all plausibly the same person at the
same domain. Combined with name matching from signatures, this resolves
the majority of alias cases without ML.

#### Resolution rules (applied in order)

**Rule 1 — Exact email match (already handled by stable_id)**
Same email → same node. No additional logic needed.

**Rule 2 — Same domain + name token overlap**
Two persons share a domain AND their name tokens overlap by ≥ 1 token
(case-insensitive). The local parts of their email addresses must also share
≥ 1 token when split on `.`, `-`, `_`.

```python
def _name_tokens(name: str) -> set:
    """Split name into lowercase tokens, drop initials (single chars)."""
    return {t.lower() for t in name.replace('.', ' ').split() if len(t) > 1}

def _local_tokens(email: str) -> set:
    """Split email local part into tokens."""
    local = email.split('@')[0]
    return {t.lower() for t in re.split(r'[._\-]', local) if len(t) > 1}

def _same_domain(email_a: str, email_b: str) -> bool:
    return email_a.split('@')[-1].lower() == email_b.split('@')[-1].lower()

def _are_aliases(p1: dict, p2: dict) -> bool:
    """
    Return True if two person nodes are likely the same individual.
    Both persons must have non-empty names and emails.
    """
    if not _same_domain(p1['email'], p2['email']):
        return False
    name_overlap  = _name_tokens(p1['name']) & _name_tokens(p2['name'])
    local_overlap = _local_tokens(p1['email']) & _local_tokens(p2['email'])
    return len(name_overlap) >= 1 and len(local_overlap) >= 1
```

**Rule 3 — Same domain + identical local part, different display name**
`john.doe@co.com` with `"John"` vs `"John Doe"` → merge, keep the longer name.

#### Merge strategy

When two nodes are identified as aliases:
- **Keep**: the node with the most complete name (longer wins)
- **Redirect**: all edges pointing to the absorbed node are rewritten to the canonical node
- **Record**: `aliases` array on the canonical node stores all merged emails

```python
def resolve_person_aliases(persons: list, edges: list) -> tuple[list, list]:
    """
    Merge alias person nodes. Returns (deduplicated persons, updated edges).
    Runs after all person nodes are built for the current batch.
    """
    parent = {p['id']: p['id'] for p in persons}

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(a, b):
        ra, rb = find(a), find(b)
        if ra != rb:
            parent[rb] = ra  # merge b into a

    persons_by_id = {p['id']: p for p in persons}

    # Compare all pairs within same domain
    domain_groups: dict[str, list] = {}
    for p in persons:
        domain = p['email'].split('@')[-1].lower()
        domain_groups.setdefault(domain, []).append(p)

    for domain, group in domain_groups.items():
        for i, p1 in enumerate(group):
            for p2 in group[i+1:]:
                if _are_aliases(p1, p2):
                    union(p1['id'], p2['id'])

    # Build canonical nodes
    canonical: dict[str, dict] = {}
    for p in persons:
        root = find(p['id'])
        if root not in canonical:
            canonical[root] = dict(p)
            canonical[root].setdefault('aliases', [])
        else:
            existing = canonical[root]
            if len(p.get('name', '')) > len(existing.get('name', '')):
                existing['name'] = p['name']
            if p['email'] not in existing['aliases']:
                existing['aliases'].append(p['email'])

    # Rewrite edges: replace absorbed node IDs with canonical ID
    id_remap = {p['id']: find(p['id']) for p in persons}
    updated_edges = []
    seen_edges = set()
    for e in edges:
        new_from = id_remap.get(e['from'], e['from'])
        new_to   = id_remap.get(e['to'],   e['to'])
        key = (new_from, e['relation'], new_to)
        if key not in seen_edges:
            seen_edges.add(key)
            updated_edges.append({**e, 'from': new_from, 'to': new_to})

    return list(canonical.values()), updated_edges
```

#### Where it runs

```python
# End of Phase 1, after all person nodes are created:
graph['nodes']['persons'], graph['edges'] = resolve_person_aliases(
    graph['nodes']['persons'], graph['edges']
)
```

Also runs at the end of `extend_chunk_graph()` on incremental runs, since
new emails may introduce aliases of existing persons.

---

### Incremental Extension

On each incremental run, extend the existing chunk graph with new chunks only.
Do not rebuild from scratch.

```python
def extend_chunk_graph(existing_graph: dict, new_emails: list) -> dict:
    """
    Append new chunks to existing graph.
    - New messages in existing threads → append chunks, increment positions
    - New threads → create new thread node
    - New persons/companies → add nodes
    - Update CROSS_REF edges for newly connected threads
    Returns updated graph.
    """
    # Index existing data for fast lookup
    threads_by_key  = {t['thread_key']: t for t in existing_graph['nodes']['threads']}
    chunks_by_msgid = {c['message_id']: c for c in existing_graph['nodes']['chunks']}
    persons_by_email = {p['email'].lower(): p for p in existing_graph['nodes']['persons']}
    companies_by_name = {c['name'].lower(): c for c in existing_graph['nodes']['companies']}
    existing_edges = set(
        (e['from'], e['relation'], e['to']) for e in existing_graph['edges']
    )

    # Group new emails into threads
    new_threads = group_into_threads(new_emails)
    threads_updated = []

    for thread_key, emails in new_threads.items():
        # Skip emails already in graph (Message-ID dedup)
        truly_new = [e for e in emails
                     if e.get('message_id') not in chunks_by_msgid]
        if not truly_new:
            continue

        # Get or create thread node
        if thread_key in threads_by_key:
            thread = threads_by_key[thread_key]
        else:
            thread_id = stable_id('thread', thread_key)
            thread = {
                'id': thread_id,
                'thread_key': thread_key,
                'subject': truly_new[0].get('subject', ''),
                'chunk_ids': [],
                'message_ids': [],
                'first_date': truly_new[0].get('date', ''),
                'last_date': truly_new[-1].get('date', ''),
            }
            existing_graph['nodes']['threads'].append(thread)
            threads_by_key[thread_key] = thread

        # Determine starting position for new chunks
        start_pos = len(thread['chunk_ids'])

        for i, email in enumerate(truly_new):
            pos = start_pos + i
            chunk = email_to_chunk(email, thread['id'], pos)
            existing_graph['nodes']['chunks'].append(chunk)
            thread['chunk_ids'].append(chunk['id'])
            thread['message_ids'].append(email.get('message_id', ''))
            thread['last_date'] = email.get('date', thread.get('last_date', ''))
            chunks_by_msgid[email.get('message_id', '')] = chunk

            # Edges: SAME_THREAD
            e = (chunk['id'], 'SAME_THREAD', thread['id'])
            if e not in existing_edges:
                existing_graph['edges'].append({'from': e[0], 'relation': e[1], 'to': e[2], 'meta': {}})
                existing_edges.add(e)

            # Edges: REPLIES_TO
            if pos > 0:
                prev_chunk_id = thread['chunk_ids'][pos - 1]
                e = (chunk['id'], 'REPLIES_TO', prev_chunk_id)
                if e not in existing_edges:
                    existing_graph['edges'].append({'from': e[0], 'relation': e[1], 'to': e[2], 'meta': {}})
                    existing_edges.add(e)

            # Edges: SENT_BY, RECEIVED_BY, WORKS_AT (via get_or_create_person)
            # [same logic as full build — omitted for brevity]

        threads_updated.append(thread['id'])

    # Rebuild CROSS_REF edges for updated persons
    # [same logic as full build]

    # Update meta
    existing_graph['meta']['last_updated'] = datetime.now(timezone.utc).isoformat()
    existing_graph['meta']['thread_count']  = len(existing_graph['nodes']['threads'])
    existing_graph['meta']['chunk_count']   = len(existing_graph['nodes']['chunks'])
    existing_graph['meta']['person_count']  = len(existing_graph['nodes']['persons'])
    existing_graph['meta']['company_count'] = len(existing_graph['nodes']['companies'])
    existing_graph['meta']['edge_count']    = len(existing_graph['edges'])

    return existing_graph, threads_updated
```

---

## Phase 2 — Action Items Graph

### Required Sections — All Four Are Mandatory

The action items graph MUST always contain all four top-level sections.
Omitting any one makes the graph incomplete for CRM use:

| Section | Purpose |
|---------|---------|
| `persons` | Enriched person registry: roles extracted from email signatures, company types |
| `companies` | Company registry: domain + relationship type (internal/client/vendor/partner/external_unknown) |
| `events` | State-change moments: meetings cancelled, NDAs signed, deals won, speakers confirmed |
| `action_items` | Semantic action items with chunk provenance, owner IDs, dependency edges |

### Persons Section

Roles are extracted from **email signature lines** in chunk bodies — not inferred.
The person node is also the primary contact record, carrying CRM-grade contact
fields derived from email behaviour.

```json
{
  "id": "person_a3f9c21d7e",
  "name": "Emily Blunt",
  "email": "Emily.Blunt@xyz.net",
  "role": "Account Manager",
  "company": "XYZ",
  "company_id": "company_b71d3a0f12",
  "company_type": "client",

  "is_primary_contact": true,

  "seniority_signal": "decision_maker",

  "last_contact_date": "2026-02-24",
  "first_contact_date": "2025-10-19",
  "contact_count_30d": 8,
  "contact_count_90d": 31,

  "sent_count": 14,
  "received_count": 17,
  "response_rate": 0.82,

  "open_action_items_owned": ["action_7f3a1d2b9c"],

  "appears_in_threads": ["thread_af75f00118", "thread_81c29afe0f"],

  "deal_roles": [
    {"deal_id": "deal_b71d3a0f12", "role": "champion", "source": "manual", "date": "2026-02-20T09:00:00Z"}
  ],

  "notes": []
}
```

**Field notes:**
- `is_primary_contact`: `true` if this person has the highest email count with the internal team within their company. One primary contact per external company.
- `seniority_signal`: inferred from CC patterns — `"decision_maker"` (often the sole recipient, rarely CC'd), `"influencer"` (frequently CC'd on substantive threads), `"executor"` (primarily on operational threads), `"unknown"`. Never inferred from job title alone.
- `response_rate`: fraction of received emails where a reply from this person appears in a later chunk of the same thread.
- `sent_count` / `received_count`: total emails sent to vs. received from internal team across all threads.
- `contact_count_30d` / `contact_count_90d`: emails exchanged (both directions) in rolling windows.
- Role extraction rules: scan chunk body for signature lines after message body. Use most recent occurrence if role appears multiple times. Set `"role": null` if not found — do not infer.
- `deal_roles`: array of deal-scoped role assignments for this person. One entry per deal they appear in. Roles: `"champion"` | `"economic_buyer"` | `"technical_buyer"` | `"blocker"` | `"influencer"` | `"user"` | `"unknown"`. Initially `"unknown"` — updated by user via manual command or inferred by pipeline with `source: "pipeline"`. Kept in sync with `deal.contact_roles` — both are updated together. A person may have different roles on different deals.

### Companies Section

The company node is the primary account record. It must carry enough state to
replace a CRM account record — not just support graph traversal.

```json
{
  "id": "company_b71d3a0f12",
  "name": "Acme Corp",
  "domain": "acme.com",
  "type": "client",
  "type_source": "pipeline",       // "pipeline" | "manual" — manual_override takes precedence
  "type_confidence": 0.91,         // LLM confidence in this classification; null if manual

  "account_owner_id": {"value": "person_internal_abc123", "source": "manual", "date": "2026-03-06T10:00:00Z", "manual_override": true},
  "account_owner":    {"value": "James Smith",            "source": "manual", "date": "2026-03-06T10:00:00Z", "manual_override": true},

  "tier":     {"value": "A",    "source": "manual", "date": "2026-03-06T10:00:00Z", "manual_override": true},

  "industry": {"value": "Tech", "source": "manual", "date": "2026-03-06T10:00:00Z", "manual_override": true},

  "first_contact_date": "2025-10-01",
  "last_activity_date": "2026-02-24",
  "days_since_last_contact": 10,

  "contact_frequency_30d": 12,
  "contact_frequency_prev_30d": 8,
  "frequency_trend": "increasing",

  "open_action_items": 3,
  "overdue_action_items": 1,

  "deal_stage_id": "stage_03",
  "deal_stage_name": "Proposal",
  "days_in_current_stage": 9,

  "health_score": 74,
  "health_score_updated": "2026-02-24T22:00:00Z",

  "threads": ["thread_af75f00118", "thread_81c29afe0f"],
  "primary_contacts": ["person_a3f9c21d7e"],

  "notes": [
    {
      "date": "2026-02-10",
      "author_id": "person_internal_abc123",
      "text": "CEO mentioned budget freeze verbally at conference",
      "source": "manual"
    }
  ]
}
```

**Field notes:**
- `tier`: `"A"` | `"B"` | `"C"` | `null` — user-assigned priority. Not inferred. Set via manual annotation or the query interface.
- `industry`: not inferred from email alone. Set via manual annotation.
- `account_owner_id`: the internal person most frequently appearing in threads with this company. Computed as the internal sender with the highest email count to this company's domain.
- `contact_frequency_30d` / `contact_frequency_prev_30d`: count of emails exchanged in the current vs. previous 30-day window. Used to compute `frequency_trend`.
- `frequency_trend`: `"increasing"` | `"stable"` | `"declining"` | `"silent"` (no contact in 30+ days).
- `health_score`: 0–100 computed score. See Relationship Health Scoring module.
- `notes`: manual annotations with `source: manual`. Always appended, never overwritten. Phase 2 LLM-sourced notes use `source: email_chunk` and include `chunk_id`.
- `type` values: `internal` | `client` | `vendor` | `partner` | `external_unknown`
  - `internal` — the owner's own company. Set by identity resolution, never by LLM.
  - `client` — you sell to them. Set by LLM when commercial direction is clear.
  - `vendor` — they sell to you. Set by LLM when commercial direction is clear.
  - `partner` — mutual referral or co-selling relationship. Set by LLM when explicit partner language present.
  - `external_unknown` — **default for all new companies**. LLM could not determine type with confidence. User must classify.
  - `vendor_partner` removed — was ambiguous. Replaced by explicit `vendor` or `partner`.
- `type_source`: `"pipeline"` (LLM-classified) | `"manual"` (user-set via command). Manual always wins on subsequent runs — `should_pipeline_update()` guards this field.
- `type_confidence`: LLM's confidence in the classification (0.0–1.0). `null` when `type_source` is `"manual"`. Companies with `type_confidence < 0.7` are treated as `external_unknown` in reports regardless of assigned type.

### Events Section

Events are state-change moments (past) that are distinct from action items (future).

```json
{
  "id": "event_5f3a1d2b9c",
  "type": "MEETING_CANCELLED",
  "actor": "Gene Dou",
  "actor_id": "person_f8c3a21b44",
  "description": "PoC sync meeting cancelled by Gene Dou due to urgent matter",
  "date": "2026-02-24",
  "thread_id": "thread_5d8128732b",
  "evidence_chunk_id": "chunk_1f4d6a0e5a",
  "evidence": "I will not be able to attend the meeting scheduled today due to an urgent matter",
  "triggers_action": "action_7a3b2c1d9e"
}
```

**Event type taxonomy:**

Events are immutable facts with a date and evidence chunk. They record what happened,
not what needs to happen (that is an action item). Once written, an event node is never
updated — only action items change status.

*Social / relationship layer:*

| Type | Signal |
|------|--------|
| `MEETING_CANCELLED` | Calendar cancel email, "I cannot attend", "Canceled:" subject prefix |
| `MEETING_RESCHEDULED` | "Can we move to…", "reschedule", old time cancelled + new proposed |
| `MEETING_HELD` | Post-meeting follow-up, "As discussed on our call", "following up from today's meeting" |
| `NDA_SIGNED` | "Please find signed NDA", "executed MNDA in attach", "signed" doc attached |
| `SPEAKER_CONFIRMED` | "happy to take part", "confirmed for the event" |
| `INTRO_MADE` | "I'd like to introduce you to…", "connecting you with…" |
| `TICKET_OPENED` | JIRA notification, "ticket created", "TSCM-" reference |
| `ESCALATION` | "escalating to", "looping in leadership", "flagging this urgently" |
| `REFERRAL_MADE` | "I'd like to refer you to…", "you should speak with", warm intro made |

*Commercial layer:*

| Type | Signal |
|------|--------|
| `PROPOSAL_SENT` | "please find our proposal", "attached our commercial proposal", "sent quote" |
| `PROPOSAL_RECEIVED` | Incoming proposal from vendor/partner attached or described |
| `CONTRACT_SENT` | "please find the agreement", "MSA/SOW attached for review" |
| `CONTRACT_SIGNED` | "executed copy attached", "countersigned", "fully signed agreement" |
| `INVOICE_SENT` | "please find invoice", invoice number referenced, payment terms mentioned |
| `PAYMENT_RECEIVED` | "payment received", "funds cleared", "thank you for the payment" |
| `DEAL_WON` | CRM "Won opportunity" notification, "Congrats on closing" |
| `DEAL_LOST` | CRM lost notification, "going a different direction", "not moving forward" |
| `COMPETITOR_MENTIONED` | Competitor name appears in body — flag for intel (do not create action item automatically) |

> **Note on `MEETING_INVITED`**: this was in v1 but sits uncomfortably between event
> and action item — an invite is also a pending MEETING action item. In v2, when a
> calendar invite is detected via `content_type: CALENDAR_INVITE`, create a MEETING
> action item (status SCHEDULED) rather than a MEETING_INVITED event.

### Traversal Logic

For each thread in the Chunk Graph:
1. Retrieve all chunk nodes sorted by **`position` ascending** (0 = oldest first
   in v2 — read in chronological order directly, no reverse sort needed).
2. Read chunk bodies oldest → newest to follow conversation flow.
3. Apply action item detection taxonomy.
4. Link each item to the exact `chunk_id`(s) that evidence it.
5. Record BLOCKS dependencies between items.

### Action Item Detection Taxonomy

Action items represent things that **need to happen** — they have an owner, a status,
and can be resolved. They are strictly forward-looking; past facts are events.

| Type | Definition | Example | `response_type` |
|------|------------|---------|-----------------|
| `COMMITMENT` | Person explicitly stated they will do something | "I will get back to you with CVs" | — |
| `REQUEST` | One party asked another to act or provide information; no confirmation in later chunks. Sub-typed by `response_type`: `action` when the ask is for someone to *do* something ("Can you grant Git access?"); `information` when the ask is for a direct answer with no reply found ("Do you know if anyone else from Legal will be in NYC?") | see examples | `action` \| `information` |
| `ASSIGNMENT` | Explicit delegation to named person | "Adding @John who will lead this" | — |
| `DECISION_NEEDED` | A choice must be made by a named person or group before progress can continue | "Let me know which commercials work for you" | — |
| `APPROVAL_NEEDED` | Requires formal sign-off from a named authority | "Needs legal to sign off before we proceed" | — |
| `MEETING` | Meeting confirmed with time + participants | "7AM PST Thursday — confirmed" | — |
| `FOLLOW_UP` | Time-bound check-in not yet actioned; owner must re-engage | "Call with speaker after OOO return" | — |
| `INTRODUCTION` | Person A should be connected to Person B; intro not yet made | "I'll connect you with our Head of Partnerships" | — |
| `SYSTEM_TRIGGER` | Automated workflow or CRM event requiring a human action | "ASDLC flow triggered for support project creation" | — |

**Type removal rationale (v1 → v2):**
- `RESCHEDULING` removed — it is a status on a MEETING item, not a separate type. When a meeting is cancelled without a new time, the MEETING item status becomes `OPEN` and a `MEETING_CANCELLED` event is created.
- `UNANSWERED_QUESTION` removed — it is now `REQUEST` with `response_type: information`.
- `COMMITMENT_OVERDUE` removed as a type — it is now a **status** applied systematically by the asymmetric commitment detector (see dedicated section below), not detected ad hoc by the LLM during Phase 2 extraction.

### Action Item Node Schema

```json
{
  "id": "action_7f3a1d2b9c",
  "owner": "James Smith",
  "owner_id": "person_c3a1b72e45",
  "description": "Send selected CVs and rates of 7 screened Python engineers to Daniel Lewis",
  "status": "COMMITMENT_OVERDUE",
  "extraction_confidence": 0.97,
  "type": "COMMITMENT",
  "response_type": null,
  "triggered_by_chunks": ["chunk_bf95c0e7a9"],
  "commitment_date": "2026-02-24",
  "thread_id": "thread_81c29afe0f",
  "thread_subject": "Request for Rapid Technical Assessment (Dragonfly Platform)",
  "evidence": "'I will get back to you with selected CVs and rates' — James Smith, Feb 24",
  "blocks": ["action_3d8e2f1a7b"],
  "blocked_by": ["action_9c4b5e6f2d"],
  "first_seen": "2026-02-24T13:33:00Z",
  "last_updated": "2026-03-06T22:00:00Z",
  "status_history": [
    {"status": "OPEN",               "date": "2026-02-24T13:33:00Z", "set_by": "pipeline"},
    {"status": "COMMITMENT_OVERDUE", "date": "2026-03-03T22:00:00Z", "set_by": "detector"}
  ]
}
```

- `response_type`: `"action"` | `"information"` | `null` — only populated for `REQUEST` type items.
- `commitment_date`: ISO date string of when the commitment was made — required for the asymmetric commitment detector to compute elapsed days.
- `status_history`: append-only log of every status transition. Each entry records the status value, the ISO timestamp, and who set it. `set_by` values: `"pipeline"` (Phase 2 LLM extraction or merge/diff), `"detector"` (asymmetric commitment detector), `"inactivity_detector"` (inactivity detector), `"manual"` (user via manual command). The current `status` field always mirrors the last entry in `status_history`. Never modify or delete history entries.

**v2 adds two timestamp fields:**
- `first_seen` — ISO timestamp of the run that first created this item
- `last_updated` — ISO timestamp of the last run that modified its status

### Action Item Status Values

| Status | Meaning | Set by |
|--------|---------|--------|
| `OPEN` | Identified, not yet done | Phase 2 LLM extraction |
| `IN_PROGRESS` | Evidence of partial work in thread | Phase 2 LLM extraction |
| `PARTIAL` | Partially fulfilled — some but not all delivered | Phase 2 LLM merge/diff |
| `SCHEDULED` | Time/date confirmed by both parties (MEETING items) | Phase 2 LLM extraction |
| `AWAITING_RESPONSE` | Ball in other party's court | Phase 2 LLM extraction |
| `TRIGGERED` | Automated system action initiated (SYSTEM_TRIGGER items) | Phase 2 LLM extraction |
| `COMMITMENT_OVERDUE` | COMMITMENT or REQUEST where no delivery evidence exists after threshold days | Asymmetric commitment detector |
| `INACTIVE` | Status unchanged for 30+ days — item hidden from active list; retained for historical analysis | Inactivity detector |
| `NEEDS_REVIEW` | Extraction confidence below `CONFIDENCE_THRESHOLD` — user must confirm or dismiss | Trust layer |
| `DONE` | Resolution evidence found in a later chunk or manually confirmed | Phase 2 merge/diff / manual |
| `DISMISSED` | User explicitly dismissed the item — confirmed false positive or irrelevant | Manual only |

**`DISMISSED` rules:**
- Terminal status — pipeline never reopens a DISMISSED item.
- Added to `TERMINAL_STATUSES` alongside `DONE` and `TRIGGERED`.
- `status_history` entry records `set_by: "manual"` and the ISO timestamp.
- Timing is used to derive training signal quality:
  - Dismissed within 48h of `created_at` → `label_source: DISMISSED_EARLY` → GOLD negative (false positive)
  - Dismissed after 48h → `label_source: DISMISSED_LATE` → SILVER negative (lower signal weight)
- DISMISSED items are excluded from the active dialog report and all pipeline views.
- Retained in the graph for historical analysis and training data export.

**`INACTIVE` rules:**
- Applied to any item whose `last_updated` has not changed in 30+ days, regardless of current status (including `COMMITMENT_OVERDUE`)
- Once set to `INACTIVE`, the item stays `INACTIVE` permanently — new emails in the same thread do not reactivate it
- New action items created from new emails are tracked fresh and independently
- `INACTIVE` items are excluded from the active dialog report but retained in the graph for historical analysis
- The only statuses exempt from going `INACTIVE` are `DONE` and `TRIGGERED` — these are already terminal

### Phase 2 — LLM Prompt Template

This is the versioned prompt sent to Claude for each thread pass.
The system message defines the extraction contract. The user message
carries the chunk bodies. They are kept separate so schema changes
don't require rewriting the thread content format.

```python
PROMPT_VERSION = "2.2"

PHASE2_SYSTEM_PROMPT = """
You are a CRM extraction engine. Your job is to read a sequence of email chunks
from a single thread and extract structured action items — commitments, requests,
decisions, and assignments — with the owner, type, and any deadline clearly identified.

## Output format

Return a JSON array of action items. Each item must follow this schema exactly:

{
  "type": string,              // see TYPE VALUES below
  "owner": string | null,      // person who owns this item (name as it appears in email)
  "description": string,       // concise one-sentence description, max 120 chars
  "response_type": string | null,  // only for REQUEST: "action" or "information"
  "commitment_date": string | null, // ISO date if a deadline is stated or strongly implied
  "extraction_confidence": float,  // 0.0–1.0, your confidence this is a real action item
  "triggered_by_chunks": [string], // chunk IDs that contain the evidence
  "evidence": string           // verbatim quote (≤25 words) from the email that triggered this
}

## Type values

COMMITMENT     — someone explicitly commits to do something ("I will", "I'll send", "let me get that to you")
REQUEST        — someone asks another person to do something or provide information
ASSIGNMENT     — a task is assigned to someone by a third party
DECISION_NEEDED — a decision is required before work can proceed
APPROVAL_NEEDED — something requires explicit sign-off from a named person
MEETING        — a meeting is being scheduled, proposed, or confirmed
FOLLOW_UP      — a follow-up was requested or is implied by context
INTRODUCTION   — someone is being introduced to someone else for the first time
SYSTEM_TRIGGER — an automated system event that requires a response

## Company type classification

For each external company mentioned in the thread, assign a type based on the
commercial direction visible in the email. Add a "company_type" field to each
action item using the company involved.

Use these signals — in order of priority:

CLIENT (you are selling to them):
- You sent a proposal, quote, SOW, or invoice to them
- They ask about your pricing, services, or capabilities
- They reference your deliverables or your team doing work
- Language signals: "your proposal", "your quote", "your services", "engagement",
  "statement of work", "can you help us with", "what would it cost"

VENDOR (they are selling to you):
- They sent you an invoice, renewal notice, or subscription confirmation
- They are pitching their product or service to you
- You are the buyer, they are the supplier
- Language signals: "your invoice", "your subscription", "your account",
  "renewal", "your plan", "our pricing", "our services", "we can help you"

PARTNER (mutual, non-transactional relationship):
- Referral language — they refer clients to you or you refer to them
- Co-selling, joint proposals, or shared pipeline
- Language signals: "introduce you to", "referring", "partnership", "together
  we could", "joint proposal", "our mutual client", "collaborate"

EXTERNAL_UNKNOWN (assign this when signals are absent or ambiguous):
- First contact with no commercial language yet
- Could be client, vendor, or partner — not enough signal to decide
- Do NOT guess. Default to external_unknown.

Rules:
- Internal company (from identity context above) is always type "internal" — never classify it.
- A company can only have one type. If signals conflict, pick the strongest one
  and lower extraction_confidence.
- Return the company type as "company_type" on each action item where the
  company is identifiable. The pipeline writes it to the company node.

## Critical rules

1. OWNER must be a name that appears in the email text. If you cannot identify the owner
   from the text, set owner to null — do not guess.
2. Extract from body_clean only. Do not extract from quoted_context — that content
   was already processed in earlier chunks.
3. CALENDAR_INVITE chunks: create a MEETING item using the structured invite fields
   (subject, date, attendees). Do not NLP-extract from the body.
4. AUTO_REPLY and MARKETING chunks: return an empty array []. Do not extract.
5. Pleasantries are not commitments. "I'll keep you posted" is not a COMMITMENT.
   "I will send you the contract by Friday" is.
6. If a commitment is already resolved in a later chunk (delivery evidence present),
   set extraction_confidence to 0.0 — it will be filtered as resolved.
7. One item per action. Do not merge two distinct commitments into one item.
8. Return [] if no real action items exist. Do not invent items.
9. Default company type to external_unknown when signals are weak. Never guess.
""".strip()


def build_phase2_user_message(pass_chunks: list, identity: dict) -> str:
    """
    Format the user message for a Phase 2 LLM call.
    Includes chunk metadata and clean body for each chunk in the pass.
    """
    lines = [
        f"Internal company: {identity['company_name']} "
        f"({', '.join(identity['internal_domains']) or 'solo — owner only'})",
        f"Owner email: {identity['owner_email']}",
        "",
        "## Email chunks (chronological, oldest first)",
        "",
    ]
    for chunk in pass_chunks:
        lines += [
            f"### Chunk {chunk['id']}",
            f"Date: {chunk.get('date', 'unknown')}",
            f"From: {chunk['sender'].get('name', '')} <{chunk['sender'].get('email', '')}>",
            f"To: {', '.join(r.get('name','') or r.get('email','') for r in chunk.get('recipients',[]))}",
            f"Type: {chunk.get('content_type', 'DIRECT_MESSAGE')}",
            f"Body:",
            chunk.get('body', '').strip(),
            "",
        ]
    lines.append("Extract all action items from the chunks above. Return a JSON array.")
    return "\n".join(lines)


FEW_SHOT_EXAMPLES = [
    # ── Example 1: Commitment buried in pleasantries ────────────────────────
    {
        "description": "Commitment buried in warm closing — must not be missed",
        "chunks": [
            {
                "id": "chunk_ex1a",
                "date": "2026-01-15",
                "sender": {"name": "Sarah Chen", "email": "sarah@clientco.com"},
                "recipients": [{"name": "James", "email": "james@acme.com"}],
                "content_type": "DIRECT_MESSAGE",
                "body": (
                    "Hi James, great catching up on the call today! "
                    "Always a pleasure. I'll make sure to loop in our legal team "
                    "and get the NDA over to you by end of next week. "
                    "Looking forward to moving this forward!"
                ),
            }
        ],
        "expected_output": [
            {
                "type": "COMMITMENT",
                "owner": "Sarah Chen",
                "description": "Send NDA to James, looping in legal team",
                "commitment_date": "2026-01-24",
                "extraction_confidence": 0.92,
                "triggered_by_chunks": ["chunk_ex1a"],
                "evidence": "I'll make sure to loop in our legal team and get the NDA over to you by end of next week",
            }
        ],
        "teaching_note": (
            "'Always a pleasure' and 'Looking forward to moving this forward' "
            "are pleasantries — ignored. The commitment is 'I'll... get the NDA over to you "
            "by end of next week' — extracted with inferred date."
        ),
    },

    # ── Example 2: REQUEST with no explicit deadline ────────────────────────
    {
        "description": "REQUEST where response_type is information and deadline is implicit",
        "chunks": [
            {
                "id": "chunk_ex2a",
                "date": "2026-02-03",
                "sender": {"name": "James Smith", "email": "james@acme.com"},
                "recipients": [{"name": "Mike Reeves", "email": "mike@acme.com"}],
                "content_type": "DIRECT_MESSAGE",
                "body": (
                    "Mike, before we go into the next call with Beta Corp "
                    "can you pull together what we know about their current "
                    "tech stack? Specifically interested in what CRM they're using."
                ),
            }
        ],
        "expected_output": [
            {
                "type": "REQUEST",
                "owner": "Mike Reeves",
                "response_type": "information",
                "description": "Pull together intel on Beta Corp tech stack, specifically CRM in use",
                "commitment_date": None,
                "extraction_confidence": 0.88,
                "triggered_by_chunks": ["chunk_ex2a"],
                "evidence": "can you pull together what we know about their current tech stack",
            }
        ],
        "teaching_note": (
            "No deadline stated → commitment_date null. Owner is the person asked "
            "('Mike Reeves'), not the sender. response_type is 'information'."
        ),
    },

    # ── Example 3: Multi-person thread, ambiguous owner ─────────────────────
    {
        "description": "Multi-person thread — owner is unambiguous despite multiple participants",
        "chunks": [
            {
                "id": "chunk_ex3a",
                "date": "2026-02-10",
                "sender": {"name": "Lisa Park", "email": "lisa@acme.com"},
                "recipients": [{"name": "Tom", "email": "tom@vendorco.com"}, {"name": "Dan", "email": "dan@acme.com"}],
                "content_type": "DIRECT_MESSAGE",
                "body": "Tom, Dan — we need the revised pricing sheet before Thursday. Tom, can you own that?",
            }
        ],
        "expected_output": [
            {
                "type": "REQUEST",
                "owner": "Tom",
                "response_type": "action",
                "description": "Deliver revised pricing sheet before Thursday",
                "commitment_date": "2026-02-13",
                "extraction_confidence": 0.95,
                "triggered_by_chunks": ["chunk_ex3a"],
                "evidence": "we need the revised pricing sheet before Thursday. Tom, can you own that?",
            }
        ],
        "teaching_note": (
            "Even though Lisa and Dan are also present, 'Tom, can you own that?' "
            "makes the owner unambiguous. Thursday inferred as 2026-02-13."
        ),
    },

    # ── Example 4: CALENDAR_INVITE — structured extraction only ─────────────
    {
        "description": "CALENDAR_INVITE — must use structured fields, no NLP",
        "chunks": [
            {
                "id": "chunk_ex4a",
                "date": "2026-02-20",
                "sender": {"name": "Emily Blunt", "email": "emily@partnerco.com"},
                "recipients": [{"name": "James Smith", "email": "james@acme.com"}],
                "content_type": "CALENDAR_INVITE",
                "body": "You are invited to: Q1 Kick-off Sync\nWhen: Feb 25, 2026 at 10:00 AM GMT\nWhere: Zoom\nOrganiser: Emily Blunt",
            }
        ],
        "expected_output": [
            {
                "type": "MEETING",
                "owner": "James Smith",
                "description": "Q1 Kick-off Sync with Emily Blunt (PartnerCo) — Feb 25 10:00 GMT",
                "commitment_date": "2026-02-25",
                "extraction_confidence": 1.0,
                "triggered_by_chunks": ["chunk_ex4a"],
                "evidence": "Q1 Kick-off Sync — Feb 25, 2026 at 10:00 AM GMT",
            }
        ],
        "teaching_note": (
            "content_type is CALENDAR_INVITE → extract MEETING from structured fields only. "
            "Owner is the internal recipient (James Smith), not the external organiser."
        ),
    },

    # ── Example 5: AUTO_REPLY — must return empty array ─────────────────────
    {
        "description": "AUTO_REPLY — return empty array, extract nothing",
        "chunks": [
            {
                "id": "chunk_ex5a",
                "date": "2026-02-21",
                "sender": {"name": "Emily Blunt", "email": "emily@partnerco.com"},
                "recipients": [{"name": "James Smith", "email": "james@acme.com"}],
                "content_type": "AUTO_REPLY",
                "body": (
                    "Thank you for your email. I am currently out of office "
                    "until March 3rd with limited access to email. "
                    "For urgent matters please contact john@partnerco.com."
                ),
            }
        ],
        "expected_output": [],
        "teaching_note": (
            "content_type is AUTO_REPLY → always return []. "
            "Do not extract the 'contact john' as a FOLLOW_UP or INTRODUCTION — "
            "this is an automated message, not a deliberate action item."
        ),
    },
]


def extract_action_items_for_thread(pass_chunks: list,
                                     identity: dict,
                                     include_few_shot: bool = True) -> list:
    """
    Call the Claude API to extract action items from a thread pass.
    Returns raw list of dicts (not yet validated — call validate_phase2_output next).

    include_few_shot: True for first run (cold model), can be False for
    incremental runs on familiar threads to reduce token usage.
    """
    import anthropic
    client = anthropic.Anthropic()

    system_content = PHASE2_SYSTEM_PROMPT

    if include_few_shot:
        examples_text = "\n\n## Few-shot examples\n\n"
        for i, ex in enumerate(FEW_SHOT_EXAMPLES, 1):
            examples_text += f"### Example {i}: {ex['description']}\n"
            examples_text += f"Teaching note: {ex['teaching_note']}\n"
            examples_text += f"Expected output:\n{json.dumps(ex['expected_output'], indent=2)}\n\n"
        system_content = PHASE2_SYSTEM_PROMPT + "\n\n" + examples_text.strip()

    user_message = build_phase2_user_message(pass_chunks, identity)

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        system=system_content,
        messages=[{"role": "user", "content": user_message}],
    )

    raw_text = response.content[0].text.strip()

    # Extract JSON array from response (model may include explanation text)
    import re
    json_match = re.search(r'\[.*\]', raw_text, re.DOTALL)
    if not json_match:
        return []   # no JSON found — trust layer will log this as a rejection
    try:
        items = json.loads(json_match.group())
        return items if isinstance(items, list) else []
    except json.JSONDecodeError:
        return []
```

**Prompt version tracking:** `PROMPT_VERSION` is stored in
`aicrm_action_items_graph.json` meta as `phase2_prompt_version`. When the
prompt version changes between runs, the skill logs a warning that existing
items were extracted with a different prompt — useful for debugging drift.

---

### Phase 2 — Trust Layer

Every item returned by the Phase 2 LLM call passes through three guards before
entering the graph. Items that fail validation are logged and discarded — they
never silently corrupt the graph.

```python
# ── Configuration ──────────────────────────────────────────────────────────
CONFIDENCE_THRESHOLD   = 0.75  # items below this score are flagged, not discarded
                                # (low-confidence items stored with status NEEDS_REVIEW)
PHASE2_MODEL           = "claude"   # "claude" | "crm_extract" (v3 fine-tuned model)

# JSON Schema for a single action item as returned by Phase 2 LLM
ACTION_ITEM_SCHEMA = {
    "type": "object",
    "required": ["type", "owner", "description"],
    "properties": {
        "type":                  {"type": "string"},
        "owner":                 {"type": ["string", "null"]},
        "description":           {"type": "string", "minLength": 5},
        "response_type":         {"type": ["string", "null"]},
        "commitment_date":       {"type": ["string", "null"]},
        "extraction_confidence": {"type": "number", "minimum": 0.0, "maximum": 1.0},
        "triggered_by_chunks":   {"type": "array"},
        "blocks":                {"type": "array"},
        "blocked_by":            {"type": "array"},
    },
    "additionalProperties": True,
}

VALID_ACTION_TYPES = {
    'COMMITMENT', 'REQUEST', 'ASSIGNMENT', 'DECISION_NEEDED',
    'APPROVAL_NEEDED', 'MEETING', 'FOLLOW_UP', 'INTRODUCTION',
    'SYSTEM_TRIGGER',
}

# ── Guard 1: JSON schema validation ────────────────────────────────────────
def validate_action_item_schema(item: dict) -> tuple[bool, str]:
    """
    Check required fields and basic type constraints.
    Returns (is_valid, reason). Uses stdlib only — no jsonschema dependency.
    """
    if not isinstance(item, dict):
        return False, "item is not a dict"
    for field in ACTION_ITEM_SCHEMA['required']:
        if field not in item:
            return False, f"missing required field: '{field}'"
    if item.get('type') not in VALID_ACTION_TYPES:
        return False, f"unknown action type: '{item.get('type')}'"
    desc = item.get('description', '')
    if not isinstance(desc, str) or len(desc) < 5:
        return False, "description too short or wrong type"
    conf = item.get('extraction_confidence')
    if conf is not None and not (0.0 <= float(conf) <= 1.0):
        return False, f"extraction_confidence out of range: {conf}"
    return True, "ok"


# ── Guard 2: owner grounding check ─────────────────────────────────────────
def owner_in_chunk_body(owner: str | None, chunks: list) -> bool:
    """
    Check that the owner name appears in at least one chunk body in the thread.
    Prevents hallucinated owner names from entering the graph.
    A None owner (unassigned) passes automatically.
    """
    if not owner:
        return True
    owner_lower = owner.lower().strip()
    for chunk in chunks:
        body = (chunk.get('body', '') + ' ' + chunk.get('quoted_context', '')).lower()
        # Match on first name, last name, or email prefix
        parts = owner_lower.replace('.', ' ').split()
        if any(part in body for part in parts if len(part) > 2):
            return True
    return False


# ── Guard 3: confidence routing ────────────────────────────────────────────
def route_by_confidence(item: dict,
                         body_tokenized: str = '',
                         structural_metadata: dict = None) -> dict:
    """
    Items below CONFIDENCE_THRESHOLD are not discarded — they are stored
    with status NEEDS_REVIEW so the user can confirm or reject them.
    Items above threshold proceed normally.

    Regardless of confidence level, a training example is recorded here —
    immediately after extraction, before any user interaction. This captures
    the tokenized input and Sonnet's original output as an immutable snapshot.
    Behavioral labels are added later when the user acts on the item.

    body_tokenized and structural_metadata must be passed from the extraction
    call site — they are what was actually sent to the LLM.
    """
    conf = float(item.get('extraction_confidence', 1.0))
    if conf < CONFIDENCE_THRESHOLD:
        item['status'] = 'NEEDS_REVIEW'
        item['status_history'] = [{
            'status':  'NEEDS_REVIEW',
            'date':    datetime.now(timezone.utc).isoformat(),
            'set_by':  'trust_layer',
            'reason':  f"extraction_confidence {conf:.2f} below threshold {CONFIDENCE_THRESHOLD}",
        }]

    # Always record training example — confidence level does not gate this.
    # Low-confidence items are the most valuable training candidates once labeled.
    record_training_example(
        item=item,
        body_tokenized=body_tokenized,
        structural_metadata=structural_metadata or {},
    )

    return item


# ── Validation pipeline ─────────────────────────────────────────────────────
def validate_phase2_output(raw_items: list, thread_chunks: list) -> tuple[list, list]:
    """
    Run all three guards on a list of raw LLM-extracted items.
    Returns (valid_items, rejected_items).
    Rejected items are logged but never written to the graph.
    """
    valid, rejected = [], []
    for item in raw_items:
        ok, reason = validate_action_item_schema(item)
        if not ok:
            rejected.append({'item': item, 'reason': reason})
            continue
        if not owner_in_chunk_body(item.get('owner'), thread_chunks):
            rejected.append({'item': item,
                             'reason': f"owner '{item.get('owner')}' not found in chunk bodies"})
            continue
        item = route_by_confidence(item)
        valid.append(item)
    return valid, rejected
```

**`NEEDS_REVIEW` in the dialog report** — appears as a dedicated section between
OVERDUE and OPEN items, so the user sees them without them cluttering the main list:

```
🔍  NEEDS REVIEW  (low confidence — please confirm or dismiss)
  ? James Smith → Send technical spec to Lisa  [COMMITMENT · conf: 0.61]
    To confirm: "confirm action [id]"   To dismiss: "dismiss action [id]"
```

**`PHASE2_MODEL` config flag** — placeholder for v3 CRMExtract integration.
When set to `"crm_extract"`, `extract_action_items_for_thread()` routes to
the local inference endpoint instead of Claude API. Items returned below
`CONFIDENCE_THRESHOLD` automatically fall back to Claude for re-extraction.

---

### Phase 2 — Merge / Diff (Incremental Runs)

On incremental runs, Phase 2 runs only for threads that received new chunks.
Existing action items from unaffected threads are left untouched.

```python
from datetime import datetime, timezone

def merge_action_items(existing_items: list, new_items: list) -> list:
    """
    Merge newly extracted items into existing set.

    Matching logic:
      - Primary match: stable_id (same description + owner → same ID)
      - On match: update status, confidence, last_updated. Never duplicate.
      - On no match: append as new item.

    Status resolution rules:
      - OPEN → DONE   if new extraction finds resolution evidence in newer chunks
      - OPEN → PARTIAL if partial delivery detected
      - Any status → COMMITMENT_OVERDUE if time elapsed without delivery
      - Never downgrade: DONE → OPEN is not allowed
    """
    TERMINAL_STATUSES = {'DONE', 'TRIGGERED', 'DISMISSED'}
    now     = datetime.now(timezone.utc)          # datetime object — passed to record_status_change
    now_str = now.isoformat()                     # string — used for direct field assignment

    existing_by_id = {ai['id']: ai for ai in existing_items}

    for new_item in new_items:
        item_id = new_item['id']
        if item_id in existing_by_id:
            existing = existing_by_id[item_id]
            # Do not overwrite terminal statuses
            if existing.get('status') not in TERMINAL_STATUSES:
                if existing.get('status') != new_item['status']:
                    record_status_change(existing, new_item['status'],
                                         set_by='pipeline', now=now)  # ← was now_dt (undefined)
                existing['extraction_confidence'] = new_item.get('extraction_confidence')
                # Extend chunk evidence without duplicating
                existing_chunks = set(existing.get('triggered_by_chunks', []))
                for cid in new_item.get('triggered_by_chunks', []):
                    existing_chunks.add(cid)
                existing['triggered_by_chunks'] = list(existing_chunks)
        else:
            new_item['first_seen']    = now_str
            new_item['last_updated']  = now_str
            existing_by_id[item_id]   = new_item

    return list(existing_by_id.values())

MAX_CHUNKS_PER_PHASE2_PASS = 50      # max chunks sent to LLM in a single call
MAX_CHARS_PER_PHASE2_PASS  = 80_000  # ~80KB hard ceiling (safety net for long bodies)

def _chunk_passes(chunks: list) -> list[list]:
    """
    Split a thread's chunks into passes that respect both limits.
    Each pass is a contiguous window that stays within MAX_CHUNKS and MAX_CHARS.
    Passes overlap by 2 chunks so context is not lost at boundaries.
    Returns a list of chunk lists (passes).
    """
    passes, current, current_chars = [], [], 0
    OVERLAP = 2

    for chunk in chunks:
        chunk_chars = len(chunk.get('body', '')) + len(chunk.get('quoted_context', ''))
        if current and (
            len(current) >= MAX_CHUNKS_PER_PHASE2_PASS
            or current_chars + chunk_chars > MAX_CHARS_PER_PHASE2_PASS
        ):
            passes.append(current)
            # Carry last OVERLAP chunks into next pass for context continuity
            current       = current[-OVERLAP:]
            current_chars = sum(len(c.get('body','')) + len(c.get('quoted_context',''))
                                for c in current)
        current.append(chunk)
        current_chars += chunk_chars

    if current:
        passes.append(current)
    return passes


def run_phase2_incremental(chunk_graph: dict,
                            existing_action_graph: dict,
                            threads_updated: list) -> dict:
    """
    Re-run Phase 2 only for threads that received new chunks.
    Splits large threads into windowed passes to respect context limits.
    Merges results into existing action items graph.
    Prints progress every 10 threads with ETA.
    """
    import time
    total      = len(threads_updated)
    t_start    = time.time()
    SECS_PER_THREAD = 3  # kept in sync with run_full / run_incremental estimate

    for idx, thread_id in enumerate(threads_updated):
        # Progress update every 10 threads (and on the last one)
        if total > 0 and (idx % 10 == 0 or idx == total - 1):
            elapsed   = time.time() - t_start
            done      = idx + 1
            remaining = total - done
            if idx > 0:
                secs_per = elapsed / idx
                eta_secs  = int(remaining * secs_per)
                eta_str   = f"{eta_secs // 60}m {eta_secs % 60}s" if eta_secs >= 60 else f"{eta_secs}s"
            else:
                eta_secs  = remaining * SECS_PER_THREAD
                eta_str   = f"~{eta_secs // 60}m {eta_secs % 60}s" if eta_secs >= 60 else f"~{eta_secs}s"
            bar_fill  = int(20 * done / total)
            bar       = "█" * bar_fill + "░" * (20 - bar_fill)
            print(f"        [{bar}] {done}/{total}  ETA {eta_str}")

        # Get chunks for this thread (chronological order)
        thread_chunks = sorted(
            [c for c in chunk_graph['nodes']['chunks'] if c['thread_id'] == thread_id],
            key=lambda c: c['position']
        )
        # Split into context-safe passes
        passes = _chunk_passes(thread_chunks)
        all_new_items = []
        for pass_chunks in passes:
            raw_items = extract_action_items_for_thread(pass_chunks, identity)   # LLM call
            valid_items, rejected = validate_phase2_output(raw_items, pass_chunks)
            if rejected:
                # Log rejections to graph meta for inspection — do not raise
                existing_action_graph.setdefault('meta', {}).setdefault(
                    'validation_rejections', []).extend(rejected)
            all_new_items.extend(valid_items)
        # Merge all passes into existing graph
        existing_action_graph['action_items'] = merge_action_items(
            existing_action_graph['action_items'], all_new_items
        )

    # Rebuild edges from updated blocks/blocked_by
    existing_action_graph['edges'] = []
    for ai in existing_action_graph['action_items']:
        for blocked_id in ai.get('blocks', []):
            existing_action_graph['edges'].append(
                {'from': ai['id'], 'relation': 'BLOCKS', 'to': blocked_id}
            )

    # Update meta counts
    from collections import Counter
    existing_action_graph['meta']['last_updated']      = datetime.now(timezone.utc).isoformat()
    existing_action_graph['meta']['total_action_items'] = len(existing_action_graph['action_items'])
    existing_action_graph['meta']['by_status'] = dict(
        Counter(ai['status'] for ai in existing_action_graph['action_items'])
    )
    return existing_action_graph
```

---

## Module: Relationship Health Scoring

This module runs **after Phase 2**, computing a 0–100 health score for every
external company in the graph. It requires no LLM call — all inputs are already
in the chunk graph and action items graph. The score is written directly into
each company node and is recomputed on every pipeline run.

### Why health scoring replaces CRM "last touched" fields

Traditional CRMs track "last activity date" as their primary health signal. That
is a single data point that misses direction: a company contacted 3 days ago after
6 weeks of silence looks identical to a company contacted 3 days ago after a
month of daily exchanges. The health score captures both recency AND trend.

### Scoring components

| Component | Weight | What it measures |
|-----------|--------|-----------------|
| Recency | 35% | Days since last contact — decays non-linearly (0 days = 100, 7 days = 85, 30 days = 50, 60+ days = 10) |
| Frequency trend | 25% | contact_count_30d vs. contact_prev_30d — increasing = 100, stable = 70, declining = 40, silent = 0 |
| Overdue items | 20% | Fraction of action items NOT overdue — 0 overdue = 100, all overdue = 0 |
| Deal momentum | 20% | Days stuck in current stage vs. expected — 0–7 days = 100, 8–14 = 75, 15–30 = 50, 30+ = 20 |

```python
from datetime import datetime, timezone

def compute_health_score(company: dict,
                         action_graph: dict,
                         now: datetime = None) -> int:
    """
    Compute 0-100 relationship health score for a company.
    Higher = healthier. Recomputed on every run.
    """
    if now is None:
        now = datetime.now(timezone.utc)

    # --- Recency (35%) ---
    last_activity = company.get('last_activity_date')
    if last_activity:
        try:
            last_dt = datetime.fromisoformat(last_activity)
            if last_dt.tzinfo is None:
                last_dt = last_dt.replace(tzinfo=timezone.utc)
            days_ago = (now - last_dt).days
        except ValueError:
            days_ago = 999
    else:
        days_ago = 999

    if   days_ago == 0:  recency = 100
    elif days_ago <= 3:  recency = 95
    elif days_ago <= 7:  recency = 85
    elif days_ago <= 14: recency = 70
    elif days_ago <= 30: recency = 50
    elif days_ago <= 60: recency = 25
    else:                recency = 5

    # --- Frequency trend (25%) ---
    freq_30  = company.get('contact_frequency_30d', 0)
    freq_60  = company.get('contact_frequency_prev_30d', 0)
    if   freq_30 == 0:                   trend = 0   # silent
    elif freq_60 == 0 and freq_30 > 0:   trend = 80  # re-engaging
    elif freq_30 > freq_60 * 1.2:        trend = 100 # increasing
    elif freq_30 >= freq_60 * 0.8:       trend = 70  # stable
    else:                                trend = 30  # declining

    # --- Overdue items (20%) ---
    company_id     = company['id']
    company_items  = [ai for ai in action_graph.get('action_items', [])
                      if ai.get('thread_id') in company.get('threads', [])]
    total_items    = len(company_items)
    overdue_items  = sum(1 for ai in company_items
                        if ai.get('status') == 'COMMITMENT_OVERDUE')
    if total_items == 0:
        overdue_score = 80   # no items = neutral
    else:
        overdue_score = int((1 - overdue_items / total_items) * 100)

    # --- Deal momentum (20%) ---
    days_in_stage = company.get('days_in_current_stage', 0) or 0
    if   days_in_stage <= 7:  momentum = 100
    elif days_in_stage <= 14: momentum = 75
    elif days_in_stage <= 30: momentum = 50
    elif days_in_stage <= 60: momentum = 25
    else:                     momentum = 10

    # Weighted sum
    score = int(
        recency      * 0.35 +
        trend        * 0.25 +
        overdue_score * 0.20 +
        momentum     * 0.20
    )
    return max(0, min(100, score))


def compute_frequency_trend(company: dict) -> str:
    freq_30 = company.get('contact_frequency_30d', 0)
    freq_60 = company.get('contact_frequency_prev_30d', 0)
    if   freq_30 == 0:                 return 'silent'
    elif freq_60 == 0:                 return 'increasing'
    elif freq_30 > freq_60 * 1.2:     return 'increasing'
    elif freq_30 >= freq_60 * 0.8:    return 'stable'
    else:                              return 'declining'


def update_all_health_scores(action_graph: dict, chunk_graph: dict) -> dict:
    """Update health scores for all companies. Run post-Phase-2, pre-report."""
    now = datetime.now(timezone.utc)

    # Build lookup: company_id → list of thread_ids
    # (via persons → WORKS_AT → company edges)
    company_threads = {}
    for edge in chunk_graph.get('edges', []):
        if edge['relation'] == 'WORKS_AT':
            person_id  = edge['from']
            company_id = edge['to']
            company_threads.setdefault(company_id, set())
            # find threads containing this person
            for e2 in chunk_graph['edges']:
                if e2['relation'] in ('SENT_BY', 'RECEIVED_BY') and e2['to'] == person_id:
                    chunk_id = e2['from']
                    chunk    = next((c for c in chunk_graph['nodes']['chunks']
                                     if c['id'] == chunk_id), None)
                    if chunk:
                        company_threads[company_id].add(chunk['thread_id'])

    for company in action_graph.get('companies', []):
        c_id = company['id']
        # Attach thread list to company node
        company['threads'] = list(company_threads.get(c_id, []))

        # Compute contact frequency for rolling windows
        all_chunks = [c for c in chunk_graph['nodes']['chunks']
                      if c.get('thread_id') in company['threads']]
        company['contact_frequency_30d'] = sum(
            1 for c in all_chunks
            if _days_ago(c.get('date', ''), now) <= 30
        )
        company['contact_frequency_prev_30d'] = sum(
            1 for c in all_chunks
            if 30 < _days_ago(c.get('date', ''), now) <= 60
        )
        company['frequency_trend'] = compute_frequency_trend(company)

        # Last activity date
        dated = [c for c in all_chunks if c.get('date')]
        if dated:
            latest = max(dated, key=lambda c: c['date'])
            company['last_activity_date'] = latest['date'][:10]
            company['days_since_last_contact'] = _days_ago(latest['date'], now)

        # Open / overdue item counts
        c_items = [ai for ai in action_graph.get('action_items', [])
                   if ai.get('thread_id') in company['threads']]
        company['open_action_items']    = sum(1 for ai in c_items
                                              if ai['status'] not in ('DONE', 'TRIGGERED'))
        company['overdue_action_items'] = sum(1 for ai in c_items
                                              if ai['status'] == 'COMMITMENT_OVERDUE')

        # Health score
        company['health_score']         = compute_health_score(company, action_graph, now)
        company['health_score_updated'] = now.isoformat()

    return action_graph


def _days_ago(date_str: str, now: datetime) -> int:
    try:
        dt = datetime.fromisoformat(date_str[:19])
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        return (now - dt).days
    except (ValueError, TypeError):
        return 9999
```

### Health score in the dialog report

The dialog report adds an account health section when health scoring is active:

```
📊  ACCOUNT HEALTH
   🟢  Acme Corp          87  (Proposal · 3 open items · increasing)
   🟡  Beta Industries     61  (Qualified · 1 overdue · stable)
   🔴  Gamma Ltd           28  (Prospecting · silent 38 days)
   ⚫  Delta Corp          12  (Proposal · 2 overdue · declining · 47 days in stage)
```

Colour thresholds: 🟢 75–100 · 🟡 50–74 · 🔴 25–49 · ⚫ 0–24

### Where it runs in the orchestrator

```python
# After asymmetric commitment detector, before reporting:
action_graph = update_all_health_scores(action_graph, chunk_graph)
safe_overwrite(ACTION_GRAPH_PATH, action_graph)
```

---

## Module: Asymmetric Commitment Detector

This module runs **after Phase 2**, as a post-processing pass on the full action
items graph. It does not require an LLM call — it operates purely on the data
already in the graph.

### What it detects

A COMMITMENT or REQUEST is "asymmetric" when:
1. Person A made a commitment or was asked to do something (chunk at position N)
2. There is no delivery evidence in any later chunk (positions N+1 through newest)
3. The elapsed time since `commitment_date` exceeds the configured threshold

This is `COMMITMENT_OVERDUE` computed systematically rather than relying on the
LLM to notice it during Phase 2 traversal. The LLM can miss these — especially
in long threads where the commitment appears early and the latest messages discuss
a different topic entirely.

### Configuration

```python
OVERDUE_THRESHOLDS = {
    'COMMITMENT': 7,   # days — explicit "I will do X" with no follow-through
    'REQUEST':    5,   # days — asked for something, no acknowledgement or delivery
    'FOLLOW_UP':  3,   # days — time-bound check-in past its date
    'APPROVAL_NEEDED': 3,  # days — sign-off pending
    'DECISION_NEEDED': 5,  # days — choice pending
}

# Items in these statuses are never marked overdue
EXEMPT_STATUSES = {'DONE', 'TRIGGERED', 'COMMITMENT_OVERDUE', 'SCHEDULED'}
```

### Delivery evidence detection

```python
_DELIVERY_SIGNALS = [
    # Completion language
    r'\b(done|completed|finished|sent|delivered|shared|attached|provided)\b',
    r'\bplease find\b', r'\bhere is\b', r'\bhere are\b',
    r'\bas promised\b', r'\bas requested\b', r'\bper your request\b',
    # Acknowledgement language
    r'\b(received|got it|noted|confirmed|thanks for (sending|sharing|providing))\b',
    r'\bthank you for\b',
    # Explicit closure
    r'\b(resolved|closed|sorted|no longer needed|cancell)\b',
]

def has_delivery_evidence(action_item: dict, chunks_by_id: dict) -> bool:
    """
    Check chunks AFTER the commitment chunk for delivery evidence.
    Only looks at chunks newer than the commitment (higher position).
    """
    triggered_chunk_ids = action_item.get('triggered_by_chunks', [])
    if not triggered_chunk_ids:
        return False

    # Find the position of the earliest triggering chunk
    trigger_positions = [
        chunks_by_id[cid]['position']
        for cid in triggered_chunk_ids
        if cid in chunks_by_id
    ]
    if not trigger_positions:
        return False

    min_trigger_pos = min(trigger_positions)
    thread_id = action_item.get('thread_id')

    # Get all chunks in this thread that are AFTER the commitment chunk
    later_chunks = [
        c for c in chunks_by_id.values()
        if c.get('thread_id') == thread_id
        and c.get('position', 0) > min_trigger_pos
        and c.get('content_type') not in ('AUTO_REPLY', 'MARKETING')
    ]

    compiled = [re.compile(p, re.IGNORECASE) for p in _DELIVERY_SIGNALS]
    for chunk in later_chunks:
        text = chunk.get('body', '') + ' ' + chunk.get('quoted_context', '')
        if any(pat.search(text) for pat in compiled):
            return True
    return False
```

### Detector function

```python
from datetime import datetime, timezone, timedelta

def detect_overdue_commitments(action_graph: dict,
                                chunk_graph: dict,
                                thresholds: dict = None) -> dict:
    """
    Post-Phase-2 pass. Marks COMMITMENT / REQUEST / FOLLOW_UP items as
    COMMITMENT_OVERDUE if:
      - status is not in EXEMPT_STATUSES
      - commitment_date is set
      - elapsed days > threshold for this type
      - no delivery evidence found in later chunks

    Mutates action_graph in place and returns it.
    """
    if thresholds is None:
        thresholds = OVERDUE_THRESHOLDS

    now          = datetime.now(timezone.utc)
    chunks_by_id = {c['id']: c for c in chunk_graph['nodes']['chunks']}
    overdue_count = 0

    for item in action_graph['action_items']:
        if item.get('status') in EXEMPT_STATUSES:
            continue
        item_type  = item.get('type')
        threshold  = thresholds.get(item_type)
        if threshold is None:
            continue

        commitment_date_str = item.get('commitment_date')
        if not commitment_date_str:
            continue

        try:
            commitment_dt = datetime.fromisoformat(commitment_date_str)
            if commitment_dt.tzinfo is None:
                commitment_dt = commitment_dt.replace(tzinfo=timezone.utc)
        except ValueError:
            continue

        elapsed_days = (now - commitment_dt).days
        if elapsed_days < threshold:
            continue

        # Check for delivery evidence in later chunks
        if has_delivery_evidence(item, chunks_by_id):
            continue

        # Mark overdue
        record_status_change(item, 'COMMITMENT_OVERDUE',
                              set_by='detector', now=now)
        item['overdue_days'] = elapsed_days
        overdue_count += 1

    # Update meta counts
    from collections import Counter
    action_graph['meta']['by_status'] = dict(
        Counter(ai['status'] for ai in action_graph['action_items'])
    )
    action_graph['meta']['overdue_count'] = overdue_count

    return action_graph
```

### Where it runs in the orchestrator

```python
# After Phase 2 extraction / merge, before reporting:
action_graph = detect_overdue_commitments(action_graph, chunk_graph)
safe_overwrite(ACTION_GRAPH_PATH, action_graph)
```

This runs on every pipeline invocation — both first run and subsequent runs — so
that items already in the graph can become overdue as time passes, even with no
new emails.

---

### Action Items Graph — Top-Level JSON Structure

```json
{
  "meta": {
    "phase": "action_items_graph",
    "version": "2.0",
    "source_graph": "aicrm_chunk_graph.json",
    "last_updated": "2026-02-24T22:00:00Z",
    "query_method": "LLM_graph_traversal",
    "total_action_items": 19,
    "total_persons": 23,
    "total_companies": 11,
    "total_events": 9,
    "total_edges": 10,
    "by_status": {"OPEN": 12, "PARTIAL": 1, "SCHEDULED": 2, "DONE": 1,
                  "AWAITING_RESPONSE": 1, "IN_PROGRESS": 2, "TRIGGERED": 1},
    "by_type": {"COMMITMENT": 5, "REQUEST": 6, "ASSIGNMENT": 1,
                "MEETING": 2, "DECISION_NEEDED": 1, "APPROVAL_NEEDED": 1,
                "FOLLOW_UP": 1, "INTRODUCTION": 1, "SYSTEM_TRIGGER": 1},
    "by_event_type": {"MEETING_CANCELLED": 2, "MEETING_RESCHEDULED": 1,
                      "NDA_SIGNED": 2, "DEAL_WON": 2, "SPEAKER_CONFIRMED": 1,
                      "MEETING_INVITED": 1},
    "by_thread": {"Thread subject": ["action_id_1", "action_id_2"]}
  },
  "persons":      [ { ...person node with role + company_type... } ],
  "companies":    [ { ...company node with type... } ],
  "events":       [ { ...event node with evidence_chunk_id + triggers_action... } ],
  "action_items": [ { ...action item node with first_seen + last_updated... } ],
  "edges": [
    {"from": "action_id", "relation": "BLOCKS",   "to": "action_id"},
    {"from": "event_id",  "relation": "TRIGGERS",  "to": "action_id"}
  ]
}
```

### Action Items Graph — Edge Types

| Relation | From → To | Meaning |
|----------|-----------|---------|
| `BLOCKS` | action → action | Item A must complete before item B |
| `TRIGGERS` | event → action | Real-world event created this action item |

Cross-graph references:
- `action_item.triggered_by_chunks` → `chunk.id` in chunk graph
- `action_item.owner_id` → `person.id` in this same graph
- `action_item.thread_id` → `thread.id` in chunk graph
- `event.evidence_chunk_id` → `chunk.id` in chunk graph
- `person.id` — same stable hash across both graphs

---

---

## Module: Inactivity Detector

Runs **after the Asymmetric Commitment Detector**, as the final post-processing
pass before reporting. Marks any action item whose status has not changed in
30+ days as `INACTIVE`, hiding it from the active report while retaining it
in the graph for historical analysis.

### Configuration

```python
INACTIVITY_THRESHOLD_DAYS = 30   # days of status silence before marking INACTIVE

# These statuses are terminal — never marked INACTIVE
INACTIVITY_EXEMPT_STATUSES = {'DONE', 'TRIGGERED', 'DISMISSED'}
# Note: INACTIVE itself is also implicitly exempt (already set)
```

### Detector function

```python
from datetime import datetime, timezone, timedelta

def detect_inactive_items(action_graph: dict,
                           threshold_days: int = INACTIVITY_THRESHOLD_DAYS) -> dict:
    """
    Mark action items INACTIVE if their status has not changed in threshold_days.

    Checks `last_updated` on each item. If the gap between last_updated and
    now exceeds the threshold, and the item is not in an exempt terminal status,
    set status to INACTIVE and append to status_history.

    Items already INACTIVE are skipped (idempotent).
    """
    now       = datetime.now(timezone.utc)
    cutoff    = now - timedelta(days=threshold_days)
    inactive_count = 0

    for item in action_graph.get('action_items', []):
        if item.get('status') in INACTIVITY_EXEMPT_STATUSES:
            continue
        if item.get('status') == 'INACTIVE':
            continue  # already set — skip

        last_updated_str = item.get('last_updated') or item.get('first_seen')
        if not last_updated_str:
            continue

        try:
            last_updated = datetime.fromisoformat(last_updated_str)
            if last_updated.tzinfo is None:
                last_updated = last_updated.replace(tzinfo=timezone.utc)
        except ValueError:
            continue

        if last_updated <= cutoff:
            record_status_change(item,
                                 new_status='INACTIVE',
                                 set_by='inactivity_detector',
                                 now=now)
            inactive_count += 1

    # Update meta counts
    from collections import Counter
    action_graph['meta']['by_status'] = dict(
        Counter(ai['status'] for ai in action_graph['action_items'])
    )
    action_graph['meta']['inactive_count'] = inactive_count

    return action_graph
```

### Where it runs in the orchestrator

```python
# Execution order post-Phase-2:
action_graph = update_all_health_scores(action_graph, chunk_graph)     # health scoring
action_graph = detect_overdue_commitments(action_graph, chunk_graph)   # overdue detector
action_graph = detect_inactive_items(action_graph)                     # inactivity detector
safe_overwrite(ACTION_GRAPH_PATH, action_graph)
# then: reporting
```

### INACTIVE items in the dialog report

Active report sections (`OPEN`, `OVERDUE`, `SCHEDULED`, etc.) filter out
`INACTIVE` items entirely. A single summary line at the bottom acknowledges them:

```
─────────────────────────────────────────────────────
  ⚫  12 items hidden (INACTIVE — no activity 30+ days)
     To review: ask "show inactive items for [company]"
```

### UNCLASSIFIED companies in the dialog report

Companies with `type: external_unknown` (or `type_confidence < 0.7`) appear
as a dedicated section at the bottom of the report, after action items but
before the INACTIVE summary. This ensures unclassified companies are surfaced
on every run until the user resolves them — not buried in graph data.

```
─────────────────────────────────────────────────────
  ❓  COMPANIES NEEDING CLASSIFICATION  (3)
─────────────────────────────────────────────────────
  PartnerCo       partnerco.com    first email Feb 25   → "set PartnerCo as client|vendor|partner"
  BetaCorp        betacorp.com     1 open action item   → "set BetaCorp as client|vendor|partner"
  Unknown Ltd     unknown.co.uk    no activity          → "set Unknown Ltd as client|vendor|partner"
─────────────────────────────────────────────────────
```

The section disappears once all companies are classified. Companies classified
manually with `type_source: "manual"` never reappear here even if the LLM
would have assigned `external_unknown` on a subsequent run.

```python
def get_unclassified_companies(action_graph: dict) -> list:
    """
    Return companies that need user classification.
    Includes: type == 'external_unknown', OR
              type_confidence < 0.70 and type_source != 'manual'.
    """
    return [
        c for c in action_graph.get('companies', [])
        if c.get('type') == 'external_unknown'
        or (
            c.get('type_source') != 'manual'
            and float(c.get('type_confidence') or 1.0) < 0.70
        )
    ]
```

---

## Module: Deal Entity

This module runs **after Phase 2 and after health scoring**, deriving deal nodes
from commercial events already in the graph. It requires no LLM call — deal
detection is entirely rule-based, triggered by the event taxonomy.

A deal is a named commercial relationship between the internal team and an
external company, with a lifecycle stage and evidence trail. Every deal is
anchored to at least one commercial event.

### Deal node schema

```json
{
  "id": "deal_a3f9c21d7e",
  "name": "Acme Corp — Platform Agreement",
  "company_id": "company_b71d3a0f12",
  "company_name": "Acme Corp",
  "stage": "PROPOSAL",
  "stage_entered_date": "2026-02-15",
  "days_in_stage": 9,
  "owner_id": "person_internal_abc123",
  "owner": "James Smith",
  "threads": ["thread_af75f00118", "thread_81c29afe0f"],
  "persons_involved": ["person_a3f9c21d7e", "person_b2c8d31f44"],
  "evidence_events": ["event_5f3a1d2b9c"],
  "open_action_items": ["action_7f3a1d2b9c"],
  "created_date": "2025-10-01",
  "last_activity_date": "2026-02-24",
  "outcome": null
}
```

- `outcome`: `null` (active) | `"WON"` | `"LOST"` | `"STALLED"`
- `name`: derived as `"{company_name} — {thread_subject of first commercial event}"`. Truncated to 60 chars. Never LLM-generated.

### Deal stage taxonomy

Stages are inferred deterministically from commercial event types.

| Stage | Triggered by | Definition |
|-------|-------------|------------|
| `PROSPECT` | First `DIRECT_MESSAGE` with external company containing no commercial events yet | Initial contact, no commercial signal |
| `QUALIFIED` | `MEETING_HELD` + at least one `REQUEST` or `COMMITMENT` action item | Active conversation with concrete next steps |
| `PROPOSAL` | `PROPOSAL_SENT` or `PROPOSAL_RECEIVED` | Commercial terms under discussion |
| `NEGOTIATION` | `CONTRACT_SENT` | Agreement in review |
| `CLOSED_WON` | `CONTRACT_SIGNED` or `DEAL_WON` | Terminal — deal complete |
| `CLOSED_LOST` | `DEAL_LOST` | Terminal — deal ended |

Stage transitions are **forward only** — a deal never moves backwards.

### Deal detection

```python
DEAL_TRIGGER_EVENTS = {
    'PROPOSAL_SENT', 'PROPOSAL_RECEIVED',
    'CONTRACT_SENT', 'CONTRACT_SIGNED',
    'MEETING_HELD',
    'DEAL_WON', 'DEAL_LOST',
    'INTRO_MADE',
}

EVENT_TO_STAGE = {
    'INTRO_MADE':        'PROSPECT',
    'MEETING_HELD':      'QUALIFIED',
    'PROPOSAL_SENT':     'PROPOSAL',
    'PROPOSAL_RECEIVED': 'PROPOSAL',
    'CONTRACT_SENT':     'NEGOTIATION',
    'CONTRACT_SIGNED':   'CLOSED_WON',
    'DEAL_WON':          'CLOSED_WON',
    'DEAL_LOST':         'CLOSED_LOST',
}

STAGE_ORDER = ['PROSPECT', 'QUALIFIED', 'PROPOSAL', 'NEGOTIATION', 'CLOSED_WON', 'CLOSED_LOST']

def _stage_rank(stage: str) -> int:
    try:    return STAGE_ORDER.index(stage)
    except: return -1

def detect_and_update_deals(action_graph: dict, chunk_graph: dict) -> dict:
    """
    Derive deal nodes from commercial events already in the graph.
    Creates new deals, advances stages (never regresses).
    Runs post-Phase-2, post-health-scoring.
    """
    from datetime import datetime, timezone, date
    now   = datetime.now(timezone.utc)
    today = now.date().isoformat()

    deals_by_company: dict[str, dict] = {
        d['company_id']: d for d in action_graph.get('deals', [])
    }

    for event in sorted(action_graph.get('events', []), key=lambda e: e.get('date', '')):
        event_type = event.get('type')
        if event_type not in DEAL_TRIGGER_EVENTS:
            continue

        thread_id  = event.get('thread_id')
        company_id = _company_from_thread(thread_id, chunk_graph, action_graph)
        if not company_id:
            continue

        company = next((c for c in action_graph.get('companies', []) if c['id'] == company_id), None)
        if not company:
            continue

        # Create deal if none exists for this company
        if company_id not in deals_by_company:
            thread   = next((t for t in chunk_graph['nodes']['threads'] if t['id'] == thread_id), {})
            raw_name = f"{company['name']} — {thread.get('subject', 'New Deal')}"
            deals_by_company[company_id] = {
                'id':                 stable_id('deal', company_id),
                'name':               raw_name[:60],
                'company_id':         company_id,
                'company_name':       company['name'],
                'stage':              'PROSPECT',
                'stage_entered_date': event.get('date', today),
                'days_in_stage':      0,
                'owner_id':           company.get('account_owner_id', {}).get('value'),
                'owner':              company.get('account_owner', {}).get('value'),
                'threads':            [],
                'persons_involved':   [],
                'evidence_events':    [],
                'open_action_items':  [],
                'created_date':       event.get('date', today),
                'last_activity_date': event.get('date', today),
                'outcome':            None,
            }

        deal = deals_by_company[company_id]

        # Advance stage (never regress)
        new_stage = EVENT_TO_STAGE.get(event_type)
        if new_stage and _stage_rank(new_stage) > _stage_rank(deal['stage']):
            deal['stage']              = new_stage
            deal['stage_entered_date'] = event.get('date', today)

        if deal['stage'] == 'CLOSED_WON':  deal['outcome'] = 'WON'
        elif deal['stage'] == 'CLOSED_LOST': deal['outcome'] = 'LOST'

        if event['id'] not in deal['evidence_events']:
            deal['evidence_events'].append(event['id'])
        if thread_id and thread_id not in deal['threads']:
            deal['threads'].append(thread_id)
        if event.get('date', '') > deal.get('last_activity_date', ''):
            deal['last_activity_date'] = event['date']

    # Compute days_in_stage + stalled detection (45+ days no activity)
    for deal in deals_by_company.values():
        try:
            entered = date.fromisoformat(deal['stage_entered_date'])
            deal['days_in_stage'] = (now.date() - entered).days
        except (ValueError, TypeError):
            deal['days_in_stage'] = 0
        if deal['outcome'] is None:
            try:
                last = date.fromisoformat(deal['last_activity_date'])
                if (now.date() - last).days > 45:
                    deal['outcome'] = 'STALLED'
            except (ValueError, TypeError):
                pass

    action_graph['deals'] = list(deals_by_company.values())
    return action_graph


def _company_from_thread(thread_id: str, chunk_graph: dict, action_graph: dict) -> str | None:
    """
    Resolve the external company associated with a thread.
    Returns company_id of the first non-internal company found in the thread.
    """
    thread_chunks = [c for c in chunk_graph['nodes']['chunks'] if c.get('thread_id') == thread_id]
    emails = set()
    for chunk in thread_chunks:
        for p in chunk.get('recipients', []) + chunk.get('cc', []):
            emails.add(p.get('email', '').lower())
        emails.add(chunk.get('sender', {}).get('email', '').lower())
    for company in action_graph.get('companies', []):
        if company.get('type') == 'internal':
            continue
        domain = company.get('domain', '').lower()
        if domain and any(e.endswith('@' + domain) for e in emails):
            return company['id']
    return None
```

### Deal storage

Deals are stored as a top-level `deals` array in the action items graph,
alongside `persons`, `companies`, `events`, and `action_items`.

### Where it runs in the orchestrator

```python
# After health scoring, before reporting:
action_graph = detect_and_update_deals(action_graph, chunk_graph)
safe_overwrite(ACTION_GRAPH_PATH, action_graph)
```

### Deal queries via CRM Query Interface

```
Show all active deals
What stage is Acme Corp in
Which deals have been stalled for over 30 days
Show deals in Proposal stage
Mark Acme Corp deal as lost
```

---

## Module: Dark Period Detector

Detects accounts with open deals where no email has been sent or received for
21+ days. This is a relationship-level signal — distinct from action item
inactivity. An account can have all action items active and still be going dark
if the conversation has stalled. Gong and Clari charge separately for this signal.
The skill computes it for free from the chunk graph.

### Configuration

```python
DARK_PERIOD_THRESHOLD_DAYS  = 21    # days without email activity on an open deal
DARK_PERIOD_EXEMPT_STAGES   = {'Closed Won', 'Closed Lost'}   # terminal stages — skip
```

### Detector function

```python
def detect_dark_period_accounts(action_graph: dict,
                                 chunk_graph: dict,
                                 threshold_days: int = DARK_PERIOD_THRESHOLD_DAYS) -> dict:
    """
    For every deal that is not in a terminal stage, check whether any email has
    been sent or received in threads linked to that deal within threshold_days.

    If not: set deal.dark_period_alert = True, update days_since_last_activity.
    If yes: ensure dark_period_alert = False.

    Also fires a DARK_PERIOD event into action_graph['events'] the first time
    a deal crosses the threshold (idempotent — one event per crossing, not per run).
    """
    now    = datetime.now(timezone.utc)
    cutoff = now - timedelta(days=threshold_days)

    # Build a lookup: thread_id → most recent chunk date
    thread_last_activity = {}
    for chunk in chunk_graph['nodes'].get('chunks', []):
        tid  = chunk.get('thread_id')
        date = chunk.get('date', '')
        if tid and date:
            if tid not in thread_last_activity or date > thread_last_activity[tid]:
                thread_last_activity[tid] = date

    deals = action_graph.get('deals', [])
    for deal in deals:
        # Skip terminal stages
        if deal.get('stage_name') in DARK_PERIOD_EXEMPT_STAGES:
            deal['dark_period_alert']        = False
            deal['days_since_last_activity'] = None
            continue

        # Find the most recent activity across all threads linked to this deal
        deal_threads = deal.get('threads', [])
        last_dates   = [thread_last_activity[tid]
                        for tid in deal_threads if tid in thread_last_activity]

        if not last_dates:
            # No threads at all — treat as dark from first_contact_date
            last_activity_str = deal.get('first_contact_date', '')
        else:
            last_activity_str = max(last_dates)

        deal['last_activity_date'] = last_activity_str

        try:
            last_dt = datetime.fromisoformat(last_activity_str)
            if last_dt.tzinfo is None:
                last_dt = last_dt.replace(tzinfo=timezone.utc)
        except (ValueError, TypeError):
            continue

        days_silent = (now - last_dt).days
        deal['days_since_last_activity'] = days_silent

        was_dark = deal.get('dark_period_alert', False)

        if days_silent >= threshold_days:
            deal['dark_period_alert'] = True

            # Fire a DARK_PERIOD event only on first crossing (not already present)
            existing_dp_events = [
                e for e in action_graph.get('events', [])
                if e.get('type') == 'DARK_PERIOD'
                and e.get('deal_id') == deal['id']
            ]
            if not was_dark and not existing_dp_events:
                action_graph.setdefault('events', []).append({
                    'id':          stable_id('event', 'DARK_PERIOD', deal['id'], last_activity_str),
                    'type':        'DARK_PERIOD',
                    'deal_id':     deal['id'],
                    'company_id':  deal['company_id'],
                    'company_name': deal.get('company_name', ''),
                    'description': (f"No email activity with {deal.get('company_name','')} "
                                    f"for {days_silent} days on deal in stage "
                                    f"'{deal.get('stage_name','')}'."),
                    'date':        now.date().isoformat(),
                    'days_silent': days_silent,
                    'threshold':   threshold_days,
                })
        else:
            deal['dark_period_alert'] = False

    return action_graph
```

### How dark periods surface in the report

Dark period accounts appear as a dedicated block in the dialog report, above
the normal action items list, so they are never buried:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔇 DARK PERIOD ALERTS  (no email activity ≥21 days)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Acme Corp      [Proposal]    28 days silent   $120k @ 65%
  Beta Systems   [Qualified]   22 days silent   amount unknown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Where it runs in the orchestrator

```python
action_graph = update_all_health_scores(action_graph, chunk_graph)
action_graph = detect_overdue_commitments(action_graph, chunk_graph)
action_graph = detect_inactive_items(action_graph)
action_graph = detect_dark_period_accounts(action_graph, chunk_graph)  # ← new
safe_overwrite(ACTION_GRAPH_PATH, action_graph)
```

---

## Module: Meeting-as-Signal

When a `CALENDAR_INVITE` email is detected for a company with an open deal,
it is a high-value engagement signal — a meeting is being scheduled, which
typically precedes either a decision or a stall. The pipeline stamps the deal
and opens a 7-day follow-up detection window. If no follow-up email appears
within that window, a FOLLOW_UP action item is automatically created.

### Detection logic

```python
MEETING_FOLLOWUP_WINDOW_DAYS = 7

def process_meeting_signals(action_graph: dict,
                             chunk_graph: dict) -> dict:
    """
    Scan all chunks with content_type == 'CALENDAR_INVITE'.
    For each invite linked to a company with an open deal:
      1. Stamp deal.last_meeting_scheduled with the invite date.
      2. Set deal.meeting_followup_due = invite_date + 7 days.
      3. Check whether any chunk from the same company threads has a date
         after invite_date (follow-up email evidence).
      4. Set deal.meeting_followup_status accordingly.
      5. If status == 'overdue' and no FOLLOW_UP action item exists for this
         meeting, create one.
    """
    now    = datetime.now(timezone.utc)
    chunks = chunk_graph['nodes'].get('chunks', [])

    # Build company_id → deal lookup (open deals only)
    open_deals = {
        d['company_id']: d for d in action_graph.get('deals', [])
        if d.get('stage_name') not in DARK_PERIOD_EXEMPT_STAGES
    }

    # Build thread_id → company_id lookup via edges (WORKS_AT + RECEIVED_BY)
    thread_company_map = _build_thread_company_map(chunk_graph)

    # Find all CALENDAR_INVITE chunks, sorted newest first
    invite_chunks = sorted(
        [c for c in chunks if c.get('content_type') == 'CALENDAR_INVITE'],
        key=lambda c: c.get('date', ''),
        reverse=True,
    )

    for invite in invite_chunks:
        company_id = thread_company_map.get(invite.get('thread_id'))
        if not company_id or company_id not in open_deals:
            continue

        deal        = open_deals[company_id]
        invite_date = invite.get('date', '')[:10]  # ISO date only

        # Only update if this is the most recent invite for this deal
        existing = deal.get('last_meeting_scheduled', '')
        if existing and invite_date <= existing:
            continue

        deal['last_meeting_scheduled'] = invite_date

        try:
            invite_dt = datetime.fromisoformat(invite_date)
            if invite_dt.tzinfo is None:
                invite_dt = invite_dt.replace(tzinfo=timezone.utc)
        except ValueError:
            continue

        followup_due    = invite_dt + timedelta(days=MEETING_FOLLOWUP_WINDOW_DAYS)
        deal['meeting_followup_due'] = followup_due.date().isoformat()

        # Check for any email activity in this deal's threads AFTER the invite date
        deal_threads   = set(deal.get('threads', []))
        post_invite_chunks = [
            c for c in chunks
            if c.get('thread_id') in deal_threads
            and c.get('date', '') > invite_date
            and c.get('content_type') not in ('CALENDAR_INVITE', 'AUTO_REPLY', 'SYSTEM_NOTIFICATION')
        ]

        if post_invite_chunks:
            deal['meeting_followup_status'] = 'received'
        elif now > followup_due:
            deal['meeting_followup_status'] = 'overdue'

            # Create FOLLOW_UP action item if not already present
            followup_id = stable_id('action', 'MEETING_FOLLOWUP', deal['id'], invite_date)
            existing_followups = [
                ai for ai in action_graph.get('action_items', [])
                if ai.get('id') == followup_id
            ]
            if not existing_followups:
                action_graph.setdefault('action_items', []).append({
                    'id':           followup_id,
                    'type':         'FOLLOW_UP',
                    'status':       'OPEN',
                    'owner':        deal.get('company_name', 'Unknown'),
                    'description':  (f"No follow-up received after meeting scheduled "
                                     f"with {deal.get('company_name','')} on {invite_date}. "
                                     f"Follow-up due by {followup_due.date().isoformat()}."),
                    'deal_id':      deal['id'],
                    'company_id':   company_id,
                    'source':       'meeting_signal_detector',
                    'first_seen':   now.isoformat(),
                    'last_updated': now.isoformat(),
                    'status_history': [
                        {'status': 'OPEN', 'date': now.isoformat(), 'set_by': 'meeting_signal_detector'}
                    ],
                })
        else:
            deal['meeting_followup_status'] = 'pending'

    return action_graph
```

**Note:** `_build_thread_company_map(chunk_graph)` resolves thread → company_id
by traversing `SENT_BY` / `RECEIVED_BY` → person → `WORKS_AT` → company edges
and returning the external company (non-internal) for each thread.

### Where it runs in the orchestrator

```python
action_graph = update_all_health_scores(action_graph, chunk_graph)
action_graph = detect_overdue_commitments(action_graph, chunk_graph)
action_graph = detect_inactive_items(action_graph)
action_graph = detect_dark_period_accounts(action_graph, chunk_graph)
action_graph = process_meeting_signals(action_graph, chunk_graph)       # ← new
safe_overwrite(ACTION_GRAPH_PATH, action_graph)
```

---

## Step 3 — Reporting

Results are printed to the dialog only. No files are created for the report.
The three graph/checkpoint files are still written (they are data, not reports).

### Dialog Report (printed to conversation after every run)

Format for first run and incremental runs:

```
╔══════════════════════════════════════════════════════════════╗
║  AICRM OPEN ACTION ITEMS  — [date] UTC                      ║
║  [N] total items | [N] OPEN | [N] OVERDUE | [N] SCHEDULED  ║
╚══════════════════════════════════════════════════════════════╝

⚠  OVERDUE ([N])
   [owner] → [description]
   Thread: [subject] | Since: [date]
   Evidence: "[quote]"

🔴  OPEN — AWAITING THEIR ACTION ([N])
   [owner] → [description]
   Thread: [subject]
   Evidence: "[quote]"

🟡  OPEN — YOUR ACTION ([N])
   [owner: internal XYZ person] → [description]
   [BLOCKED BY: item description]

🟢  SCHEDULED ([N])
   [owner] — [description] @ [time/date]

✅  NEWLY RESOLVED THIS RUN ([N])   ← incremental runs only
   [owner] → [description] → now DONE

── New since last run: [N] new items ─────────────────────────
```

Classify items by owner's `company_type` to separate:
- "YOUR ACTION" = items owned by internal (`company_type: internal`) persons
- "AWAITING THEIR ACTION" = items owned by external persons

### Dialog Rendering (pseudo-code)

```python
def print_dialog_report(action_graph: dict, newly_resolved: list,
                         new_item_ids: list | None = None):
    """
    Print the post-run dialog report.

    Structure (every run):
      ⚡ PRIORITY 0 — items needing immediate attention (OVERDUE + BLOCKED)
      📋 PRIORITY 1 — all other active items (OPEN / IN_PROGRESS / SCHEDULED)

    Additional section (incremental runs only, when there are changes):
      🔄 WHAT CHANGED — newly resolved + new items added this run

    Dashboard link is printed by the caller (run_full / run_incremental),
    not here — so it always appears as the final line of the response.
    """
    items   = action_graph['action_items']
    persons = {p['id']: p for p in action_graph['persons']}

    # ── Priority classification ───────────────────────────────────────────────
    # P0: overdue commitments, and any item blocked by an unresolved dependency
    p0 = [i for i in items
          if i['status'] == 'COMMITMENT_OVERDUE'
          or (i.get('blocked_by') and i['status'] not in ('DONE', 'TRIGGERED', 'DISMISSED'))]

    p0_ids = {i['id'] for i in p0}

    # P1: all other active items (excludes terminal statuses and P0)
    TERMINAL = {'DONE', 'TRIGGERED', 'DISMISSED', 'COMMITMENT_OVERDUE'}
    p1 = [i for i in items
          if i['id'] not in p0_ids
          and i['status'] not in TERMINAL]

    def fmt_item(item) -> str:
        owner = item.get('owner', '—')
        desc  = item.get('description', '')
        subj  = item.get('thread_subject', '')
        due   = f"  Due: {item['due_date']}" if item.get('due_date') else ''
        line  = f"  [{item['status']}] {owner} → {desc}"
        if subj:
            line += f"\n  Thread: {subj}"
        if due:
            line += due
        return line

    # ── Print ─────────────────────────────────────────────────────────────────
    date_str = datetime.now(timezone.utc).strftime('%b %-d, %Y')
    print(f"\n{'─'*55}")
    print(f"  AICRM  ·  {date_str}")
    print(f"{'─'*55}")

    if p0:
        print(f"\n⚡  PRIORITY 0 — Immediate attention  ({len(p0)})")
        for i in p0:
            print(fmt_item(i))
    else:
        print(f"\n⚡  PRIORITY 0 — None")

    if p1:
        print(f"\n📋  PRIORITY 1 — Active  ({len(p1)})")
        for i in p1:
            print(fmt_item(i))
    else:
        print(f"\n📋  PRIORITY 1 — None")

    # What changed — only shown on incremental runs when there is something to report
    new_item_ids = new_item_ids or []
    new_items = [i for i in items if i['id'] in set(new_item_ids)]
    if newly_resolved or new_items:
        print(f"\n🔄  WHAT CHANGED THIS RUN")
        if new_items:
            print(f"  New ({len(new_items)})")
            for i in new_items:
                print(f"    + [{i['status']}] {i.get('owner','—')} → {i.get('description','')}")
        if newly_resolved:
            print(f"  Resolved ({len(newly_resolved)})")
            for i in newly_resolved:
                print(f"    ✓ {i.get('owner','—')} → {i.get('description','')}")

    # ── Suggested deep-dive questions ─────────────────────────────────────────
    # Always printed at the end of every run report.
    # Questions are grouped by theme. Claude should print them verbatim so the
    # user knows exactly what they can ask next to explore the data further.
    print(f"\n{'─'*55}")
    print(f"  💬  Things you can ask me")
    print(f"{'─'*55}")
    print(f"""
  Action items & commitments
    · What commitments are overdue and who owns them?
    · Show me everything blocked and what is blocking it
    · What has [person] committed to but not yet delivered?
    · Which open items have no due date set?

  Deals & pipeline
    · What deals are in the pipeline and what stage are they at?
    · Which deals have gone dark — no activity in 30+ days?
    · Give me a weighted pipeline value breakdown by forecast category
    · Which deals are stalled and why?
    · What is the probability-weighted value of my pipeline?

  Accounts & relationships
    · What is the relationship health score for [company]?
    · Who are the key contacts at [company] and what are their roles?
    · Which accounts have I not had any activity with in the last 30 days?
    · Who is the economic buyer / champion on [deal]?

  People
    · What is [person] responsible for across all threads?
    · Who are the most active contacts in my mailbox?
    · Show me all internal people and their open commitments

  Threads & communication
    · Summarise all communication with [company]
    · What is the latest status on [thread subject]?
    · Which threads have the most unresolved action items?
    · Show me all unanswered questions outstanding

  Cross-cutting
    · What are my top 3 priorities right now?
    · Which accounts are most at risk this week?
    · What should I focus on today?
    · Show me everything due in the next 7 days
""")

# No file is written. The dialog output is the complete report.
# The dashboard link is printed by the caller immediately after this function.
# ⛔ DO NOT call present_files after this function or anywhere in the pipeline.
# The dashboard computer:// link printed in Step 5 is the only file reference shown to the user.
# present_files may only be called if the user explicitly requests a specific file by name in their prompt.
```

---

## Step 4 — On-Demand Invocation Only

### No Automatic Scheduling

This skill runs **only when the user explicitly invokes it**. There is no background
scheduler, no cron job, and no automatic 12-hour cycle. Every run is triggered
by the user saying something like "check my emails", "update my CRM", or
"what are my open action items".

This keeps the user in full control of when their mailbox is accessed and when
graphs are updated.

### Run Routing Logic

The orchestrator determines whether this is a first run or a subsequent run by
checking for an existing checkpoint — no flags required:

```python
def main():
    checkpoint   = read_checkpoint()
    is_first_run = checkpoint.get('last_run_utc') is None

    if is_first_run:
        print("First run — building graphs from full mailbox.")
        run_full(checkpoint)
    else:
        print(f"Subsequent run — checking for new emails since {checkpoint['last_run_utc']}.")
        run_incremental(checkpoint)
```

---

## Orchestrator — run_pipeline_v2.py

```python
"""
Ambient CRM v2 — Full mailbox pipeline orchestrator.
No third-party modules required. Stdlib only.

Runs on-demand only — invoked by the user, never on a schedule.
The orchestrator automatically detects first run vs. subsequent run
by checking for an existing checkpoint file.

Usage:
  python3 run_pipeline_v2.py
"""
import json, os
from datetime import datetime, timezone

CHUNK_GRAPH_PATH  = "aicrm_chunk_graph.json"
ACTION_GRAPH_PATH = "aicrm_action_items_graph.json"
CHECKPOINT_PATH   = "aicrm_checkpoint.json"
OUTPUT_DIR        = "."

import shutil

def safe_overwrite(path: str, data: dict):
    """
    Write JSON data to path, preserving the previous version as path.bak.
    Only the action items graph uses this — it is the only file where a
    bad merge run could silently corrupt previously-correct statuses.
    """
    if os.path.exists(path):
        shutil.copy2(path, path + '.bak')
    with open(path, 'w', encoding='utf-8') as f:
        json.dump(data, f, indent=2, ensure_ascii=False)


MAX_STATUS_HISTORY_ENTRIES = 50   # cap per item — oldest entries pruned when exceeded

# ---------------------------------------------------------------------------
# TRAINING DATA COLLECTION
# ---------------------------------------------------------------------------
# Passive behavioral signal capture. No user interaction required.
# Every extraction creates a training_example row. Behavioral events
# (completions, dismissals, field edits) populate the label fields later.
# ---------------------------------------------------------------------------

TRAINING_EXAMPLES_PATH = os.path.join(WORKSPACE_DIR, 'aicrm_training_examples.jsonl')

QUALITY_TIER_BY_SOURCE = {
    'COMPLETED_BY_OWNER':       'gold',
    'COMPLETED_BY_OTHER':       'gold',
    'DISMISSED_EARLY':          'gold',
    'DISMISSED_LATE':           'silver',
    'EDITED_OWNER':             'gold',
    'EDITED_TYPE':              'gold',
    'EDITED_DESCRIPTION':       'gold',
    'EDITED_DATE':              'gold',
    'MANUAL_ADD_SAME_THREAD':   'gold',
    'AUTO_SURVIVED_30D':        'silver',
    'AUTO_SURVIVED_60D_EXCLUDE':'exclude',
    'THREAD_EXCLUDED':          'exclude',
}

def record_training_example(item: dict,
                             body_tokenized: str,
                             structural_metadata: dict) -> None:
    """
    Write an unlabeled training example at extraction time.
    Called immediately after Phase 2 produces each action item.
    The tokenized input and original extraction are captured here.
    Labels are added later by record_training_label().

    body_tokenized and structural_metadata are what was sent to the LLM —
    they must be passed from the extraction call site, not reconstructed.
    They contain tokens not real PII values.
    """
    import json, uuid
    example = {
        'example_id':               str(uuid.uuid4()),
        'action_item_id':           item.get('id'),
        'chunk_id':                 item.get('source_chunk_id'),
        'thread_id':                item.get('thread_id'),
        'created_at':               datetime.now(timezone.utc).isoformat(),
        'labeled_at':               None,
        # Input — tokenized, no PII
        'body_tokenized':           body_tokenized,
        'structural_metadata':      structural_metadata,
        # Sonnet's original output — tokenized, immutable after creation
        'extraction_json_original': {
            'type':              item.get('type'),
            'owner':             item.get('owner'),       # token at extraction time
            'description':       item.get('description'),
            'commitment_date':   item.get('commitment_date'),
            'extraction_confidence': item.get('extraction_confidence'),
        },
        # Label fields — populated by record_training_label()
        'label_source':             None,
        'quality_tier':             None,
        'labeled_by_owner':         None,
        # Correction diff — populated for EDITED_* label sources
        'extraction_json_final':    None,
        'correction_field':         None,
        'correction_delta':         None,
        # False negative — populated for MANUAL_ADD_SAME_THREAD
        'manually_added_item':      None,
        'missed_extraction':        False,
        # Consent
        'opted_out':                False,
        'exportable':               False,   # set to True by record_training_label()
    }
    with open(TRAINING_EXAMPLES_PATH, 'a') as f:
        f.write(json.dumps(example) + '\n')


def record_training_label(action_item_id: str,
                           action_graph: dict,
                           label_source: str,
                           correction_field: str = None,
                           correction_delta: dict = None,
                           labeled_by_owner: bool = None) -> None:
    """
    Update the training example for action_item_id with a behavioral label.
    Reads the JSONL file, updates the matching row, rewrites the file.
    Called from close_item, dismiss_item, and field-edit handlers.

    label_source must be a key in QUALITY_TIER_BY_SOURCE.
    """
    import json
    if not os.path.exists(TRAINING_EXAMPLES_PATH):
        return

    now = datetime.now(timezone.utc).isoformat()
    updated_lines = []
    found = False

    with open(TRAINING_EXAMPLES_PATH, 'r') as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            ex = json.loads(line)
            if ex.get('action_item_id') == action_item_id and ex.get('label_source') is None:
                ex['label_source']     = label_source
                ex['quality_tier']     = QUALITY_TIER_BY_SOURCE.get(label_source, 'exclude')
                ex['labeled_at']       = now
                ex['labeled_by_owner'] = labeled_by_owner
                ex['exportable']       = (
                    not ex.get('opted_out', False) and
                    ex['quality_tier'] != 'exclude'
                )
                if correction_field:
                    ex['correction_field'] = correction_field
                if correction_delta:
                    ex['correction_delta'] = correction_delta
                found = True
            updated_lines.append(json.dumps(ex))

    if found:
        with open(TRAINING_EXAMPLES_PATH, 'w') as f:
            f.write('\n'.join(updated_lines) + '\n')


def record_training_correction(action_item_id: str,
                                field: str,
                                old_value,
                                new_value) -> None:
    """
    Convenience wrapper for field-edit corrections.
    Maps edited field to label_source and writes diff.

    field: 'owner' | 'type' | 'description' | 'commitment_date'
    old_value / new_value: token-form values (not re-hydrated real names)
    """
    source_map = {
        'owner':           'EDITED_OWNER',
        'type':            'EDITED_TYPE',
        'description':     'EDITED_DESCRIPTION',
        'commitment_date': 'EDITED_DATE',
    }
    label_source = source_map.get(field, 'EDITED_DESCRIPTION')
    record_training_label(
        action_item_id=action_item_id,
        action_graph={},   # not needed for label update
        label_source=label_source,
        correction_field=field,
        correction_delta={'from': old_value, 'to': new_value},
    )


def _hours_since(iso_timestamp: str) -> float:
    """Return hours elapsed since iso_timestamp. Returns 999 if unparseable."""
    try:
        from datetime import datetime, timezone
        then = datetime.fromisoformat(iso_timestamp.replace('Z', '+00:00'))
        now  = datetime.now(timezone.utc)
        return (now - then).total_seconds() / 3600
    except Exception:
        return 999.0


def _current_user_name() -> str:
    """
    Return the name of the user currently operating the session.
    Used to determine whether the completing user is the attributed owner.
    Falls back to empty string if not determinable — caller treats as non-owner.
    In v1 this is set from the active mailbox owner in the session context.
    """
    return os.environ.get('AICRM_CURRENT_USER', '')

# ---------------------------------------------------------------------------

def record_status_change(item: dict,
                          new_status: str,
                          set_by: str,
                          now: datetime = None) -> dict:
    """
    Update an action item's status and append an entry to its status_history.
    Always use this function instead of setting item['status'] directly —
    it guarantees the history log stays in sync with the current status.

    set_by values:
      "pipeline"            — Phase 2 LLM extraction or merge/diff
      "detector"            — asymmetric commitment detector
      "inactivity_detector" — inactivity detector
      "trust_layer"         — confidence-based routing
      "meeting_signal_detector" — meeting-as-signal module
      "manual"              — user via manual command

    Returns the mutated item.
    """
    if now is None:
        from datetime import datetime, timezone
        now = datetime.now(timezone.utc)

    if 'status_history' not in item:
        # Backfill: create history from existing status + first_seen if missing
        item['status_history'] = [{
            'status':  item.get('status', 'OPEN'),
            'date':    item.get('first_seen', now.isoformat()),
            'set_by':  'pipeline',
        }]

    item['status']       = new_status
    item['last_updated'] = now.isoformat()
    item['status_history'].append({
        'status': new_status,
        'date':   now.isoformat(),
        'set_by': set_by,
    })

    # Enforce size cap — retain first entry (creation) + most recent N-1 entries
    # so the audit trail is never fully lost
    history = item['status_history']
    if len(history) > MAX_STATUS_HISTORY_ENTRIES:
        item['status_history'] = [history[0]] + history[-(MAX_STATUS_HISTORY_ENTRIES - 1):]

    return item

def run_full(checkpoint):
    print("=== AICRM — FIRST RUN ===\n")

    # Step 1: Check for new mails (all emails are new on first run)
    print("Step 1  Connecting mailbox and fetching emails...")
    print(f"        Limit: last {FIRST_RUN_MAX_DAYS} days · max {FIRST_RUN_MAX_EMAILS} emails (most recent first)")
    print("        Skipping: Junk, Trash, Spam, Deleted Items")
    print("        To override: pass max_days= or max_emails= when invoking")
    connector  = discover_email_connector()
    identity   = load_or_resolve_identity(connector)
    print(f"        Identity: {identity['owner_name']}  ({identity['company_mode']} · {identity['owner_domain']})")
    all_emails = fetch_first_run_emails(connector)
    print(f"        Fetched {len(all_emails)} emails")

    # Step 2: Update chunk graph (build from scratch on first run)
    print("\nStep 2  Building chunk graph...")
    threads = group_into_threads(all_emails)
    graph   = build_chunk_graph_from_threads(threads)
    with open(CHUNK_GRAPH_PATH, 'w') as f:
        json.dump(graph, f, indent=2, ensure_ascii=False)

    # Step 3: Update action items (extract from scratch on first run)
    n_threads = graph['meta']['thread_count']
    SECS_PER_THREAD = 3  # conservative estimate: LLM call + validation + merge
    est_secs  = n_threads * SECS_PER_THREAD
    est_str   = f"~{est_secs // 60} min {est_secs % 60}s" if est_secs >= 60 else f"~{est_secs}s"
    print(f"\nStep 3  Extracting action items...")
    print(f"        Threads to process : {n_threads}")
    print(f"        Estimated duration : {est_str}  ({SECS_PER_THREAD}s per thread, LLM call)")
    print(f"        Progress below — updates every 10 threads")
    action_graph = build_action_items_graph(graph)
    safe_overwrite(ACTION_GRAPH_PATH, action_graph)  # backs up previous .bak if exists

    # Step 4: Report (dialog only — no file written)
    print("\nStep 4  Reporting open action items...")
    print_dialog_report(action_graph, newly_resolved=[])

    # Step 5: Write checkpoint + generate dashboard
    all_msg_ids = [e.get('message_id','') for e in all_emails]
    write_checkpoint(checkpoint, all_msg_ids, list(threads.keys()), 'first_run')
    handle_generate_dashboard(action_graph, graph, silent=True)
    print(f"\n  Dashboard → computer://{DASHBOARD_PATH}")
    # ⛔ DO NOT call present_files here or after this point.
    # The dashboard link above is the only file reference returned to the user.
    # present_files must not be called for any output file unless the user
    # explicitly requests a specific file by name in their prompt.


def run_incremental(checkpoint):
    print(f"=== AICRM — SUBSEQUENT RUN ===")
    print(f"    Last run: {checkpoint.get('last_run_utc', 'unknown')}\n")

    # Step 1: Check for new mails
    print("Step 1  Checking for new emails...")
    print("        (Skipping Junk, Trash, Spam, Deleted Items)")
    connector  = discover_email_connector()
    identity   = load_or_resolve_identity(connector)
    new_emails = fetch_new_emails(connector, checkpoint)
    print(f"        {len(new_emails)} new email(s) since last run")

    if not new_emails:
        print("\n        Mailbox is up to date — no new emails to process.")
        write_checkpoint(checkpoint, [], [], 'incremental')
        return

    # Step 2: Update chunk graph
    print("\nStep 2  Updating chunk graph...")
    with open(CHUNK_GRAPH_PATH) as f:
        chunk_graph = json.load(f)
    chunk_graph, threads_updated = extend_chunk_graph(chunk_graph, new_emails)
    with open(CHUNK_GRAPH_PATH, 'w') as f:
        json.dump(chunk_graph, f, indent=2, ensure_ascii=False)

    # Step 3: Update action items
    n_threads = len(threads_updated)
    SECS_PER_THREAD = 3
    est_secs  = n_threads * SECS_PER_THREAD
    est_str   = f"~{est_secs // 60} min {est_secs % 60}s" if est_secs >= 60 else f"~{est_secs}s"
    print(f"\nStep 3  Updating action items...")
    print(f"        Threads to process : {n_threads}")
    print(f"        Estimated duration : {est_str}  ({SECS_PER_THREAD}s per thread, LLM call)")
    print(f"        Progress below — updates every 10 threads")
    with open(ACTION_GRAPH_PATH) as f:
        action_graph = json.load(f)
    prev_ids      = {ai['id'] for ai in action_graph['action_items']}
    prev_statuses = {ai['id']: ai['status'] for ai in action_graph['action_items']}
    action_graph  = run_phase2_incremental(chunk_graph, action_graph, threads_updated)
    newly_resolved = [
        ai for ai in action_graph['action_items']
        if ai['status'] == 'DONE' and prev_statuses.get(ai['id']) != 'DONE'
    ]
    new_item_ids = [ai['id'] for ai in action_graph['action_items']
                    if ai['id'] not in prev_ids]
    safe_overwrite(ACTION_GRAPH_PATH, action_graph)  # previous version saved to .bak

    # Step 4: Report (dialog only — no file written)
    print("\nStep 4  Reporting changes since last run...")
    print_dialog_report(action_graph, newly_resolved, new_item_ids=new_item_ids)

    # Step 5: Update checkpoint + generate dashboard
    new_msg_ids = [e.get('message_id','') for e in new_emails]
    write_checkpoint(checkpoint, new_msg_ids, threads_updated, 'incremental')
    handle_generate_dashboard(action_graph, chunk_graph, silent=True)
    print(f"\n  Dashboard → computer://{DASHBOARD_PATH}")
    # ⛔ DO NOT call present_files here or after this point.
    # The dashboard link above is the only file reference returned to the user.
    # present_files must not be called for any output file unless the user
    # explicitly requests a specific file by name in their prompt.


if __name__ == "__main__":
    checkpoint   = read_checkpoint()
    is_first_run = checkpoint.get('last_run_utc') is None
    if is_first_run:
        run_full(checkpoint)
    else:
        run_incremental(checkpoint)
```

---

## Quality Characteristics

### What this architecture captures that rule-based pipelines miss

| Item type | Rule-based (spaCy) | This architecture |
|-----------|-------------------|-------------------|
| Explicit commitments | Partial (keyword match) | Full — evidence quoted |
| Implicit assignments ("@Name will lead") | Missed | ASSIGNMENT type |
| Unanswered questions in newest chunk | Missed | UNANSWERED_QUESTION type |
| Non-English content (Ukrainian, Arabic, etc.) | Fails | Handled natively |
| Conditional commitments | Missed | Two linked items + BLOCKS edge |
| Overdue commitments | Missed | COMMITMENT_OVERDUE |
| Cross-thread person links | Not tracked | CROSS_REF edges |
| Dependency chains | Not tracked | BLOCKS / BLOCKED_BY |
| System-triggered actions (CRM, ASDLC) | Missed | SYSTEM_TRIGGER type |
| Partial delivery | Binary done/not done | PARTIAL status |
| Status transitions over time | Not tracked | `first_seen` / `last_updated` + DONE status |

---

## Running

```bash
# Single command — always the same. Orchestrator auto-detects first vs. subsequent run.
python3 run_pipeline_v2.py
```

The script detects whether a checkpoint exists:
- **No checkpoint** → first run, builds all graphs from the full mailbox
- **Checkpoint found** → subsequent run, checks for new emails only

Runs only when the user invokes the skill. No scheduling, no background process.

Expected output files (three data files only — no report file):
```
aicrm_chunk_graph.json             ← Step 2 output (required)
aicrm_action_items_graph.json      ← Step 3 output (required)
aicrm_action_items_graph.json.bak  ← previous action items graph, one rollback level
aicrm_checkpoint.json              ← Step 5 run state (required)
```

The action items summary is printed directly to the dialog. No report file is created.
The `.bak` is written before every overwrite of the action items graph — restoring it
is a manual copy: `cp aicrm_action_items_graph.json.bak aicrm_action_items_graph.json`.

No `pip install` required. All modules used (`json`, `hashlib`, `re`,
`email`, `collections`, `datetime`, `os`) are Python stdlib.

---

---

## Module: Deal Stage Identification (Optional, User-Activated)

This module is **opt-in**. It is dormant until the user activates it by providing
their stage definitions. Once active, it runs as a post-Phase-2 pass on the action
items graph, inferring the current deal stage for each tracked company from the
events and action items already extracted.

### Activation

The user activates this module by invoking the skill with a specific trigger phrase
and providing stage definitions in plain English:

> **Trigger:** "define my deal stages" / "set up deal stages" / "activate deal tracking"

When triggered, Claude asks the user to describe their stages. The user provides
them in natural language, for example:

```
1. Prospecting — we've identified the company and made first contact
2. Qualified — we've confirmed budget, authority, need, and timeline
3. Proposal — we've sent a commercial proposal or SOW
4. Negotiation — they've responded to the proposal; we're aligning on terms
5. Closed Won — contract signed or PO received
6. Closed Lost — they've disengaged, chosen a competitor, or said no
```

Claude converts this into a `deal_stages.json` file saved alongside the other
graph files. This file is the persistent stage schema — it is not regenerated on
each run, only updated when the user re-invokes the activation trigger.

```json
{
  "version": "1.0",
  "last_updated": "2026-03-06T10:00:00Z",
  "stages": [
    {
      "id": "stage_01",
      "name": "Prospecting",
      "position": 1,
      "description": "we've identified the company and made first contact",
      "entry_signals": {
        "events": ["INTRO_MADE", "REFERRAL_MADE"],
        "action_item_types": ["MEETING", "INTRODUCTION"],
        "keywords": ["introduction", "first contact", "reaching out", "following up on our introduction"]
      },
      "exit_signals": {
        "events": [],
        "keywords": ["budget confirmed", "good fit", "timeline", "qualified"]
      }
    },
    {
      "id": "stage_03",
      "name": "Proposal",
      "position": 3,
      "description": "we've sent a commercial proposal or SOW",
      "entry_signals": {
        "events": ["PROPOSAL_SENT"],
        "action_item_types": [],
        "keywords": ["proposal", "quote", "commercial", "pricing", "SOW"]
      },
      "exit_signals": {
        "events": ["CONTRACT_SIGNED", "DEAL_WON", "DEAL_LOST"],
        "keywords": ["moving forward", "signed", "going a different direction"]
      }
    }
  ]
}
```

### Deal Node Schema

When the module is active, a `deals` section is added to the action items graph.
Each deal tracks one commercial engagement per company:

```json
{
  "id": "deal_b71d3a0f12",
  "company_id": "company_b71d3a0f12",
  "company_name": "Acme Corp",
  "stage_id": "stage_03",
  "stage_name": "Proposal",
  "stage_confidence": 0.91,
  "stage_evidence": "PROPOSAL_SENT event on 2026-02-15 in thread 'MSA and SOW for Phase 1'",
  "first_contact_date": "2026-01-10",
  "last_activity_date": "2026-02-24",
  "days_in_current_stage": 9,

  "amount":            {"value": 120000, "source": "manual", "date": "2026-02-15T10:00:00Z", "manual_override": true},
  "close_date":        {"value": "2026-04-30", "source": "manual", "date": "2026-02-15T10:00:00Z", "manual_override": true},
  "probability":       {"value": 0.65, "source": "pipeline", "date": "2026-02-24T22:00:00Z", "manual_override": false},
  "forecast_category": {"value": "best_case", "source": "manual", "date": "2026-02-20T09:00:00Z", "manual_override": true},
  "weighted_value":    78000,

  "last_meeting_scheduled": "2026-02-20",
  "meeting_followup_due":   "2026-02-27",
  "meeting_followup_status": "received",

  "dark_period_alert": false,
  "days_since_last_activity": 3,

  "contact_roles": [
    {"person_id": "person_a3f9c21d7e", "person_name": "Emily Blunt",  "role": "champion"},
    {"person_id": "person_f8c3a21b44", "person_name": "Gene Dou",     "role": "economic_buyer"},
    {"person_id": "person_c2d5e81a3f", "person_name": "Lisa Park",    "role": "technical_buyer"},
    {"person_id": "person_b9f1d43c7a", "person_name": "Mark Reeves",  "role": "unknown"}
  ],

  "open_action_items": ["action_7f3a1d2b9c", "action_3d8e2f1a7b"],
  "events": ["event_5f3a1d2b9c"],
  "threads": ["thread_af75f00118", "thread_81c29afe0f"]
}
```

**Deal economics field notes:**
- `amount`: deal value in the user's base currency. Not inferred — set via manual command or CRM Query Interface. `null` until provided.
- `close_date`: expected close date. Not inferred from email alone. Set manually. Pipeline may suggest a date if explicit deadline language appears in a COMMITMENT item (e.g. "we need to sign by end of Q1"), but always stores `source: "pipeline"` and `manual_override: false` — user must confirm to promote to `manual_override: true`.
- `probability`: 0.0–1.0. Default values by stage (defined in `deal_stages.json` as `default_probability` per stage). User can override per deal. Pipeline recalculates default when stage changes.
- `forecast_category`: `"commit"` | `"best_case"` | `"pipeline"` | `"omit"`. User-set. Default derives from stage position: early stages → `"pipeline"`, late stages → `"best_case"`, final stage before close → `"commit"`.
- `weighted_value`: computed field — `amount.value × probability.value`. Recomputed on every run if both fields are set. Used in pipeline summary report.
- `contact_roles`: array of person–role assignments for this deal. Roles: `"champion"` | `"economic_buyer"` | `"technical_buyer"` | `"blocker"` | `"influencer"` | `"user"` | `"unknown"`. Initially all `"unknown"` — user assigns roles via manual command. Pipeline may infer `"champion"` (most active contact in deal threads) and `"economic_buyer"` (senior contact with budget language) but marks these `source: "pipeline"`.
- `last_meeting_scheduled`: ISO date of the most recent CALENDAR_INVITE detected for threads linked to this deal. Updated automatically.
- `meeting_followup_due`: `last_meeting_scheduled + 7 days`. If no follow-up email is detected within this window, a FOLLOW_UP action item is created.
- `meeting_followup_status`: `"pending"` | `"received"` | `"overdue"`. Set by meeting-as-signal detector.
- `dark_period_alert`: `true` if `days_since_last_activity >= 21` and deal is not Closed Won/Lost. Computed by dark period detector.
- `days_since_last_activity`: recomputed on every run from `last_activity_date`.

### Stage Inference Logic

```python
def infer_deal_stage(company_id: str,
                     action_graph: dict,
                     chunk_graph: dict,
                     stage_schema: list) -> dict:
    """
    Infer the current deal stage for a company by matching events and
    action items against the stage entry/exit signals.

    Strategy:
    - Start from the highest possible stage (work backwards to find evidence)
    - A stage is 'reached' if at least one entry signal is present
    - A stage is 'exited' if an exit signal for that stage is present
    - The current stage is the highest reached stage that has not been exited
    - Confidence = fraction of entry signals matched / total entry signals defined
    """
    # Collect all events and action items for this company
    company_threads = [
        t['id'] for t in chunk_graph['nodes']['threads']
        # thread is linked to company via persons → WORKS_AT → company
        # (use CROSS_REF edges to find all threads involving this company's persons)
    ]
    company_events = [
        e for e in action_graph.get('events', [])
        if e.get('thread_id') in company_threads
    ]
    company_items = [
        ai for ai in action_graph['action_items']
        if ai.get('thread_id') in company_threads
    ]

    # Collect all text for keyword matching
    all_evidence_text = ' '.join(
        e.get('description', '') + ' ' + e.get('evidence', '')
        for e in company_events
    ) + ' '.join(
        ai.get('description', '') + ' ' + ai.get('evidence', '')
        for ai in company_items
    )

    event_types_present = {e['type'] for e in company_events}
    action_types_present = {ai['type'] for ai in company_items}

    # Score each stage
    stages_sorted = sorted(stage_schema, key=lambda s: s['position'], reverse=True)
    for stage in stages_sorted:
        signals = stage.get('entry_signals', {})
        hits = 0
        total = 0

        for evt_type in signals.get('events', []):
            total += 1
            if evt_type in event_types_present:
                hits += 1

        for ai_type in signals.get('action_item_types', []):
            total += 1
            if ai_type in action_types_present:
                hits += 1

        for kw in signals.get('keywords', []):
            total += 1
            if kw.lower() in all_evidence_text.lower():
                hits += 1

        if total == 0:
            continue
        confidence = hits / total
        if confidence >= 0.4:  # at least 40% of signals present → stage reached
            return {
                'stage_id':         stage['id'],
                'stage_name':       stage['name'],
                'stage_confidence': round(confidence, 2),
            }

    # Default: Prospecting (always the floor)
    return {
        'stage_id':         stages_sorted[-1]['id'],
        'stage_name':       stages_sorted[-1]['name'],
        'stage_confidence': 0.5,
    }
```

### Integration into the orchestrator

When `deal_stages.json` exists, the orchestrator runs deal stage inference after
the asymmetric commitment detector, before reporting:

```python
if os.path.exists('deal_stages.json'):
    with open('deal_stages.json') as f:
        stage_schema = json.load(f)['stages']
    action_graph = build_or_update_deals(action_graph, chunk_graph, stage_schema)
    safe_overwrite(ACTION_GRAPH_PATH, action_graph)
```

### Stage schema — add default_probability

Each stage entry in `deal_stages.json` should now include a `default_probability`
field. The pipeline uses it to compute `weighted_value` until the user overrides
the probability manually:

```json
{
  "id": "stage_03",
  "name": "Proposal",
  "position": 3,
  "default_probability": 0.50,
  "entry_signals": { ... },
  "exit_signals": { ... }
}
```

Recommended defaults: Prospecting → 0.10, Qualified → 0.25, Proposal → 0.50,
Negotiation → 0.75, Closed Won → 1.0, Closed Lost → 0.0.

### Manual commands for deal economics and contact roles

```
set deal amount for [company] to [amount]
set close date for [company] to [date]
set probability for [company] to [percent]
set forecast for [company] to commit | best_case | pipeline | omit
set [person] as champion | economic buyer | technical buyer | blocker for [company]
show pipeline summary         → weighted pipeline table across all open deals
show deal roles for [company] → contact role map for that account
```

### Clarifying questions before building your stage definitions

Before setting up deal stages, Claude will ask:

1. **Granularity** — Is a "deal" one engagement per company (all emails with Acme = one deal), or can you have multiple concurrent deals with the same company (e.g. separate renewal and expansion)?
2. **Stage direction** — Can deals go backwards? (e.g. Proposal → Qualified if they push back on pricing) or are stages strictly forward-progressing?
3. **Multi-product** — Do different product lines have different stage definitions, or one universal set?
4. **Stage owner** — Who owns the deal stage: the AE/BDM who is most active in threads with that company, or a fixed person?

---

---

## Manual Input — Rules and Implementation

Manual input fills the gaps email cannot cover: tier assignments, verbal notes,
industry tagging, new contacts met at events, corrections to misclassified data.
It runs independently of the pipeline and never triggers an email fetch.

---

### Rule 1 — Provenance tagging (mandatory)

Every field set manually is wrapped in a provenance object rather than stored
as a bare value. This is what allows the pipeline to distinguish its own
inferences from the user's explicit decisions.

```python
def manual_field(value, note: str = None) -> dict:
    """Wrap a manually-set field value with provenance metadata."""
    from datetime import datetime, timezone
    entry = {
        "value":           value,
        "source":          "manual",
        "date":            datetime.now(timezone.utc).isoformat(),
        "manual_override": True,
    }
    if note:
        entry["note"] = note
    return entry
```

**Example — tier assignment stored in company node:**

```json
"tier": {
  "value": "A",
  "source": "manual",
  "date": "2026-03-06T10:00:00Z",
  "manual_override": true
}
```

Fields that are never manually settable (always recomputed) are stored as bare
values without provenance: `health_score`, `contact_frequency_30d`,
`days_since_last_contact`, `frequency_trend`. The pipeline writes these directly
every run without checking for `manual_override`.

---

### Rule 2 — Pipeline immutability for manual fields

Before writing any field, the pipeline checks whether a manual override exists.
If it does, the pipeline skips that field entirely.

```python
def should_pipeline_update(node: dict, field: str) -> bool:
    """
    Return False if the field has been manually set — pipeline must not overwrite it.
    Return True if the field is fair game for pipeline inference.
    """
    existing = node.get(field)
    if isinstance(existing, dict) and existing.get('manual_override') is True:
        return False
    return True

# Usage in pipeline — example: updating company type
if should_pipeline_update(company_node, 'type'):
    company_node['type'] = inferred_type
# else: leave the manually-set type untouched
```

**Fields that are pipeline-immutable once manually set:**
`tier`, `industry`, `type` (when manually corrected), `account_owner_id`,
`account_owner`, `deal_stage_id` / `deal_stage_name` (when manually overridden).

**Fields always recomputed by pipeline regardless:**
`health_score`, `last_activity_date`, `days_since_last_contact`,
`contact_frequency_30d`, `contact_frequency_prev_30d`, `frequency_trend`,
`open_action_items`, `overdue_action_items`.

---

### Rule 3 — Confirmation before every write

All manual writes follow a two-step pattern: Claude shows what it will write,
waits for explicit confirmation, then commits. No manual change is committed
without an affirmative response in the dialog.

```
User:  "set Acme Corp to tier A"

Claude: I'll update Acme Corp:
        tier: null → "A"  (manual override — pipeline will not change this)

        Confirm? (yes / no)

User:  yes

Claude: ✓ Acme Corp updated. Saved to aicrm_action_items_graph.json.bak → aicrm_action_items_graph.json
```

The `.bak` write happens automatically via `safe_overwrite()` — a single manual
session with multiple changes still produces only one backup level, so if the
user makes several changes in sequence, they should be aware that only the state
before the first change in that session is recoverable from `.bak`.

---

### Rule 4 — Fuzzy company / person matching

Manual input uses names, not IDs. Claude must resolve the name to a graph node
before writing. If the match is ambiguous, surface candidates and ask.

```python
FUZZY_CONFIRM_THRESHOLD = 0.85   # even a single match above this score requires
                                  # user confirmation before any write is committed

def _similarity_score(a: str, b: str) -> float:
    """Token overlap similarity — fraction of tokens in `a` present in `b`."""
    tokens_a = set(a.lower().split())
    tokens_b = set(b.lower().split())
    if not tokens_a:
        return 0.0
    return len(tokens_a & tokens_b) / len(tokens_a)


def fuzzy_find_company(name: str, companies: list) -> list:
    """
    Return candidate companies matching the name, scored by similarity.
    Match strategy (scored, highest first):
      1. Exact match (1.0)
      2. Substring containment (0.9)
      3. Token overlap >= 0.4
      4. Domain match (0.7)
    Returns list of company dicts filtered to score >= 0.4, deduplicated.
    """
    name_lower = name.strip().lower()
    scored, seen_ids = [], set()

    for c in companies:
        cname  = c['name'].lower()
        domain = c.get('domain', '').lower()
        score  = 0.0

        if cname == name_lower:
            score = 1.0
        elif name_lower in cname or cname in name_lower:
            score = 0.9
        else:
            score = _similarity_score(name_lower, cname)
            if score < 0.4 and domain and (name_lower in domain or domain in name_lower):
                score = 0.7

        if score >= 0.4 and c['id'] not in seen_ids:
            scored.append((score, c))
            seen_ids.add(c['id'])

    return [c for _, c in sorted(scored, key=lambda x: x[0], reverse=True)]


def resolve_or_ask(name: str, candidates: list, entity_type: str) -> dict | None:
    """
    Disambiguation with mandatory user confirmation for ALL writes.

    Rules:
    - Zero candidates → return None (new entity creation prompt)
    - One candidate  → show for confirmation; never silently pick
      (prevents "set Apple to Tier A" writing to Apple Inc when
       Apple Recruitment is also in the graph)
    - Multiple candidates → numbered list for selection

    Returns None in all cases — caller waits for user response then re-enters
    with the confirmed entity via the manual command handler.
    """
    if not candidates:
        print(f"  No {entity_type} record found for '{name}'.")
        print(f"  Create a new one? (yes / no)")
        return None

    if len(candidates) == 1:
        c = candidates[0]
        print(f"  Found {entity_type}: \"{c['name']}\"  ({c.get('domain', 'no domain')})")
        print(f"  Proceed with this record? (yes / no / new)")
        return None  # wait for confirmation

    print(f"  Multiple {entity_type} records match '{name}':")
    for i, c in enumerate(candidates, 1):
        print(f"  {i}. {c['name']}  ({c.get('domain', 'no domain')})")
    print("  Which one? (enter number, or 'new' to create a new record)")
    return None


def _update_company_in_graph(action_graph: dict, company: dict) -> dict:
    """
    Replace the company node in action_graph['companies'] with the updated version.
    Matches on company['id']. If not found (shouldn't happen), appends as new.
    """
    companies = action_graph.get('companies', [])
    for i, c in enumerate(companies):
        if c['id'] == company['id']:
            companies[i] = company
            action_graph['companies'] = companies
            return action_graph
    # Not found — append (handles manually-created companies not yet in graph)
    companies.append(company)
    action_graph['companies'] = companies
    return action_graph


def _update_entity_in_graph(action_graph: dict, entity: dict) -> dict:
    """
    Replace a person or company node in the graph with the updated version.
    Detects entity type by presence of 'email' field (persons) vs 'domain' (companies).
    Matches on entity['id'].
    """
    if 'email' in entity:
        # Person node
        persons = action_graph.get('persons', [])
        for i, p in enumerate(persons):
            if p['id'] == entity['id']:
                persons[i] = entity
                action_graph['persons'] = persons
                return action_graph
        persons.append(entity)
        action_graph['persons'] = persons
    else:
        # Company node
        action_graph = _update_company_in_graph(action_graph, entity)
    return action_graph


def resolve_or_ask(name: str, candidates: list, entity_type: str) -> dict | None:
    """
    If one match: return it.
    If multiple: print numbered list and ask user to pick.
    If none: return None (will trigger new entity creation prompt).
    """
    if len(candidates) == 1:
        return candidates[0]
    if len(candidates) > 1:
        print(f"Multiple {entity_type} records match '{name}':")
        for i, c in enumerate(candidates, 1):
            print(f"  {i}. {c['name']}  ({c.get('domain', 'no domain')})")
        print("Which one? (enter number)")
        return None  # wait for user response
    return None  # no match — trigger new entity creation
```

---

### Rule 5 — Notes as the universal escape hatch

Any information that does not fit a structured field goes as a note. Notes are:
- Always appended, never overwritten or deleted
- Timestamped and tagged `source: manual`
- Attached to a company node, person node, or action item
- Searchable by the CRM Query Interface module via keyword match

```python
def append_note(node: dict, text: str, author: str = "user") -> dict:
    from datetime import datetime, timezone
    if 'notes' not in node:
        node['notes'] = []
    node['notes'].append({
        "date":    datetime.now(timezone.utc).isoformat(),
        "author":  author,
        "text":    text,
        "source":  "manual",
    })
    return node
```

Pipeline-sourced notes (e.g. evidence quotes stored during Phase 2) use
`"source": "pipeline"` and include `"chunk_id"`. They are never mixed
with manual notes in the dialog display.

---

### Rule 6 — New entity creation (no email evidence)

A manually-created entity — a company you're prospecting but haven't emailed yet,
or a contact met at a conference — has no chunks, no threads, no computed fields.

```python
def create_manual_company(name: str, domain: str = None,
                          tier: str = None, type_: str = 'client') -> dict:
    from datetime import datetime, timezone
    now = datetime.now(timezone.utc).isoformat()
    return {
        "id":                stable_id('company', name, domain or ''),
        "name":              name,
        "domain":            domain,
        "type":              manual_field(type_),
        "tier":              manual_field(tier) if tier else None,
        "source":            "manual",
        "email_evidence":    False,
        "created_date":      now,
        "threads":           [],
        "primary_contacts":  [],
        "notes":             [],
        # Computed fields initialised to null — set by pipeline once emails arrive
        "health_score":            None,
        "last_activity_date":      None,
        "days_since_last_contact": None,
        "contact_frequency_30d":   0,
        "open_action_items":       0,
    }
```

**Rules for no-evidence entities:**
- `email_evidence: false` flag is always set and never removed manually — only the pipeline sets it to `true` once a chunk links to this entity
- Health score is `null` (not 0) until email evidence exists — null entities are excluded from health score rankings and at-risk lists
- They appear in a dedicated section of the dialog report: **"Tracked — no email contact yet"**
- They participate in deal stage tracking if the user manually sets a stage on them

---

### Rule 7 — Manual action item closure

The pipeline is the only system that sets action item status to `DONE` via email
evidence. However, a user may legitimately need to close an item resolved outside
email (verbal confirmation, phone call, message in another channel).

The correct pattern is to create a **synthetic closing event** rather than directly
setting the status — this preserves the audit trail and prevents the pipeline from
re-opening the item.

```python
def manually_close_action_item(item: dict, reason: str,
                                action_graph: dict) -> dict:
    """
    Close an action item that was resolved outside email.
    Creates a synthetic MANUAL_RESOLUTION event and sets status to DONE
    with manual_override=True so the pipeline never re-opens it.
    """
    from datetime import datetime, timezone
    now = datetime.now(timezone.utc)

    # Create synthetic closing event
    closing_event = {
        "id":              stable_id('event', 'manual_resolution', item['id'], now.isoformat()),
        "type":            "MANUAL_RESOLUTION",
        "actor":           item.get('owner'),
        "actor_id":        item.get('owner_id'),
        "description":     f"Manually closed: {reason}",
        "date":            now.isoformat(),
        "thread_id":       item.get('thread_id'),
        "evidence_chunk_id": None,
        "evidence":        reason,
        "triggers_action": None,
        "source":          "manual",
    }
    action_graph.setdefault('events', []).append(closing_event)

    # Close the action item with override protection
    record_status_change(item, 'DONE', set_by='manual', now=now)
    item['manual_override'] = True
    item['closing_note']    = reason

    return action_graph
```

The pipeline respects `manual_override: true` on action item status exactly as it
does on company fields — it will never revert a manually-closed item to `OPEN`
even if new emails arrive in the same thread.

---

### Supported manual commands (full list)

| Command pattern | What it does | Fields written |
|----------------|--------------|----------------|
| `set [company] to tier [A/B/C]` | Assign account tier | `company.tier` (manual_field) |
| `add note to [company]: [text]` | Append note to company | `company.notes[]` |
| `add note to [person]: [text]` | Append note to contact | `person.notes[]` |
| `assign [company] to [person]` | Set account owner | `company.account_owner`, `company.account_owner_id` (manual_field) |
| `[company] industry is [industry]` | Set industry | `company.industry` (manual_field) |
| `mark [company] as won` | Close deal as won | `company.deal_stage` → Closed Won + DEAL_WON event (manual) |
| `mark [company] as lost` | Close deal as lost | `company.deal_stage` → Closed Lost + DEAL_LOST event (manual) |
| `add company: [name], [domain]` | Create new company node | new company node, `email_evidence: false` |
| `add contact: [name], [email], [company]` | Create new person node | new person node, `email_evidence: false` |
| `close action item [description/id]: [reason]` | Manually resolve item | `action_item.status = DONE`, synthetic MANUAL_RESOLUTION event, training label COMPLETED_BY_OWNER/OTHER |
| `dismiss [action item description/id]` | Mark item as false positive | `action_item.status = DISMISSED` (terminal), training label DISMISSED_EARLY or DISMISSED_LATE based on creation time |
| `correct [company] type to [type]` | Fix misclassified company type | `company.type` (manual_field, overrides pipeline) |

Every command follows Rule 3 — Claude shows the proposed change and waits for
confirmation before calling `safe_overwrite()`.

---

## Module: CRM Query Interface

The pipeline (Phases 1 and 2) is write-only: it processes emails and writes graphs.
The CRM Query Interface is the read layer — it allows the user to query the graphs in natural
language between pipeline runs, without triggering a new email fetch.

### Trigger phrases

The CRM Query Interface is invoked when the user asks a question that refers to CRM data without
triggering the full pipeline. Detection examples:

```
"what's the status of Acme?"          → account brief query
"who haven't I spoken to in 30 days?" → stale contact query
"what do I need to do this week?"     → personal action items
"which deals are at risk?"            → health score < 50 query
"show me everything open with Beta"   → company-scoped action items
"who is our main contact at Gamma?"   → primary contact lookup
"generate dashboard"                  → generate aicrm_dashboard.html and open
"show dashboard"                      → same as generate dashboard
```

If `aicrm_action_items_graph.json` exists, the CRM Query Interface reads it directly. If it does
not exist, instruct the user to run the pipeline first.

### Query types and graph traversal

```python
QUERY_TYPES = {

    'account_brief': {
        'description': "Full status summary for a named company",
        'inputs': ['company_name'],
        'traversal': [
            'find company by name (fuzzy match)',
            'read company node (health, stage, tier, owner)',
            'find all action items where thread_id in company.threads',
            'find all events where thread_id in company.threads',
            'find primary contacts (is_primary_contact=True)',
            'synthesise natural language brief',
        ],
        'output': 'account_brief_template'
    },

    'stale_contacts': {
        'description': "Companies or contacts not reached in N days",
        'inputs': ['days_threshold (default 30)'],
        'traversal': [
            'filter companies where days_since_last_contact > threshold',
            'sort by days_since_last_contact desc',
            'include health_score and open_action_items',
        ],
        'output': 'stale_contact_list'
    },

    'at_risk_deals': {
        'description': "Companies with health_score below threshold",
        'inputs': ['score_threshold (default 50)'],
        'traversal': [
            'filter companies where health_score < threshold and type = client',
            'sort by health_score asc',
            'include frequency_trend, overdue_action_items, days_in_current_stage',
        ],
        'output': 'risk_list'
    },

    'my_action_items': {
        'description': "Open action items owned by the internal user",
        'inputs': ['owner_email (from checkpoint or user profile)'],
        'traversal': [
            'filter action items where owner_id = internal user person_id',
            'exclude DONE, TRIGGERED',
            'sort by: COMMITMENT_OVERDUE first, then OPEN by commitment_date',
        ],
        'output': 'personal_action_list'
    },

    'company_action_items': {
        'description': "All open items related to a named company",
        'inputs': ['company_name'],
        'traversal': [
            'find company threads',
            'filter action items by thread_id',
            'group by owner (internal vs external)',
        ],
        'output': 'company_action_list'
    },

    'pipeline_summary': {
        'description': "Weighted pipeline view across all open deals",
        'inputs': [],
        'traversal': [
            'load all deals where stage_name not in DARK_PERIOD_EXEMPT_STAGES',
            'group by forecast_category (commit / best_case / pipeline / omit)',
            'compute weighted_value = amount × probability for each deal',
            'sum weighted_value per forecast_category',
            'flag deals with dark_period_alert = True',
            'flag deals with meeting_followup_status = overdue',
            'sort within each category by close_date asc',
        ],
        'output': 'pipeline_summary_table'
    },

    'deal_roles': {
        'description': "Contact role map for a named company deal",
        'inputs': ['company_name'],
        'traversal': [
            'find deal for company',
            'read deal.contact_roles array',
            'enrich each entry with person.seniority_signal, response_rate, last_contact_date',
            'flag persons with role = unknown (prompt user to assign)',
        ],
        'output': 'deal_role_map'
    },

    'dark_period_accounts': {
        'description': "All accounts with open deals that have gone silent",
        'inputs': ['threshold_days (default 21)'],
        'traversal': [
            'filter deals where dark_period_alert = True',
            'sort by days_since_last_activity desc',
            'include deal amount, stage, forecast_category',
        ],
        'output': 'dark_period_list'
    },
}
```

### Pipeline summary output format

```
═══════════════════════════════════════════════════════════════════════
  PIPELINE SUMMARY  —  as of Mar 7, 2026
  Total open: $840k  |  Weighted: $412k  |  2 dark period alerts
═══════════════════════════════════════════════════════════════════════

COMMIT                                          $180k weighted
  Acme Corp       Negotiation  Apr 30  $200k@90%   champion: Emily Blunt
  Delta Inc       Negotiation  Mar 31  $   0k@90%  ⚠ amount not set

BEST CASE                                       $175k weighted
  Beta Systems    Proposal     May 15  $350k@50%   🔇 22 days silent
  Gamma LLC       Proposal     Jun 1   $100k@50%   📅 meeting followup overdue

PIPELINE                                        $ 57k weighted
  Zeta Partners   Qualified    Jun 30  $ 80k@25%   🔇 31 days silent
  Eta Corp        Prospecting   —      amount ?    champion: unknown
═══════════════════════════════════════════════════════════════════════
  🔇 dark period   📅 meeting followup overdue   ⚠ data missing
```

### Account brief output format

```
═══════════════════════════════════════════════════════
  ACME CORP — Account Brief  [health: 74 🟡]
  Stage: Proposal (9 days)  |  Tier: A  |  Owner: James Smith
  Last contact: Feb 24 (10 days ago)  |  Trend: increasing
═══════════════════════════════════════════════════════

PRIMARY CONTACTS
  Emily Blunt — Account Manager (decision_maker)
  Last contact: Feb 24  |  Response rate: 82%

OPEN ACTION ITEMS (3)
  🔴  James Smith → Send final SOW draft  [COMMITMENT · Feb 24]
  🟡  Acme Legal → Review and sign NDA  [APPROVAL_NEEDED · Feb 15 · 19 days]
  🟡  Emily Blunt → Confirm kickoff date  [DECISION_NEEDED · Feb 20]

RECENT EVENTS
  CONTRACT_SENT — Feb 15 — MSA sent for review
  MEETING_HELD — Feb 10 — Initial scoping call

THREADS (2 active)
  "MSA and SOW for Phase 1" — last activity Feb 24
  "Project kickoff planning" — last activity Feb 20
```

### Manual annotation via query interface

The CRM Query Interface accepts write commands for data that cannot be derived from email.
Every command follows the two-step confirm-then-commit pattern from Rule 3.
Writes use `safe_overwrite()` — the previous graph is backed up to `.bak`
before any change is committed.

```python
MANUAL_COMMANDS = {
    'set_tier':          r'set\s+(.+?)\s+to\s+tier\s+([ABC])',
    'add_note':          r'add\s+note\s+to\s+(.+?):\s+(.+)',
    'assign_owner':      r'assign\s+(.+?)\s+to\s+(.+)',
    'mark_lost':         r'mark\s+(.+?)\s+as\s+lost',
    'mark_won':          r'mark\s+(.+?)\s+as\s+(won|closed)',
    'set_industry':      r'(.+?)\s+industry\s+is\s+(.+)',
    'correct_type':      r'correct\s+(.+?)\s+type\s+to\s+(.+)',
    'add_company':       r'add\s+company:\s+(.+)',
    'add_contact':       r'add\s+contact:\s+(.+)',
    'close_item':        r'close\s+action\s+item\s+(.+?):\s+(.+)',
    'dismiss_item':      r'dismiss\s+(?:action\s+item\s+)?(.+)',

    # Deal economics
    'set_amount':        r'set\s+deal\s+amount\s+for\s+(.+?)\s+to\s+([\d,\.]+)',
    'set_close_date':    r'set\s+close\s+date\s+for\s+(.+?)\s+to\s+(.+)',
    'set_probability':   r'set\s+probability\s+for\s+(.+?)\s+to\s+([\d\.]+%?)',
    'set_forecast':      r'set\s+forecast\s+for\s+(.+?)\s+to\s+(commit|best_case|pipeline|omit)',

    # Contact roles
    'set_role':          r'set\s+(.+?)\s+as\s+(champion|economic buyer|technical buyer|blocker|influencer|user)\s+for\s+(.+)',

    # Company type classification
    'set_company_type':  r'set\s+(.+?)\s+as\s+(client|vendor|partner)',

    # Dashboard
    'generate_dashboard': r'(generate|show|open|create)\s+(dashboard|report)',
}

def handle_manual_command(text: str, action_graph: dict, chunk_graph: dict) -> dict | None:
    """
    Parse a manual command, resolve entities, show confirmation, and commit.
    Returns updated action_graph on confirmation, None if cancelled.

    All writes are protected by:
      - fuzzy_find_company() / fuzzy_find_person() for entity resolution
      - should_pipeline_update() guard (already-set manual fields are shown
        with their existing value in the confirmation dialog)
      - safe_overwrite() on commit
    """
    import re

    for cmd_type, pattern in MANUAL_COMMANDS.items():
        m = re.search(pattern, text, re.IGNORECASE)
        if not m:
            continue

        if cmd_type == 'set_tier':
            company_name, tier = m.group(1).strip(), m.group(2).upper()
            candidates = fuzzy_find_company(company_name, action_graph['companies'])
            company    = resolve_or_ask(company_name, candidates, 'company')
            if not company:
                return None  # waiting for disambiguation

            existing_tier = company.get('tier', {})
            if isinstance(existing_tier, dict):
                existing_val = existing_tier.get('value', 'null')
            else:
                existing_val = existing_tier or 'null'

            # Confirmation dialog
            print(f"  Update {company['name']}:")
            print(f"    tier: {existing_val} → \"{tier}\"  (manual override — pipeline will not change this)")
            print("  Confirm? (yes / no)")
            # Execution continues after user confirms via dialog
            # On confirm:
            company['tier'] = manual_field(tier)
            action_graph    = _update_company_in_graph(action_graph, company)
            safe_overwrite(ACTION_GRAPH_PATH, action_graph)
            print(f"  ✓ {company['name']} tier set to {tier}.")

        elif cmd_type == 'add_note':
            entity_name, note_text = m.group(1).strip(), m.group(2).strip()
            # Try company first, then person
            candidates = fuzzy_find_company(entity_name, action_graph['companies'])
            if not candidates:
                candidates = [p for p in action_graph['persons']
                              if entity_name.lower() in p['name'].lower()]
            entity = resolve_or_ask(entity_name, candidates, 'company or contact')
            if not entity:
                return None

            print(f"  Add note to {entity['name']}:")
            print(f"    \"{note_text}\"")
            print("  Confirm? (yes / no)")
            # On confirm:
            entity = append_note(entity, note_text)
            action_graph = _update_entity_in_graph(action_graph, entity)
            safe_overwrite(ACTION_GRAPH_PATH, action_graph)
            print(f"  ✓ Note added to {entity['name']}.")

        elif cmd_type == 'close_item':
            item_ref, reason = m.group(1).strip(), m.group(2).strip()
            # Match action item by partial description or id
            candidates = [ai for ai in action_graph['action_items']
                          if item_ref.lower() in ai.get('description', '').lower()
                          or ai['id'] == item_ref]
            if not candidates:
                print(f"  No action item matching '{item_ref}' found.")
                return None
            item = candidates[0] if len(candidates) == 1 else resolve_or_ask(
                item_ref, candidates, 'action item')
            if not item:
                return None

            print(f"  Manually close action item:")
            print(f"    {item['owner']} → {item['description']}")
            print(f"    Reason: \"{reason}\"")
            print(f"    Status: {item['status']} → DONE  (manual override — pipeline will not reopen)")
            print("  Confirm? (yes / no)")
            # On confirm:
            # Capture training signal before closing: owner completing own item
            labeled_by_owner = (item.get('owner', '').lower() ==
                                 _current_user_name().lower())
            record_training_label(item['id'], action_graph,
                                  label_source='COMPLETED_BY_OWNER' if labeled_by_owner
                                               else 'COMPLETED_BY_OTHER')
            action_graph = manually_close_action_item(item, reason, action_graph)
            safe_overwrite(ACTION_GRAPH_PATH, action_graph)
            print(f"  ✓ Action item closed.")

        elif cmd_type == 'dismiss_item':
            item_ref = m.group(1).strip()
            candidates = [ai for ai in action_graph['action_items']
                          if item_ref.lower() in ai.get('description', '').lower()
                          or ai['id'] == item_ref]
            if not candidates:
                print(f"  No action item matching '{item_ref}' found.")
                return None
            item = candidates[0] if len(candidates) == 1 else resolve_or_ask(
                item_ref, candidates, 'action item')
            if not item:
                return None

            print(f"  Dismiss action item:")
            print(f"    {item['owner']} → {item['description']}")
            print(f"    Status: {item['status']} → DISMISSED")
            print("  Confirm? (yes / no)")
            # On confirm:
            # Derive label_source from time since creation
            created_at = item.get('first_seen') or item.get('status_history', [{}])[0].get('date', '')
            hours_since_creation = _hours_since(created_at)
            label_source = 'DISMISSED_EARLY' if hours_since_creation <= 48 else 'DISMISSED_LATE'
            record_training_label(item['id'], action_graph, label_source=label_source)
            record_status_change(item, 'DISMISSED', set_by='manual',
                                 now=datetime.now(timezone.utc))
            safe_overwrite(ACTION_GRAPH_PATH, action_graph)
            print(f"  ✓ Action item dismissed.")

        elif cmd_type == 'set_company_type':
            company_name, new_type = m.group(1).strip(), m.group(2).lower()
            candidates = fuzzy_find_company(company_name, action_graph['companies'])
            company    = resolve_or_ask(company_name, candidates, 'company')
            if not company:
                return None  # waiting for disambiguation

            existing_type   = company.get('type', 'external_unknown')
            existing_source = company.get('type_source', 'pipeline')

            print(f"  Classify {company['name']}:")
            print(f"    type: {existing_type} → \"{new_type}\"  (manual — pipeline will not change this)")
            if existing_source == 'manual':
                print(f"    ⚠ Previously manually set to '{existing_type}'. Overwriting.")
            print("  Confirm? (yes / no)")
            # On confirm:
            company['type']            = new_type
            company['type_source']     = 'manual'
            company['type_confidence'] = None   # manual classification has no confidence score
            action_graph = _update_company_in_graph(action_graph, company)
            safe_overwrite(ACTION_GRAPH_PATH, action_graph)
            print(f"  ✓ {company['name']} classified as {new_type}.")

        return action_graph

    return None  # no pattern matched — fall through to pipeline query handler
```

---

## Module: Dashboard Generator

The Dashboard Generator is a command inside the CRM Query Interface that produces a
self-contained single-file HTML dashboard from the current graph state. It requires
no separate skill, no server, and no external dependencies.

When the user says `"generate dashboard"`, `"show dashboard"`, or `"open dashboard"`:

1. Read `aicrm_action_items_graph.json` (required) and `aicrm_chunk_graph.json` (optional, for email counts).
2. Call `generate_dashboard_html(action_graph, chunk_graph)`.
3. Write the returned HTML string to `aicrm_dashboard.html` in the workspace folder using `safe_overwrite()`.
4. Print: `"Dashboard saved → [computer://…/aicrm_dashboard.html] — open in your browser."`

The HTML is entirely self-contained: graph data is embedded inline as a JavaScript constant.
No `fetch()` calls, no CDN imports, no local server required.

### Trigger phrases

```
"generate dashboard"
"show dashboard"
"open dashboard"
"create dashboard"
"show me the dashboard"
"generate report"
```

### `generate_dashboard_html()` — Python pseudocode

```python
DASHBOARD_PATH = os.path.join(WORKSPACE_DIR, 'aicrm_dashboard.html')

def generate_dashboard_html(action_graph: dict, chunk_graph: dict | None = None) -> str:
    """
    Produce a self-contained HTML string that renders the CRM graph as
    an interactive browser dashboard. No external dependencies.

    Tabs:
      1. Summary      — headline KPIs + alert cards
                        KPIs: Open Deals, Pipeline $, Weighted $, Overdue Items,
                              Blocked Items, Dark Periods, Avg Health, Win/Loss ratio.
                        Trend delta vs previous run shown under each KPI where available
                        (e.g. "+2 since last run"). Requires checkpoint data passed in.
                        Two alert cards below KPIs:
                          ⚡ Priority 0 — COMMITMENT_OVERDUE + BLOCKED items table:
                             columns: Status, Description, Company, Owner, Due
                          🔇 Dark Period Alerts — deals gone silent table:
                             columns: Company, Stage, Amount, Days Silent
      2. Pipeline     — deals grouped by forecast_category (Commit → Best Case → Pipeline → Omit)
                        Columns: Company, Stage, Deal Age (days in current stage),
                                 Close Date, Amount, Probability,
                                 Roles (champion / economic buyer / blocker — all shown),
                                 Days Silent, Flags (🔇 dark / 📅 follow-up overdue / ⚠ no amount)
                        All column headers clickable to sort asc/desc.
      3. Action Items — five filters: Status, Type, Owner (person dropdown), Company (dropdown),
                        free-text search (description + thread subject)
                        Columns: Status, Type, Description, Thread Subject, Company,
                                 Owner, Due Date, Blocked By
                        Rows are expandable — click a row to reveal the evidence quote
                        that justified extraction.
                        Sorted overdue-first then by commitment_date.
                        Paginated at 50 rows. CSV export button in tab header.
      4. Accounts     — external companies sorted by health score descending
                        Columns: Company, Type badge, Health bar+score, Frequency Trend badge
                                 (increasing / stable / declining / silent),
                                 Stage, Tier, Open Deals (count), Contacts (count),
                                 Last Contact (relative time + ISO date tooltip), Open Items
      5. People       — all persons sorted by open commitment count descending
                        Columns: Name, Company, Int/Ext badge, Deal Roles (all roles across deals),
                                 Open Commitments (count), Threads (count), Last Seen (relative)
                        Filter toggle: All / Internal / External only

    Global behaviours:
      - All table column headers are sortable (click once = asc, click again = desc,
        active sort column highlighted with ▲/▼ indicator).
      - CSV export button on Action Items tab — exports currently filtered rows.
      - Action Items and People tabs paginate at 50 rows with Prev/Next controls.
      - Header shows two timestamps: "Generated: <now>" and "Data as of: <last_run_utc>"
        pulled from the checkpoint data embedded alongside graph data.
      - All relative-time cells ("3d ago") have a title tooltip showing the full ISO date.
    """

    # ── 1. Prepare data slices ──────────────────────────────────────────────

    companies   = action_graph.get('companies', [])
    persons     = action_graph.get('persons', [])
    action_items = action_graph.get('action_items', [])
    deals       = action_graph.get('deals', [])
    generated_at = datetime.now(timezone.utc).strftime('%b %d, %Y %H:%M UTC')

    # Summary KPIs
    open_deals        = [d for d in deals if d.get('stage_name') not in ('Won', 'Lost', 'Closed')]
    open_items        = [i for i in action_items if i.get('status') not in ('DONE', 'TRIGGERED')]
    overdue_items     = [i for i in open_items if i.get('status') == 'COMMITMENT_OVERDUE']
    dark_period_deals = [d for d in open_deals if d.get('dark_period_alert')]
    avg_health        = _avg_health(companies)

    # Pipeline by forecast_category
    pipeline_groups = {}
    for deal in open_deals:
        cat = deal.get('forecast_category', 'pipeline')
        pipeline_groups.setdefault(cat, []).append(deal)

    # ── 2. Embed graph data as inline JSON ─────────────────────────────────

    import json

    # Load checkpoint for last_run_utc and previous-run KPI snapshot (for trend deltas)
    prev_snapshot = {}
    last_run_utc  = None
    try:
        with open(CHECKPOINT_PATH) as f:
            cp = json.load(f)
        last_run_utc  = cp.get('last_run_utc')
        prev_snapshot = cp.get('dashboard_kpi_snapshot', {})
        # Write current KPI snapshot into checkpoint for next run's trend deltas
        cp['dashboard_kpi_snapshot'] = {
            'open_deals':    len([d for d in deals if d.get('stage_name') not in ('Won','Lost','Closed')]),
            'overdue_items': len([i for i in action_items if i.get('status') == 'COMMITMENT_OVERDUE']),
            'blocked_items': len([i for i in action_items
                                  if i.get('blocked_by') and i.get('status') not in ('DONE','TRIGGERED','DISMISSED')]),
            'dark_periods':  len([d for d in deals
                                  if d.get('dark_period_alert') and
                                  d.get('stage_name') not in ('Won','Lost','Closed')]),
        }
        with open(CHECKPOINT_PATH, 'w') as f:
            json.dump(cp, f, indent=2)
    except Exception:
        pass  # checkpoint optional — dashboard still renders without it

    graph_json = json.dumps({
        'companies':    companies,
        'persons':      persons,
        'action_items': action_items,
        'deals':        deals,
        'last_run_utc': last_run_utc,   # shown in header as "Data as of"
        'prev':         prev_snapshot,  # previous-run KPI values for trend deltas
    }, default=str, indent=0)

    # ── 3. Build and return HTML ───────────────────────────────────────────

    html = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>AICRM Dashboard · {generated_at}</title>
<style>
  /* ── Reset & base ── */
  *, *::before, *::after {{ box-sizing: border-box; margin: 0; padding: 0; }}
  body {{ font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
         background: #0f1117; color: #e2e8f0; font-size: 14px; line-height: 1.5; }}
  a {{ color: inherit; text-decoration: none; }}

  /* ── Layout ── */
  header {{ background: #1a1d27; border-bottom: 1px solid #2d3148;
            padding: 14px 24px; display: flex; align-items: center;
            justify-content: space-between; position: sticky; top: 0; z-index: 100; }}
  header h1 {{ font-size: 16px; font-weight: 600; letter-spacing: .5px; color: #a78bfa; }}
  header .ts {{ font-size: 11px; color: #64748b; }}

  nav {{ display: flex; gap: 2px; padding: 12px 24px 0;
         border-bottom: 1px solid #2d3148; background: #0f1117; }}
  nav button {{ background: none; border: none; color: #94a3b8; cursor: pointer;
                padding: 8px 16px; font-size: 13px; font-weight: 500;
                border-bottom: 2px solid transparent; margin-bottom: -1px;
                transition: color .15s, border-color .15s; }}
  nav button.active, nav button:hover {{ color: #a78bfa; border-bottom-color: #a78bfa; }}

  main {{ padding: 24px; max-width: 1200px; margin: 0 auto; }}
  .tab-panel {{ display: none; }}
  .tab-panel.active {{ display: block; }}

  /* ── KPI cards ── */
  .kpi-row {{ display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
              gap: 12px; margin-bottom: 24px; }}
  .kpi {{ background: #1a1d27; border: 1px solid #2d3148; border-radius: 10px;
          padding: 16px 20px; }}
  .kpi .label {{ font-size: 11px; color: #64748b; text-transform: uppercase;
                 letter-spacing: .7px; margin-bottom: 4px; }}
  .kpi .value {{ font-size: 28px; font-weight: 700; color: #e2e8f0; }}
  .kpi .sub {{ font-size: 11px; color: #64748b; margin-top: 2px; }}
  .kpi.warn .value {{ color: #f97316; }}
  .kpi.danger .value {{ color: #ef4444; }}
  .kpi.ok .value {{ color: #34d399; }}

  /* ── Tables ── */
  .card {{ background: #1a1d27; border: 1px solid #2d3148; border-radius: 10px;
           overflow: hidden; margin-bottom: 16px; }}
  .card-header {{ padding: 12px 16px; border-bottom: 1px solid #2d3148;
                  font-size: 12px; font-weight: 600; text-transform: uppercase;
                  letter-spacing: .7px; color: #64748b; display: flex;
                  align-items: center; justify-content: space-between; }}
  table {{ width: 100%; border-collapse: collapse; }}
  th {{ padding: 10px 14px; text-align: left; font-size: 11px; font-weight: 600;
        text-transform: uppercase; letter-spacing: .5px; color: #64748b;
        border-bottom: 1px solid #2d3148; }}
  td {{ padding: 10px 14px; border-bottom: 1px solid #1e2233; vertical-align: top;
        font-size: 13px; }}
  tr:last-child td {{ border-bottom: none; }}
  tr:hover td {{ background: #1e2233; }}

  /* ── Badges ── */
  .badge {{ display: inline-block; padding: 2px 8px; border-radius: 20px;
            font-size: 11px; font-weight: 600; }}
  .badge-open    {{ background: #1e3a5f; color: #60a5fa; }}
  .badge-done    {{ background: #14302a; color: #34d399; }}
  .badge-overdue {{ background: #3b1c1c; color: #ef4444; }}
  .badge-review  {{ background: #2d2010; color: #fbbf24; }}
  .badge-commit  {{ background: #1c2d1c; color: #34d399; }}
  .badge-bestcase{{ background: #1c2636; color: #60a5fa; }}
  .badge-pipeline{{ background: #22222a; color: #94a3b8; }}
  .badge-client  {{ background: #1c1c3b; color: #818cf8; }}
  .badge-vendor  {{ background: #2a1c2a; color: #c084fc; }}
  .badge-partner {{ background: #1c2a2a; color: #22d3ee; }}
  .badge-unknown {{ background: #2a2a1c; color: #fbbf24; }}

  /* ── Health bar ── */
  .hbar {{ width: 80px; height: 6px; background: #2d3148; border-radius: 3px;
           overflow: hidden; display: inline-block; vertical-align: middle; }}
  .hbar-fill {{ height: 100%; border-radius: 3px;
                background: linear-gradient(90deg, #ef4444 0%, #fbbf24 50%, #34d399 100%);
                background-size: 80px 100%; }}

  /* ── Filters (action items tab) ── */
  .filters {{ display: flex; gap: 8px; flex-wrap: wrap; margin-bottom: 12px; }}
  .filters select, .filters input {{
    background: #1a1d27; border: 1px solid #2d3148; color: #e2e8f0;
    padding: 6px 10px; border-radius: 6px; font-size: 12px; }}
  .filters select:focus, .filters input:focus {{ outline: none; border-color: #a78bfa; }}

  /* ── Flag icons ── */
  .flag {{ font-size: 13px; cursor: default; }}

  /* ── Empty state ── */
  .empty {{ padding: 32px; text-align: center; color: #64748b; font-size: 13px; }}
</style>
</head>
<body>

<header>
  <h1>⚡ AICRM Dashboard</h1>
  <span class="ts">Generated {generated_at}
    <!-- data_as_of injected below from checkpoint.last_run_utc if available -->
    <span id="data-as-of"></span>
  </span>
</header>

<nav>
  <button class="active" onclick="showTab('summary',this)">Summary</button>
  <button onclick="showTab('pipeline',this)">Pipeline</button>
  <button onclick="showTab('items',this)">Action Items</button>
  <button onclick="showTab('accounts',this)">Accounts</button>
  <button onclick="showTab('people',this)">People</button>
</nav>

<main>

  <!-- ── TAB: Summary ── -->
  <div id="tab-summary" class="tab-panel active">
    <div class="kpi-row" id="kpi-row"></div>
    <div id="summary-alerts"></div>
  </div>

  <!-- ── TAB: Pipeline ── -->
  <div id="tab-pipeline" class="tab-panel">
    <div id="pipeline-content"></div>
  </div>

  <!-- ── TAB: Action Items ── -->
  <div id="tab-items" class="tab-panel">
    <div class="filters">
      <select id="filter-status" onchange="renderItems()">
        <option value="">All statuses</option>
        <option value="OPEN">Open</option>
        <option value="IN_PROGRESS">In progress</option>
        <option value="COMMITMENT_OVERDUE">Overdue</option>
        <option value="BLOCKED">Blocked</option>
        <option value="NEEDS_REVIEW">Needs review</option>
        <option value="PARTIAL">Partial</option>
        <option value="DONE">Done</option>
        <option value="TRIGGERED">Triggered</option>
      </select>
      <select id="filter-type" onchange="renderItems()">
        <option value="">All types</option>
        <option value="COMMITMENT">Commitment</option>
        <option value="REQUEST">Request</option>
        <option value="APPROVAL_NEEDED">Approval needed</option>
        <option value="DECISION_NEEDED">Decision needed</option>
        <option value="FOLLOW_UP">Follow-up</option>
        <option value="MEETING">Meeting</option>
        <option value="UNANSWERED_QUESTION">Unanswered question</option>
        <option value="SYSTEM_TRIGGER">System trigger</option>
      </select>
      <!-- Owner dropdown — populated dynamically by renderItems() init -->
      <select id="filter-owner" onchange="renderItems()">
        <option value="">All owners</option>
      </select>
      <!-- Company dropdown — populated dynamically -->
      <select id="filter-company" onchange="renderItems()">
        <option value="">All companies</option>
      </select>
      <input id="filter-search" placeholder="Search description / thread…"
             oninput="renderItems()" style="width:220px">
      <button onclick="exportItemsCSV()"
              style="margin-left:auto;background:#1a1d27;border:1px solid #2d3148;
                     color:#94a3b8;padding:6px 12px;border-radius:6px;
                     font-size:12px;cursor:pointer;">⬇ CSV</button>
    </div>
    <div id="items-content"></div>
    <div id="items-pagination" style="display:flex;gap:8px;padding:12px 0;
         align-items:center;color:#64748b;font-size:12px;"></div>
  </div>

  <!-- ── TAB: Accounts ── -->
  <div id="tab-accounts" class="tab-panel">
    <div id="accounts-content"></div>
  </div>

  <!-- ── TAB: People ── -->
  <div id="tab-people" class="tab-panel">
    <div class="filters">
      <select id="filter-people-scope" onchange="renderPeople()">
        <option value="">All people</option>
        <option value="external">External only</option>
        <option value="internal">Internal only</option>
      </select>
    </div>
    <div id="people-content"></div>
    <div id="people-pagination" style="display:flex;gap:8px;padding:12px 0;
         align-items:center;color:#64748b;font-size:12px;"></div>
  </div>

</main>

<script>
// ── Embedded graph data ──────────────────────────────────────────────────────
const G = {graph_json};

// ── Helpers ─────────────────────────────────────────────────────────────────

// showTab defined below near Init — accepts (name, btn) signature

function badgeStatus(s) {{
  const map = {{
    OPEN: 'badge-open', COMMITMENT_OVERDUE: 'badge-overdue', DONE: 'badge-done',
    TRIGGERED: 'badge-done', NEEDS_REVIEW: 'badge-review',
  }};
  return `<span class="badge ${{map[s] || 'badge-open'}}">${{s || '—'}}</span>`;
}}

function badgeForecast(f) {{
  const map = {{ commit:'badge-commit', best_case:'badge-bestcase',
                 pipeline:'badge-pipeline', omit:'badge-pipeline' }};
  return `<span class="badge ${{map[f] || 'badge-pipeline'}}">${{(f||'—').replace('_',' ')}}</span>`;
}}

function badgeType(t) {{
  const map = {{ client:'badge-client', vendor:'badge-vendor', partner:'badge-partner',
                 external_unknown:'badge-unknown' }};
  return `<span class="badge ${{map[t] || 'badge-unknown'}}">${{t || '?'}}</span>`;
}}

function healthBar(score) {{
  const s = Math.max(0, Math.min(100, score || 0));
  const pct = s + '%';
  const color = s >= 70 ? '#34d399' : s >= 40 ? '#fbbf24' : '#ef4444';
  return `<span class="hbar"><span class="hbar-fill" style="width:${{pct}};background:${{color}}"></span></span>
          <span style="margin-left:6px;color:${{color}};font-weight:600;">${{s}}</span>`;
}}

function fmt(dt) {{
  if (!dt) return '—';
  try {{ return new Date(dt).toLocaleDateString('en-US', {{month:'short', day:'numeric', year:'numeric'}}); }}
  catch {{ return dt; }}
}}

function ago(dt) {{
  if (!dt) return '—';
  const d = Math.round((Date.now() - new Date(dt)) / 86400000);
  return d === 0 ? 'today' : d === 1 ? '1d ago' : d + 'd ago';
}}

// ── Find company name for an action item via thread_id ─────────────────────

function companyForItem(item) {{
  if (item.company_id) {{
    const c = G.companies.find(x => x.id === item.company_id);
    if (c) return c.name;
  }}
  const tid = item.thread_id;
  for (const c of G.companies) {{
    if ((c.thread_ids || []).includes(tid)) return c.name;
  }}
  return '—';
}}

// ── TAB: Summary ─────────────────────────────────────────────────────────────

function renderSummary() {{
  const deals        = G.deals || [];
  const items        = G.action_items || [];
  const companies    = G.companies || [];
  const persons      = G.persons || [];
  const openDeals    = deals.filter(d => !['Won','Lost','Closed'].includes(d.stage_name));
  const wonDeals     = deals.filter(d => d.stage_name === 'Won');
  const lostDeals    = deals.filter(d => d.stage_name === 'Lost');
  const openItems    = items.filter(i => !['DONE','TRIGGERED','DISMISSED'].includes(i.status));
  const overdue      = items.filter(i => i.status === 'COMMITMENT_OVERDUE');
  const blocked      = items.filter(i => i.status !== 'DONE' && i.status !== 'TRIGGERED' && i.blocked_by);
  const p0           = items.filter(i => i.status === 'COMMITMENT_OVERDUE' ||
                                         (i.blocked_by && !['DONE','TRIGGERED','DISMISSED'].includes(i.status)));
  const dark         = openDeals.filter(d => d.dark_period_alert);
  const healths      = companies.filter(c => c.health_score != null).map(c => c.health_score);
  const avgH         = healths.length ? Math.round(healths.reduce((a,b)=>a+b,0)/healths.length) : null;
  const winRate      = (wonDeals.length + lostDeals.length) > 0
                       ? Math.round(wonDeals.length / (wonDeals.length + lostDeals.length) * 100)
                       : null;

  const totalPipeline = openDeals.reduce((s,d) => s + (parseFloat(d.amount)||0), 0);
  const totalWeighted = openDeals.reduce((s,d) => {{
    const amt = parseFloat(d.amount)||0;
    const prob = parseFloat(d.probability)||0;
    return s + amt * prob / 100;
  }}, 0);

  const fmt$ = n => n >= 1e6 ? '$'+(n/1e6).toFixed(1)+'M' : n >= 1e3 ? '$'+(n/1e3).toFixed(0)+'k' : '$'+Math.round(n);

  // Trend deltas — G.prev contains previous-run snapshot if available (from checkpoint)
  const prev = G.prev || {{}};
  const delta = (cur, prevVal) => {{
    if (prevVal == null) return '';
    const d = cur - prevVal;
    if (d === 0) return '<div class="sub" style="color:#64748b">no change</div>';
    const col = d > 0 ? '#f97316' : '#34d399';
    return `<div class="sub" style="color:${{col}}">${{d > 0 ? '+' : ''}}${{d}} since last run</div>`;
  }};

  // "Data as of" from checkpoint
  if (G.last_run_utc) {{
    document.getElementById('data-as-of').textContent =
      ' · Data as of: ' + new Date(G.last_run_utc).toLocaleString();
  }}

  document.getElementById('kpi-row').innerHTML = `
    <div class="kpi">
      <div class="label">Open deals</div>
      <div class="value">${{openDeals.length}}</div>
      ${{delta(openDeals.length, prev.open_deals)}}
      <div class="sub">${{wonDeals.length}} won · ${{lostDeals.length}} lost</div>
    </div>
    <div class="kpi">
      <div class="label">Pipeline</div>
      <div class="value">${{fmt$(totalPipeline)}}</div>
      <div class="sub">Weighted ${{fmt$(totalWeighted)}}</div>
    </div>
    <div class="kpi ${{winRate != null && winRate < 30 ? 'warn' : ''}}">
      <div class="label">Win rate</div>
      <div class="value">${{winRate != null ? winRate+'%' : '—'}}</div>
      <div class="sub">${{wonDeals.length}}W · ${{lostDeals.length}}L</div>
    </div>
    <div class="kpi ${{overdue.length > 0 ? 'danger' : 'ok'}}">
      <div class="label">Overdue items</div>
      <div class="value">${{overdue.length}}</div>
      ${{delta(overdue.length, prev.overdue_items)}}
      <div class="sub">${{openItems.length}} open total</div>
    </div>
    <div class="kpi ${{blocked.length > 0 ? 'warn' : 'ok'}}">
      <div class="label">Blocked items</div>
      <div class="value">${{blocked.length}}</div>
      ${{delta(blocked.length, prev.blocked_items)}}
      <div class="sub">unresolved dependencies</div>
    </div>
    <div class="kpi ${{dark.length > 0 ? 'warn' : 'ok'}}">
      <div class="label">Dark periods</div>
      <div class="value">${{dark.length}}</div>
      ${{delta(dark.length, prev.dark_periods)}}
      <div class="sub">deals gone silent</div>
    </div>
    <div class="kpi">
      <div class="label">Avg health</div>
      <div class="value ${{avgH!=null&&avgH<40?'danger':avgH!=null&&avgH<70?'warn':''}}">${{avgH != null ? avgH : '—'}}</div>
      <div class="sub">${{companies.filter(c=>!c.is_internal).length}} accounts</div>
    </div>
    <div class="kpi">
      <div class="label">Contacts tracked</div>
      <div class="value">${{persons.length}}</div>
      <div class="sub">${{persons.filter(p=>p.is_internal).length}} internal</div>
    </div>
  `;

  let alerts = '';

  // ⚡ Priority 0 alert card (overdue + blocked)
  if (p0.length) {{
    alerts += `<div class="card">
      <div class="card-header">⚡ Priority 0 — Immediate attention (${{p0.length}})</div>
      <table><thead><tr><th>Status</th><th>Description</th><th>Company</th><th>Owner</th><th>Due</th></tr></thead><tbody>`;
    for (const i of p0) {{
      const c = G.companies.find(x => x.id === i.company_id);
      const o = G.persons.find(x => x.id === i.owner_id);
      alerts += `<tr>
        <td>${{badgeStatus(i.status)}}</td>
        <td style="max-width:300px;white-space:normal">${{i.description||'—'}}</td>
        <td style="color:#94a3b8">${{c ? c.name : '—'}}</td>
        <td style="color:#94a3b8">${{o ? o.name : (i.owner_id||'—')}}</td>
        <td style="color:#ef4444">${{fmt(i.commitment_date)}}</td>
      </tr>`;
    }}
    alerts += '</tbody></table></div>';
  }}

  // 🔇 Dark period alerts
  if (dark.length) {{
    alerts += `<div class="card">
      <div class="card-header">🔇 Dark Period Alerts — ${{dark.length}} deal(s) gone silent</div>
      <table><thead><tr><th>Company</th><th>Stage</th><th>Amount</th><th>Silent for</th></tr></thead><tbody>`;
    for (const d of dark) {{
      const c = G.companies.find(x => x.id === d.company_id);
      alerts += `<tr>
        <td>${{c ? c.name : d.company_id || '—'}}</td>
        <td>${{d.stage_name || '—'}}</td>
        <td>${{d.amount ? '$'+Number(d.amount).toLocaleString() : '⚠ not set'}}</td>
        <td style="color:#ef4444">${{d.days_since_last_activity != null ? d.days_since_last_activity+'d' : '—'}}</td>
      </tr>`;
    }}
    alerts += '</tbody></table></div>';
  }}

  if (!alerts) alerts = '<div class="empty" style="color:#34d399">✓ No Priority 0 items or dark periods</div>';
  document.getElementById('summary-alerts').innerHTML = alerts;
}}

// ── TAB: Pipeline ─────────────────────────────────────────────────────────────

function renderPipeline() {{
  const deals   = G.deals || [];
  const open    = deals.filter(d => !['Won','Lost','Closed'].includes(d.stage_name));
  const ORDER   = ['commit','best_case','pipeline','omit'];
  const LABELS  = {{ commit:'Commit', best_case:'Best Case', pipeline:'Pipeline', omit:'Omit' }};

  if (!open.length) {{
    document.getElementById('pipeline-content').innerHTML =
      '<div class="empty">No open deals found. Set deal amounts with "set deal amount for [company] to [value]".</div>';
    return;
  }}

  let html = '';
  for (const cat of ORDER) {{
    const group = open.filter(d => (d.forecast_category || 'pipeline') === cat);
    if (!group.length) continue;
    const sumAmt = group.reduce((s,d)=>s+(parseFloat(d.amount)||0),0);
    const sumW   = group.reduce((s,d)=>s+(parseFloat(d.amount)||0)*(parseFloat(d.probability)||0)/100,0);
    const fmt$   = n => n ? '$'+Math.round(n).toLocaleString() : '—';
    html += `<div class="card">
      <div class="card-header">
        <span>${{badgeForecast(cat)}} &nbsp;${{LABELS[cat]}}</span>
        <span style="color:#94a3b8;font-weight:400;">${{fmt$(sumAmt)}} total · ${{fmt$(sumW)}} weighted</span>
      </div>
      <table><thead>
        <tr><th>Company</th><th>Stage</th><th>Deal Age</th><th>Close</th><th>Amount</th><th>Prob</th><th>Roles</th><th>Days Silent</th><th>Flags</th></tr>
      </thead><tbody>`;
    for (const d of group.sort((a,b)=>((a.close_date||'')>(b.close_date||''))?1:-1)) {{
      const c = G.companies.find(x => x.id === d.company_id);

      // All contact roles — champion, economic buyer, blocker etc.
      const roleMap = {{}};
      for (const r of (d.contact_roles||[])) {{
        const p = G.persons.find(x => x.id === r.person_id);
        const name = p ? p.name : r.person_id;
        roleMap[r.role] = (roleMap[r.role] || []).concat(name);
      }}
      const rolesHtml = Object.entries(roleMap).map(([role, names]) =>
        `<span style="font-size:10px;color:#64748b">${{role}}:</span> ${{names.join(', ')}}`
      ).join('<br>') || '—';

      // Deal age: days since stage_entered_date or first_seen
      const stageDate = d.stage_entered_date || d.first_seen;
      const dealAgeDays = stageDate
        ? Math.round((Date.now() - new Date(stageDate)) / 86400000)
        : null;

      const flags = [
        d.dark_period_alert ? '<span class="flag" title="Dark period — no activity 21+ days">🔇</span>' : '',
        d.meeting_followup_status === 'overdue' ? '<span class="flag" title="Meeting follow-up overdue">📅</span>' : '',
        !d.amount ? '<span class="flag" title="Amount not set">⚠</span>' : '',
      ].filter(Boolean).join(' ');

      html += `<tr>
        <td style="font-weight:600">${{c ? c.name : d.company_id || '—'}}</td>
        <td>${{d.stage_name || '—'}}</td>
        <td style="color:${{dealAgeDays!=null&&dealAgeDays>60?'#f97316':'#94a3b8'}}"
            title="${{stageDate||''}}">${{dealAgeDays != null ? dealAgeDays+'d' : '—'}}</td>
        <td>${{fmt(d.close_date)}}</td>
        <td>${{d.amount ? '$'+Number(d.amount).toLocaleString() : '<span style="color:#fbbf24">⚠ not set</span>'}}</td>
        <td>${{d.probability != null ? d.probability+'%' : '—'}}</td>
        <td style="font-size:12px;line-height:1.6">${{rolesHtml}}</td>
        <td style="color:${{d.dark_period_alert?'#ef4444':'#64748b'}}">${{d.days_since_last_activity != null ? d.days_since_last_activity+'d' : '—'}}</td>
        <td>${{flags || '—'}}</td>
      </tr>`;
    }}
    html += '</tbody></table></div>';
  }}
  document.getElementById('pipeline-content').innerHTML = html;
}}

// ── TAB: Action Items ─────────────────────────────────────────────────────────

const ITEMS_PAGE_SIZE = 50;
let itemsPage = 0;

function initItemsFilters() {{
  // Populate Owner dropdown from persons in graph
  const ownerSel = document.getElementById('filter-owner');
  const owners = (G.persons||[]).slice().sort((a,b)=>(a.name||'').localeCompare(b.name||''));
  owners.forEach(p => {{
    const o = document.createElement('option');
    o.value = p.id; o.textContent = p.name || p.id;
    ownerSel.appendChild(o);
  }});
  // Populate Company dropdown
  const compSel = document.getElementById('filter-company');
  const extCos = (G.companies||[]).filter(c=>!c.is_internal).slice()
                   .sort((a,b)=>(a.name||'').localeCompare(b.name||''));
  extCos.forEach(c => {{
    const o = document.createElement('option');
    o.value = c.id; o.textContent = c.name || c.id;
    compSel.appendChild(o);
  }});
}}

function exportItemsCSV() {{
  const items = getFilteredItems();
  const rows = [['Status','Type','Description','Thread','Company','Owner','Due','Blocked By']];
  for (const item of items) {{
    const ownerPerson = G.persons.find(p => p.id === item.owner_id);
    const blockerItem = item.blocked_by ? (G.action_items||[]).find(x=>x.id===item.blocked_by) : null;
    rows.push([
      item.status||'', item.type||'',
      (item.description||'').replace(/,/g,' '),
      (item.thread_subject||'').replace(/,/g,' '),
      companyForItem(item), ownerPerson ? ownerPerson.name : (item.owner_id||''),
      item.commitment_date||'',
      blockerItem ? (blockerItem.description||'').replace(/,/g,' ') : '',
    ]);
  }}
  const csv = rows.map(r=>r.map(v=>`"${{v}}"`).join(',')).join('\n');
  const a = document.createElement('a');
  a.href = 'data:text/csv,' + encodeURIComponent(csv);
  a.download = 'aicrm_action_items.csv'; a.click();
}}

function getFilteredItems() {{
  const statusF  = document.getElementById('filter-status').value;
  const typeF    = document.getElementById('filter-type').value;
  const ownerF   = document.getElementById('filter-owner').value;
  const companyF = document.getElementById('filter-company').value;
  const searchF  = document.getElementById('filter-search').value.toLowerCase();

  let items = G.action_items || [];
  if (statusF)  items = items.filter(i => i.status === statusF);
  if (typeF)    items = items.filter(i => i.type === typeF);
  if (ownerF)   items = items.filter(i => i.owner_id === ownerF);
  if (companyF) items = items.filter(i => i.company_id === companyF ||
    (G.companies.find(c=>c.id===companyF)?.thread_ids||[]).includes(i.thread_id));
  if (searchF)  items = items.filter(i =>
    (i.description||'').toLowerCase().includes(searchF) ||
    (i.thread_subject||'').toLowerCase().includes(searchF)
  );

  return items.sort((a,b) => {{
    if (a.status==='COMMITMENT_OVERDUE' && b.status!=='COMMITMENT_OVERDUE') return -1;
    if (b.status==='COMMITMENT_OVERDUE' && a.status!=='COMMITMENT_OVERDUE') return 1;
    return ((a.commitment_date||'')>(b.commitment_date||'')) ? 1 : -1;
  }});
}}

function renderItems() {{
  itemsPage = 0;
  renderItemsPage();
}}

function renderItemsPage() {{
  const items = getFilteredItems();
  const total = items.length;
  const page  = items.slice(itemsPage * ITEMS_PAGE_SIZE, (itemsPage+1) * ITEMS_PAGE_SIZE);

  if (!total) {{
    document.getElementById('items-content').innerHTML =
      '<div class="empty">No action items match the current filters.</div>';
    document.getElementById('items-pagination').innerHTML = '';
    return;
  }}

  let html = `<div class="card"><table><thead>
    <tr><th>Status</th><th>Type</th><th>Description</th><th>Thread</th>
        <th>Company</th><th>Owner</th><th>Due</th><th>Blocked By</th></tr>
  </thead><tbody>`;

  for (const item of page) {{
    const ownerPerson  = G.persons.find(p => p.id === item.owner_id);
    const ownerName    = ownerPerson ? ownerPerson.name : (item.owner_id || '—');
    const blockerItem  = item.blocked_by
                         ? (G.action_items||[]).find(x => x.id === item.blocked_by)
                         : null;
    const blockerLabel = blockerItem
                         ? `<span style="font-size:11px;color:#fbbf24" title="${{blockerItem.description||''}}"
                             >🔒 ${{(blockerItem.description||'').slice(0,40)}}…</span>`
                         : '—';
    const evidenceId = 'ev-' + item.id;
    html += `<tr onclick="toggleEvidence('${{evidenceId}}')" style="cursor:pointer">
      <td>${{badgeStatus(item.status)}}</td>
      <td><span style="font-size:11px;color:#64748b">${{item.type || '—'}}</span></td>
      <td style="max-width:300px;white-space:normal">${{item.description || '—'}}</td>
      <td style="font-size:11px;color:#64748b;max-width:160px;white-space:normal">${{item.thread_subject || '—'}}</td>
      <td style="color:#94a3b8">${{companyForItem(item)}}</td>
      <td style="color:#94a3b8">${{ownerName}}</td>
      <td style="color:${{item.status==='COMMITMENT_OVERDUE'?'#ef4444':'#94a3b8'}}"
          title="${{item.commitment_date||''}}">${{fmt(item.commitment_date)}}</td>
      <td>${{blockerLabel}}</td>
    </tr>`;
    if (item.evidence) {{
      html += `<tr id="${{evidenceId}}" style="display:none">
        <td colspan="8" style="background:#0f1117;padding:10px 14px;
            font-size:12px;color:#64748b;font-style:italic;border-left:3px solid #a78bfa">
          📎 Evidence: "${{item.evidence}}"
        </td></tr>`;
    }}
  }}
  html += '</tbody></table></div>';
  document.getElementById('items-content').innerHTML = html;

  // Pagination controls
  const totalPages = Math.ceil(total / ITEMS_PAGE_SIZE);
  document.getElementById('items-pagination').innerHTML = totalPages <= 1 ? '' : `
    <span>Page ${{itemsPage+1}} of ${{totalPages}} (${{total}} items)</span>
    <button onclick="itemsPage=Math.max(0,itemsPage-1);renderItemsPage()"
            ${{itemsPage===0?'disabled':''}}
            style="padding:4px 10px;background:#1a1d27;border:1px solid #2d3148;
                   color:#e2e8f0;border-radius:4px;cursor:pointer">← Prev</button>
    <button onclick="itemsPage=Math.min(${{totalPages-1}},itemsPage+1);renderItemsPage()"
            ${{itemsPage>=totalPages-1?'disabled':''}}
            style="padding:4px 10px;background:#1a1d27;border:1px solid #2d3148;
                   color:#e2e8f0;border-radius:4px;cursor:pointer">Next →</button>
  `;
}}

function toggleEvidence(id) {{
  const el = document.getElementById(id);
  if (el) el.style.display = el.style.display === 'none' ? '' : 'none';
}}

// ── TAB: Accounts ─────────────────────────────────────────────────────────────

function renderAccounts() {{
  const companies = (G.companies || [])
    .filter(c => !c.is_internal)
    .sort((a,b) => (b.health_score||0) - (a.health_score||0));

  if (!companies.length) {{
    document.getElementById('accounts-content').innerHTML =
      '<div class="empty">No external companies found yet. Run the pipeline first.</div>';
    return;
  }}

  // Frequency trend badge
  function trendBadge(t) {{
    const map = {{
      increasing: ['#34d399','↑ increasing'],
      stable:     ['#60a5fa','→ stable'],
      declining:  ['#fbbf24','↓ declining'],
      silent:     ['#ef4444','✕ silent'],
    }};
    const [col, label] = map[t] || ['#64748b', t || '—'];
    return `<span style="font-size:11px;color:${{col}}">${{label}}</span>`;
  }}

  let html = `<div class="card"><table><thead>
    <tr><th>Company</th><th>Type</th><th>Health</th><th>Trend</th><th>Stage</th><th>Tier</th>
        <th>Open Deals</th><th>Contacts</th><th>Last contact</th><th>Open items</th></tr>
  </thead><tbody>`;

  for (const c of companies) {{
    const tier = typeof c.tier === 'object' ? c.tier.value : c.tier;
    const openItems = (G.action_items||[]).filter(i =>
      !['DONE','TRIGGERED','DISMISSED'].includes(i.status) &&
      (i.company_id === c.id || (c.thread_ids||[]).includes(i.thread_id))
    ).length;
    const openDealsCount = (G.deals||[]).filter(d =>
      d.company_id === c.id && !['Won','Lost','Closed'].includes(d.stage_name)
    ).length;
    const contactCount = (G.persons||[]).filter(p =>
      p.domain && c.domain && p.domain === c.domain && !p.is_internal
    ).length;

    html += `<tr>
      <td style="font-weight:600">${{c.name || '—'}}</td>
      <td>${{badgeType(c.type)}}</td>
      <td>${{c.health_score != null ? healthBar(c.health_score) : '<span style="color:#64748b">—</span>'}}</td>
      <td>${{trendBadge(c.frequency_trend)}}</td>
      <td style="color:#94a3b8">${{c.current_stage || '—'}}</td>
      <td style="color:#94a3b8">${{tier || '—'}}</td>
      <td style="color:${{openDealsCount>0?'#60a5fa':'#64748b'}}">${{openDealsCount || '—'}}</td>
      <td style="color:#94a3b8">${{contactCount || '—'}}</td>
      <td style="color:#64748b" title="${{c.last_activity_date||''}}">${{ago(c.last_activity_date)}}</td>
      <td style="color:${{openItems>0?'#60a5fa':'#64748b'}}">${{openItems || '—'}}</td>
    </tr>`;
  }}
  html += '</tbody></table></div>';
  document.getElementById('accounts-content').innerHTML = html;
}}

// ── TAB: People ──────────────────────────────────────────────────────────────

const PEOPLE_PAGE_SIZE = 50;
let peoplePage = 0;

function renderPeople() {{
  peoplePage = 0;
  renderPeoplePage();
}}

function renderPeoplePage() {{
  const scopeF = document.getElementById('filter-people-scope').value;

  let persons = (G.persons || []).slice();
  if (scopeF === 'external') persons = persons.filter(p => !p.is_internal);
  if (scopeF === 'internal') persons = persons.filter(p => p.is_internal);

  // Compute open commitment count per person
  persons = persons.map(p => {{
    const openCommitments = (G.action_items||[]).filter(i =>
      i.owner_id === p.id &&
      !['DONE','TRIGGERED','DISMISSED'].includes(i.status)
    ).length;
    // Collect all deal roles for this person across all deals
    const roles = [];
    for (const d of (G.deals||[])) {{
      for (const r of (d.contact_roles||[])) {{
        if (r.person_id === p.id) roles.push(r.role);
      }}
    }}
    return {{ ...p, openCommitments, roles: [...new Set(roles)] }};
  }}).sort((a,b) => b.openCommitments - a.openCommitments);

  const total     = persons.length;
  const totalPages = Math.ceil(total / PEOPLE_PAGE_SIZE);
  const page      = persons.slice(peoplePage * PEOPLE_PAGE_SIZE, (peoplePage+1) * PEOPLE_PAGE_SIZE);

  if (!total) {{
    document.getElementById('people-content').innerHTML =
      '<div class="empty">No people found.</div>';
    return;
  }}

  let html = `<div class="card"><table><thead>
    <tr><th>Name</th><th>Company</th><th>Type</th><th>Deal Roles</th>
        <th>Open Commitments</th><th>Threads</th><th>Last Seen</th></tr>
  </thead><tbody>`;

  for (const p of page) {{
    const co = G.companies.find(c => c.id === p.company_id);
    const intBadge = p.is_internal
      ? `<span style="font-size:11px;color:#818cf8">internal</span>`
      : `<span style="font-size:11px;color:#94a3b8">external</span>`;
    const rolesLabel = p.roles.length
      ? p.roles.map(r=>`<span style="font-size:10px;color:#64748b">${{r}}</span>`).join(' ')
      : '—';

    html += `<tr>
      <td style="font-weight:600">${{p.name || p.email || '—'}}</td>
      <td style="color:#94a3b8">${{co ? co.name : (p.domain || '—')}}</td>
      <td>${{intBadge}}</td>
      <td>${{rolesLabel}}</td>
      <td style="color:${{p.openCommitments>0?'#60a5fa':'#64748b'}}">${{p.openCommitments || '—'}}</td>
      <td style="color:#64748b">${{(p.appears_in_threads||[]).length || '—'}}</td>
      <td style="color:#64748b" title="${{p.last_seen||''}}">${{ago(p.last_seen)}}</td>
    </tr>`;
  }}
  html += '</tbody></table></div>';
  document.getElementById('people-content').innerHTML = html;

  document.getElementById('people-pagination').innerHTML = totalPages <= 1 ? '' : `
    <span>Page ${{peoplePage+1}} of ${{totalPages}} (${{total}} people)</span>
    <button onclick="peoplePage=Math.max(0,peoplePage-1);renderPeoplePage()"
            ${{peoplePage===0?'disabled':''}}
            style="padding:4px 10px;background:#1a1d27;border:1px solid #2d3148;
                   color:#e2e8f0;border-radius:4px;cursor:pointer">← Prev</button>
    <button onclick="peoplePage=Math.min(${{totalPages-1}},peoplePage+1);renderPeoplePage()"
            ${{peoplePage>=totalPages-1?'disabled':''}}
            style="padding:4px 10px;background:#1a1d27;border:1px solid #2d3148;
                   color:#e2e8f0;border-radius:4px;cursor:pointer">Next →</button>
  `;
}}

// ── showTab: updated to accept button reference ───────────────────────────────
function showTab(name, btn) {{
  document.querySelectorAll('.tab-panel').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('nav button').forEach(b => b.classList.remove('active'));
  document.getElementById('tab-' + name).classList.add('active');
  if (btn) btn.classList.add('active');
}}

// ── Init ─────────────────────────────────────────────────────────────────────
initItemsFilters();
renderSummary();
renderPipeline();
renderItems();
renderAccounts();
renderPeople();
</script>
</body>
</html>"""

    return html


def _avg_health(companies: list) -> int | None:
    """Compute average health score across companies that have one."""
    scores = [c['health_score'] for c in companies if c.get('health_score') is not None]
    return round(sum(scores) / len(scores)) if scores else None


def handle_generate_dashboard(action_graph: dict, chunk_graph: dict | None = None,
                               silent: bool = False) -> str:
    """
    Generate aicrm_dashboard.html from the current graphs and save to workspace.
    Returns the workspace file path.

    silent=True  — called from run_full / run_incremental; no print output
                   (the caller prints the single dashboard link itself).
    silent=False — called from manual 'generate dashboard' command; print one
                   confirmation line for the user.
    """
    if not action_graph:
        return "Error: action graph not loaded — run the pipeline first."

    html = generate_dashboard_html(action_graph, chunk_graph)
    safe_overwrite(DASHBOARD_PATH, html, mode='text')

    if not silent:
        item_count    = len(action_graph.get('action_items', []))
        company_count = len([c for c in action_graph.get('companies', []) if not c.get('is_internal')])
        deal_count    = len([d for d in action_graph.get('deals', [])
                              if d.get('stage_name') not in ('Won', 'Lost', 'Closed')])
        print(f"\n  Dashboard → computer://{DASHBOARD_PATH}"
              f"  ({company_count} accounts · {deal_count} open deals · {item_count} action items)")

    return DASHBOARD_PATH
```

### safe_overwrite text mode

`safe_overwrite()` already exists for JSON graphs. Add a `mode='text'` branch
so it can write plain string content (HTML):

```python
def safe_overwrite(path: str, data, mode: str = 'json') -> None:
    """
    Atomically write data to path, keeping a .bak of the previous version.

    mode='json'  → data is a dict/list, written as pretty-printed JSON.
    mode='text'  → data is a string, written as-is (for HTML output).
    """
    if os.path.exists(path):
        shutil.copy2(path, path + '.bak')
    with open(path, 'w', encoding='utf-8') as f:
        if mode == 'json':
            import json
            json.dump(data, f, indent=2, default=str)
        else:
            f.write(data)
```

### Routing in `handle_manual_command`

Add the dashboard branch **before** the `return None` at the end of
`handle_manual_command`:

```python
        if cmd_type == 'generate_dashboard':
            chunk_graph = None
            if os.path.exists(CHUNK_GRAPH_PATH):
                chunk_graph = load_json(CHUNK_GRAPH_PATH)
            handle_generate_dashboard(action_graph, chunk_graph)
            return action_graph  # no graph mutation — returned unchanged
```

---

## PoC Acceptance Criteria

The skill runs against the user's connected mailbox — no synthetic harness required.
The PoC is validated by invoking the skill on a real inbox and verifying it handles
the following 8 scenarios correctly. Use these as a mental checklist on first run.

| # | Scenario | What to verify |
|---|----------|---------------|
| 1 | **Unresolved commitment** — you promised to send something and never did | Action item appears as `COMMITMENT_OVERDUE`, correct owner and description |
| 2 | **Resolved request** — you asked someone for something and they sent it | Action item shows `DONE`, delivery evidence quote is accurate |
| 3 | **Calendar invite received** — external contact books a meeting | `MEETING` action item created from invite fields, not NLP-extracted from body |
| 4 | **Meeting with follow-up** — meeting happened, follow-up email arrived within 7 days | Deal node: `meeting_followup_status: received` |
| 5 | **Auto-reply received** — out-of-office response to one of your emails | No action items extracted for this email |
| 6 | **First email from a tracked company** — company was manually added, now emails for the first time | Company node: `email_evidence` promoted from `false` to `true` |
| 7 | **Vague email, no clear commitment** — "we should catch up sometime" | Item appears as `NEEDS_REVIEW` with low `extraction_confidence`, not as `OPEN` |
| 8 | **Deal gone silent** — open deal, no email activity for 21+ days | Dark period alert appears at top of dialog report |

### Identity verification (first run)

Before reviewing action items, confirm the identity banner at the top of the report is correct:

```
AICRM  ·  [Your name]  (team · yourdomain.com)   ← custom domain
AICRM  ·  [Your name]  (solo)                     ← gmail / yahoo / etc.
```

If the classification is wrong (e.g. you use Gmail but work with one main client),
add a note via the CRM Query Interface: `"add note to [company]: primary client — treat as internal"`.
Full multi-domain `internal_domains` support is a v3 item.

---