Ambient CRM is a Claude skill that reads your email, extracts structured sales intelligence, and maintains a living graph of companies, contacts, action items, and deal signals — automatically, on every run. It is not a CRM integration but a replacement for the assumption that humans should file their own work.

What the skill produces on every run:
- Structured graph of all companies and contacts extracted from communication
- Open action items with owners, types, and age
- Deal events and blockers per thread
- Delta report: what changed since the last run

What is possible but requires user input:
- deal process definition
- company offerings
- slaes processes definition

What you never have to do:
- Log a call
- Update a deal stage
- Create a follow-up task
- Fill out a contact record


Prerequisites:
Claude Desktop installed
Mail connected via Claude Connectors
Claude Pro or Team plan (skill requires extended context)


Installation
1. Download the skill 
2. Open Claude Desktop
3. Go to Settings → Skills
4. Click Add Skill and select the skill you downloaded
5. Skill will appear in your skill

Check your mailbox is connected:
- In Claude Desktop, go to Settings → Connectors
- Click Add Connector → Gmail or M365
- Authorize access
- Return to the main chat


Activating the Skill in Claude Desktop
Once installed, open a new Claude chat and type:
Run the Ambient CRM skill

Claude will:
Discover your connected mailbox automatically
On first run: process last 1000 mails for 90 days (Junk and Trash excluded)
On subsequent runs: process only new emails since the last checkpoint
Print a summary of open action items and deal events in the chat

You do not need to specify folders, date ranges, or any parameters. The skill handles all of that.
You can ask Claude directly for any insights you want to get

Output Files
The skill writes three files to your working directory on every run:
FileContentsaicrm_chunk_graph.jsonStructural graph: threads, email chunks, persons, companies, edgesaicrm_action_items_graph.jsonSemantic graph: action items, events, blockers, deal signalsaicrm_checkpoint.jsonRun state: last processed timestamp and message IDs
A .bak of the action items graph is kept for one rollback level.

Querying Your CRM
After the first run, you can query your data in natural language:
Show me all open action items for Acme Corp
What happened with the Freightos deal this week
Who hasn't replied to us in over 14 days
Mark the NDA review as done: signed and returned Feb 28
Set Acme Corp to Tier A
Manual entries are tagged with provenance so the pipeline never overwrites them.

Architecture
The skill runs a two-phase pipeline:
Phase 1 — Chunk Graph
Each email becomes a chunk node. Threads are reconstructed via Message-ID headers. Quoted reply text is stripped and stored separately to prevent duplicate extraction.
Phase 2 — Action Item Graph
Chunks are traversed oldest-to-newest per thread. The LLM extracts persons, companies, events, and action items with stable IDs so incremental runs update existing items rather than creating duplicates.
Checkpoint System
Message-ID based deduplication ensures each email is processed exactly once, regardless of connector timestamp reliability.

Roadmap for the plug-in (near-term)
PII tokenization for legal compliance
DB for mailboxes exceeding ~500 threads
Confidence decay on stale open items
Multi-mailbox: merge team members into a shared graph
Behavioral baseline engine: detect when account engagement deviates from normal
Gap detection: surface what should have happened but didn't
Data export for fine-tuning domain-specific models
NanoClaw orchestration for scheduled autonomous runs
