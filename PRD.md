# Insights - PRD

#### Created by Gautham Menon

#### Category

#### Last updated time

## Product Requirements Document (PRD)

#### Project :  Insights v1.0  St ory‑Driven Analytics for Atom Conversations, Requests &

#### Incidents

#### Author  Gautham Menon

#### Date  30 June 

#### Target Release  Q4 FY 

## 1. Problem Statement

#### Administrators and executives using Atomicwork struggle to see where Atomʼs

#### automation ends and human inefficiency begins. Traditional dashboard widgets

#### surface raw metrics (ticket volume, MTTR, FRT, etc.) but do not tell a coherent

#### story that pinpoints bottlenecks (e.g., “VPN incidents stall 45 min in L1ˮ). W e need

#### an insight layer that shows evidence, not prescriptions , so teams can act with

#### confidence.

## 2. Goals & Non‑Goals

```
Goal Success Metric Non‑Goal
G1. Surface story‑based insights
broken into Act 1 (automation) & Act 
(human workflow).
```
```
 80 % of beta users
agree “I understand where
time is lostˮ (survey).
```
```
Replacing existing
operational metric
reports.
```
#### June 30, 2025 105 PM


```
G2. Enable drill‑down from high‑level
insight to raw artefacts (chat log,
runbook, ticket timeline).
```
```
90 % of insights have a
working deep‑link.
```
```
Building a
full‑fledged BI tool.
```
```
G3. Provide insight packs tailored to
the buyer journey: POC , Go‑Live ,
30‑Day, Quarterly Business Review.
```
```
Adoption of at least 2
packs per customer by
Day 30.
```
```
Creating custom
packs per tenant.
```
```
G4. Support three journey genres in v1
Incidents , Service Requests , Other
Kno wledge/HR/Finance).
```
```
All three genres shown in
Story Gallery and generate
at least one Act 1  Act 
flow.
```
```
Change/Project
modules (deferred).
```
## 3. Personas

####  Executive  Needs quarterly metrics & proof points to re‑allocate budget.

#### Time‑poor, skims for red flags.

####  Admin ITSM lead)  Gathers data, presents a story, owns remediation plans.

#### Requires granular evidence.

####  Support Agent / Team Lead  Less frequent consumer; drills into a single

#### insight to improve queue performance.

## 4. User Stories (MVP)

```
# As a ... I want to ... So that ...
Acceptance
Criteria
```
```
1 Admin
```
```
open the Story
Gallery and
select
“Incidentsˮ
```
```
I see insights
only about
unplanned
interruptions
```
```
Card grid shows
3 genres;
“Incidentsˮ lo ads
genre‑specific
dashboard.
```
```
2 Admin
```
```
view Act 
Sankey of an
incident flow
```
```
I can tell if Atom
attempted
automation &
why it failed
```
```
Nodes: Intent ▶
Automation
success/failure
▶ Out come.
Hover shows
counts & %s.
```

```
3 Admin
click “Handed
offˮ segment
```
```
I jump to Act 
timeline for the
same cohort
```
```
Deep‑link filters
ticket list and
highlights first
assignment time.
```
```
4 Executive
```
```
download a PDF
export of top 5
insights for QBR
```
```
I can attach it to
a slide deck
```
```
Export includes
charts, narrative
text, and
methodology
footer.
```
```
5 Admin flag an insight
as “actionedˮ
```
```
the team can
track
remediation
status
```
```
Toggle adds
resolved tag;
insight
disappears from
default list.
```
## 5. Functional Requirements

### 5.1 Data Model

```
Entity Key Fields Source
```
```
ConversationIntent id, timestamp, user_id, intent_label,
confidence, channel
```
```
Atom NLU service
```
```
AutomationAttempt id, conversation_id, workflow_id, status
(success
failure), failure_reason
```
```
Ticket
```
```
id, type INC/SR/OTHER, priority, created_at,
closed_at, sla_breach, initial_group,
final_resolver
```
```
Request/Incident
micro‑service
```
```
AssignmentHop ticket_id, from_group, to_group, hop_start,
hop_end
Assignment service
```
### 5.2 Insight Generation Engine

