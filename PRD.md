## 1. Problem Statement

Administrators cannot easily trace exactly **where** automation by Atom ends and **human processes** begin, nor quantify the precise breakdown by theme and sub‑theme. Traditional dashboards show raw metrics but fail to tell a coherent, evidence‑backed story that surfaces automation success, hand‑off delays, SLA breaches and thematic bottlenecks.

## 2. Goals & Non‑Goals

| Goal                                                                                       | Success Metric                                                   | Non‑Goal                           |
| ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------- | ---------------------------------- |
| G1. Surface story‑driven insights split into Act 1 (automation) & Act 2 (human).           | ≥ 80 % of pilot admins report “I understand where delays occur.” | Replace existing metric tables.    |
| G2. Provide drill‑down deep-links to raw data (chat logs, runbook traces, ticket history). | 100 % of insights include at least one evidence link.            | Build a full BI platform.          |
| G3. Support three journey genres: Incidents, Service Requests, Other.                      | Each genre yields ≥ 3 distinct insight instances.                | Add Change Mgmt or Projects in v1. |

## 3. Personas

* **Executive**: Needs high‑level narratives and slide‑ready exports.
* **Admin (ITSM Lead)**: Requires granular metrics and evidence to diagnose bottlenecks.
* **Support Agent**: Occasionally drills into a single story for queue‑level improvements.

## 4. User Stories

| # | As a …    | I want to …                               | So that …                                           | Acceptance Criteria                                                    |
| - | --------- | ----------------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| 1 | Admin     | open the Story Gallery and select a genre | I see only insights for that journey                | Gallery shows three cards; clicking loads the correct dashboard.       |
| 2 | Admin     | view Act 1 Sankey for a conversation      | I know if Atom attempted automation & why it failed | Sankey nodes: Intent → Automation (status, failure reasons) → Outcome. |
| 3 | Admin     | click the “Handed off” segment            | I jump to Act 2 timeline for that flow              | Deep‑link to filtered ticket list highlighting first assignment.       |
| 4 | Executive | export top insights to PDF                | I can include them in a quarterly deck              | PDF includes charts, narrative, methodology footer.                    |
| 5 | Admin     | mark an insight as actioned               | the remediation team can track progress             | Toggle updates insight status and filters it out of open list.         |

## 5. Functional Requirements

### 5.1 Data Model (PostgreSQL)

```sql
-- Conversation
CREATE TABLE conversations (
  conv_id         TEXT PRIMARY KEY,
  theme           TEXT,
  sub_theme       TEXT,
  automated       JSONB,       -- {status: bool, failure: text|null}
  tickets         TEXT[],      -- ['sr-123', ...]
  status          TEXT CHECK(status IN ('open','resolved')),
  resolution_notes TEXT      -- free-form resolution notes
);

CREATE TABLE messages (
  msg_id          BIGSERIAL PRIMARY KEY,
  conv_id         TEXT REFERENCES conversations(conv_id) ON DELETE CASCADE,
  direction       TEXT CHECK(direction IN ('user','atom')),
  content         TEXT,
  cta             TEXT[] DEFAULT '{}',
  clicked_ctas    JSONB DEFAULT '[]'  -- [{cta: text, timestamp: ts}]
);

-- Ticket (child of a conversation)
CREATE TABLE tickets (
  ticket_id        TEXT PRIMARY KEY,
  conv_id          TEXT REFERENCES conversations(conv_id) ON DELETE CASCADE,
  created_at       TIMESTAMP NOT NULL,
  resolved_at      TIMESTAMP,
  assignment       JSONB,     -- [{timestamp, assigned_by, assigned_to}, ...]
  sla_milestones   JSONB,     -- [{milestone, timestamp, breached: bool}, ...]
  messages         JSONB,     -- follow‑up messages array
  status           TEXT CHECK(status IN ('open','resolved')),
  resolution_notes TEXT       -- free-form resolution notes
);
```

### 5.2 Insight Generation Batch Job

* **Frequency**: Every 6 h (with on-demand trigger).
* **Process**:

  1. Fetch new/updated conversations & linked tickets.
  2. Derive `theme`/`sub_theme` via clustering of intents & ticket metadata.
  3. Compute **Act 1 metrics**:

     * Messages → first CTA, automation hit/failure rate, failure reasons.
  4. Compute **Act 2 metrics**:

     * Time to first assignment, total assignment hops, SLA breach count, MTTR.
  5. Upsert into `insight_instances` table:

     ```sql
     INSERT … ON CONFLICT … DO UPDATE …;
     ```

  Stores: `conv_id`, `title`, `genre`, `act1_metrics JSONB`, `act2_metrics JSONB`, `evidence_links`.

### 5.3 APIs

| Method | Path                         | Description                                               |
| ------ | ---------------------------- | --------------------------------------------------------- |
| GET    | `/conversations/{conv_id}`   | Full conversation payload (messages, automated, tickets). |
| GET    | `/tickets/{ticket_id}`       | Full ticket payload (messages, assignment, SLA).          |
| GET    | `/insights?genre=…&period=…` | List insight summaries for a genre and period.            |
| GET    | `/insights/{id}`             | Detail including Act 1 & 2 metrics + evidence links.      |
| POST   | `/insights/run-batch`        | Trigger on-demand batch reprocessing.                     |
| PATCH  | `/insights/{id}`             | Update insight status (e.g. actioned).                    |

### 5.4 UI Requirements

* **Insight Detail**:

  * **Entry Point**: If a conversation has no associated tickets (and `automated.status=true`), treat as fully automated and render the Unified Sankey.
  * **Portal‑Created Tickets**: If a ticket originates from a portal (flagged via metadata), do **not** show the Sankey; instead display an informational banner: *"Created via Portal"*.
  * **Unified Sankey (3‑Node)**: A single Sankey diagram showing:

    1. **Atom Conversation** (start node)
    2. **Response Type**: one of *Response Generated* / *Catalog Provided* / *Automation Triggered*
    3. **Outcome**: *Success* (resolved in Act 1) or *Failure (hand‑off to teammate)*
  * **Conditional Act 2 Metrics**: Directly beneath the Sankey, display Act 2 metrics **only** if the Outcome is Failure:

    * Time to first assignment
    * Total assignment hops
    * SLA breach indicators
    * Resolution time (MTTR)
  * **Evidence Links**: Contextual deep-links on Sankey nodes and Act 2 metric labels, opening the conversation or ticket detail at the relevant step.

## 6. Non‑Functional Requirements Non‑Functional Requirements

| Aspect        | Requirement                                                             |
| ------------- | ----------------------------------------------------------------------- |
| Performance   | List API ≤ 500 ms; detail API ≤ 2 s (P95) over 90‑day window.           |
| Security      | Tenancy isolation; role‑based access; PDFs watermarked.                 |
| Scalability   | Handle up to 250 K tickets & 500 K conversations per quarter.           |
| Observability | Metrics: batch job duration, failures; API latencies; front‑end errors. |
| Accessibility | WCAG 2.1 AA; charts keyboard navigable; alt‑text for Sankeys.           |

## 9. Dependencies & Assumptions

* **NLU & Workflow Services**: Atom’s intent labels, CTA metadata and workflow logs must be surfaced via a shared event stream.
* **Assignment Events**: Ticket‑assignment service emits JSON events with timestamps and actors.
* **SLA Engine**: A centralized SLA calculator provides milestone timestamps and breach flags.
* **Batch Infrastructure**: Cron or scheduler with ability to backfill 90+ days of data.
* **Deep‑link Routing**: Front‑end routes must accept conversation and ticket IDs plus message offsets.

## 10. Risks & Mitigations

| Risk                                                  | Mitigation                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------------- |
| Incomplete CTA tracking → undercounted Act 1 flows.   | Validate event stream in staging; fallback to message position.      |
| Theme clustering inaccuracies → mis‑grouped insights. | Periodic manual review; allow admin to override theme in UI.         |
| Large JSONB reads → slow detail loads.                | Index key JSONB fields; paginate heavy arrays; cache common queries. |
| PDF generation OOM for large tenants.                 | Limit max insight count per export; use streaming renderer.          |

---

## Sample Data (CSV)

Below are twenty sample conversation record sets in CSV format, with corresponding messages and tickets for those that were handed off.

**conversations.csv**