#### Batch job every 6 h creates InsightInstance rows, grouping data by genre,

#### period, and theme.

#### Each instance stores: title, genre, act1_metrics JSON, act2_metrics JSON,

#### evidence_links (array of URIs).


#### Job should be idempotent; supports backfill for historical periods  90 days.

### 5.3 APIs

```
Method Path Description
GET `/insights?genre={incidents sr
GET /insight/{id} Full payload incl. Act 1 & Act 2 data objects.
GET /insight/{id}/export?format=pdf Generate on‑demand PDF.
PATCH /insight/{id} Update attributes (e.g., status=actioned).
```
### 5.4 UI Components

####  Story Gallery  R eact grid, card per genre with real‑time count of open

#### insights.

####  Insight Detail

#### Header : title & period selector.

#### Act 1 Panel  Sank ey / funnel + intent table.

#### Act 2 Panel : queue timeline Gantt), SLA dashboard, escalation Sankey.

#### Moments that Matter sidebar.

####  Export Modal  Choose PDF or CSV.

#### Figma references will be provided by Design prior to DEV start.)

## 6. Non‑Functional Requirements

```
Area Requirement
Performance Insight list  500 ms; detail page  2 s P95 with 90‑day data window.
Security Row‑level tenancy isolation; PDFs water‑marked with tenant id.
Scalability Support 250 K tickets & 500 K conversations per tenant / quarter.
```
```
Observability
Instrumentation: job duration, failed job count, API p95, front‑end error
rate.
Accessibility WCAG 2.1 AA; Sankey charts keyboard‑navigable with alt‑descriptions.
```

## 7. KPIs & Success Metrics

```
Metric Target after 90 days GA
Active tenants viewing Insights weekly  30 % of paying tenants
Average insight → drill‑down click‑through  40 %
Reduction in mean SLA breach rate among tenants using
insights
```
```
‑10 % relative to control
cohort
CSAT for Insights feature (in‑product poll)  4 .0 / 
```
## 8. Milestones

```
Phase Date Deliverable
Tech‑Spec & API contract 15 Jul Reviewed & signed‑off
Data model & ingestion jobs 15 Aug Deployed in staging
Front‑end Story Gallery  Act 1 views 15 Sep Feature‑flagged
Act 2 panel + deep‑linking 15 Oct Beta open
Export & “Momentsˮ sidebar 15 Nov Code‑complete
GA & rollout to 100 % tenants 15 Dec Release notes
```
## 9. Dependencies / Assumptions

#### Reliable NLU meta‑data (intent, confidence) exposed by Atom.

#### Assignment service publishes hop events with timestamps.

#### SLA calculator exists; we will consume its summary table.

#### Design to finalize Sankey & Timeline visual spec by 1 Aug.

## 10. Risks & Mitigations

```
Risk Likelihood Impact Mitigation
Inconsistent NLU
confidence scores across
languages
```
##### M H

```
Normalize scores; fall back
to “unknownˮ int ent
bucket.
```

```
PDF generation heavy on
memory for large tenants
```
##### M M

```
Streamed render, 100 page
limit; prompt user to slice
period.
Data privacy concerns
over exposing raw chat
logs
```
##### L H

```
Deep‑links respect
role‑based access checks.
```
## 11. Out of Scope (v1)

#### Change management insights CHG

#### Cross‑tenant benchmarking

#### Real‑time 5 min) streaming updates in UI

#### Alerting / notification engine for new insights

## 12. Glossary

```
Term Definition
```
```
Act 
All events occurring before a ticket is created (self‑service &
automation).
```
```
Act 2 Events occurring after ticket creation (assignment, human workflow,
resolution).
InsightInstance Persisted analytic object containing narrative, metrics & evidence.
Moment that
Matters
```
```
Auto‑detected sub‑step causing largest delay or cost in a given
story.
```
### Appendix – Key Fields Reference

#### Detailed lists of Act 1 and Act 2 metrics, as well as journey‑specific milestone

#### tables, are included in the source document.

#### Requested Action  Engineering leads to review the data model and API contract

#### sections by 15 July and surface sizing or feasibility concerns in the Insights

#### #tech‑spec channel.