```csv
conv_id,theme,sub_theme,automated,status,resolution_notes,tickets
"conv-001","Salesforce access","Direct access","{\"status\":false,\"failure\":\"Permission API error\"}","open","","sr-1001"
"conv-002","VPN incident","Connectivity","{\"status\":true,\"failure\":null}","resolved","VPN restart resolved issue.",""
"conv-003","Email issue","Outbound mail","{\"status\":false,\"failure\":\"SMTP timeout\"}","open","","sr-1003"
"conv-004","Database error","Read-only mode","{\"status\":true,\"failure\":null}","resolved","Recovered from cache flush.",""
"conv-005","HR portal","Leave balance","{\"status\":false,\"failure\":\"KB article not found\"}","open","","sr-1005"
"conv-006","VPN incident","Authentication","{\"status\":false,\"failure\":\"Auth API error\"}","open","","inc-2006"
"conv-007","Software install","Antivirus","{\"status\":true,\"failure\":null}","resolved","Agent installed successfully.",""
"conv-008","Printer issue","Paper jam","{\"status\":false,\"failure\":\"Driver missing\"}","open","","sr-1008"
"conv-009","Slack access","Channel join","{\"status\":true,\"failure\":null}","resolved","Access granted by AD sync.",""
"conv-010","Network outage","WiFi drop","{\"status\":false,\"failure\":\"Unknown SSID\"}","open","","inc-2010"
"conv-011","License request","Windows license","{\"status\":true,\"failure\":null}","resolved","Key auto-provisioned.",""
"conv-012","VPN incident","Latency","{\"status\":true,\"failure\":null}","resolved","Route optimized.",""
"conv-013","Salesforce access","Role change","{\"status\":false,\"failure\":\"Insufficient rights\"}","open","","sr-1013"
"conv-014","Billing question","Invoice status","{\"status\":true,\"failure\":null}","resolved","Invoice emailed.",""
"conv-015","Email issue","Spam filter","{\"status\":false,\"failure\":\"Policy conflict\"}","open","","sr-1015"
"conv-016","VPN incident","DNS resolution","{\"status\":true,\"failure\":null}","resolved","DNS cache cleared.",""
"conv-017","HR portal","Expense report","{\"status\":false,\"failure\":\"Form schema error\"}","open","","sr-1017"
"conv-018","Software install","Office suite","{\"status\":true,\"failure\":null}","resolved","Deployment succeeded.",""
"conv-019","Printer issue","Network printer","{\"status\":false,\"failure\":\"Printer offline\"}","open","","sr-1019"
"conv-020","Network outage","VPN certificate","{\"status\":true,\"failure\":null}","resolved","Cert auto-renewed.",""
```

**messages.csv**

```csv
msg_id,conv_id,direction,content,cta,clicked_ctas
1,conv-001,user,"Hi, I need SFDC access","[]","[]"
2,conv-001,atom,"Found the service catalog: Request for Salesforce:submitted","[Request for Salesforce:submitted]","[{\"cta\":\"Request for Salesforce:submitted\",\"timestamp\":\"2025-07-01T09:15:00+05:30\"}]"
3,conv-002,user,"My VPN is down","[]","[]"
4,conv-002,atom,"Automated VPN restart succeeded","[]","[]"
5,conv-003,user,"Outgoing emails are failing","[]","[]"
6,conv-003,atom,"Triggered runbook: SMTP retry","[]","[]"
7,conv-004,user,"DB is in read-only","[]","[]"
8,conv-004,atom,"Cache flush automation succeeded","[]","[]"
9,conv-005,user,"What is my leave balance?","[]","[]"
10,conv-005,atom,"Sorry, no KB found. Raising request.","[]","[]"
11,conv-006,user,"VPN auth error","[]","[]"
12,conv-006,atom,"Auth API call failed","[]","[]"
13,conv-007,user,"Install antivirus please","[]","[]"
14,conv-007,atom,"Automated installer ran","[]","[]"
15,conv-008,user,"Printer shows paper jam","[]","[]"
16,conv-008,atom,"Driver install CTA:Install Driver","[Install Driver]","[{\"cta\":\"Install Driver\",\"timestamp\":\"2025-07-01T11:00:00+05:30\"}]"
17,conv-009,user,"Need Slack #general access","[]","[]"
18,conv-009,atom,"Granted via AD group sync","[]","[]"
19,conv-010,user,"WiFi keeps dropping","[]","[]"
20,conv-010,atom,"SSID mismatch, handing off","[]","[]"
21,conv-011,user,"Request Windows license","[]","[]"
22,conv-011,atom,"License key provisioned","[]","[]"
23,conv-012,user,"VPN latency issue","[]","[]"
24,conv-012,atom,"Optimized route via script","[]","[]"
25,conv-013,user,"Change SFDC role","[]","[]"
26,conv-013,atom,"Permission API error","[]","[]"
27,conv-014,user,"When is my invoice?","[]","[]"
28,conv-014,atom,"Invoice emailed to user","[]","[]"
29,conv-015,user,"Spam filter blocking me","[]","[]"
30,conv-015,atom,"Policy conflict noted, handing off","[]","[]"
31,conv-016,user,"DNS issues on VPN","[]","[]"
32,conv-016,atom,"Cache cleared, connection restored","[]","[]"
33,conv-017,user,"Expense report form error","[]","[]"
34,conv-017,atom,"Form schema failed, raising request","[]","[]"
35,conv-018,user,"Install Office suite","[]","[]"
36,conv-018,atom,"Deployment automation succeeded","[]","[]"
37,conv-019,user,"Network printer unreachable","[]","[]"
38,conv-019,atom,"Printer offline, handing off","[]","[]"
39,conv-020,user,"VPN cert expired","[]","[]"
40,conv-020,atom,"Cert renewed automatically","[]","[]"
```

**tickets.csv**

```csv
ticket_id,conv_id,created_at,resolved_at,assignment,sla_milestones,messages,status,resolution_notes
"sr-1001","conv-001","2025-07-01T09:16:00+05:30",,"[]","[]","","open",""
"sr-1003","conv-003","2025-07-01T09:30:00+05:30",,"[{\"timestamp\":\"2025-07-01T09:31:00+05:30\",\"assigned_by\":\"atom-runbook\",\"assigned_to\":\"L1-team\"}]","[{\"milestone\":\"ack\",\"timestamp\":\"2025-07-01T09:32:00+05:30\",\"breached\":false}]","[]","open",""
"sr-1005","conv-005","2025-07-01T09:45:00+05:30",,"[{\"timestamp\":\"2025-07-01T09:46:00+05:30\",\"assigned_by\":\"atom\",\"assigned_to\":\"HR-team\"}]","[{\"milestone\":\"approval\",\"timestamp\":\"2025-07-01T09:50:00+05:30\",\"breached\":false}]","[]","open",""
"inc-2006","conv-006","2025-07-01T10:00:00+05:30","2025-07-01T10:20:00+05:30","[{\"timestamp\":\"2025-07-01T10:01:00+05:30\",\"assigned_by\":\"atom\",\"assigned_to\":\"Network-team\"}]","[{\"milestone\":\"resolved\",\"timestamp\":\"2025-07-01T10:20:00+05:30\",\"breached\":false}]","[]","resolved","Fixed auth server config."
"sr-1008","conv-008","2025-07-01T11:05:00+05:30",,"[{\"timestamp\":\"2025-07-01T11:06:00+05:30\",\"assigned_by\":\"atom\",\"assigned_to\":\"Printer-support\"}]","[{\"milestone\":\"ack\",\"timestamp\":\"2025-07-01T11:07:00+05:30\",\"breached\":false}]","[]","open",""
"inc-2010","conv-010","2025-07-01T11:30:00+05:30","2025-07-01T11:50:00+05:30","[{\"timestamp\":\"2025-07-01T11:31:00+05:30\",\"assigned_by\":\"atom\",\"assigned_to\":\"Network-team\"},{\"timestamp\":\"2025-07-01T11:40:00+05:30\",\"assigned_by\":\"Network-team\",\"assigned_to\":\"L2-team\"}]","[{\"milestone\":\"escalation\",\"timestamp\":\"2025-07-01T11:45:00+05:30\",\"breached\":true}]","[]","resolved","Replaced faulty AP."
"sr-1013","conv-013","2025-07-01T12:00:00+05:30",,"[{\"timestamp\":\"2025-07-01T12...

```
