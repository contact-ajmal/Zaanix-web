---
title: Alerts, snapshots and sharing out
order: 14
group: Work with data
description: SQL alerts and scheduled snapshots delivered to Slack, Teams, email, PagerDuty and webhooks; signed embeds in your own application; queries published as HTTP APIs; templates.
---

## Channels

**Dashboards → Channels** are where alerts, snapshots, quality results and monitors are delivered. A channel belongs to a workspace, or to the whole server (administrators).

| Type | What arrives |
|---|---|
| Slack | Block Kit: title, text, fields, the snapshot image, an *Open in ZAANIX* button |
| Microsoft Teams | An Adaptive Card through a Workflows or incoming webhook |
| Email | HTML and text through the server's SMTP settings; a snapshot inline, the PDF attached |
| PagerDuty | Events API v2: *trigger* and *resolve* with a stable dedup key |
| Webhook | JSON with an HMAC signature (`X-ZAANIX-Signature`) and a signing secret shown once |

Secrets are encrypted and write-only. Every URL passes an **egress guard**: https only, public addresses only, no redirects — checked at connection time, so there is no DNS-rebinding window. Deliveries are retried on network errors and 5xx, and logged.

## SQL alerts

**Dashboards → Alerts**: a read-only query, a condition and a schedule (every N minutes, cron with a time zone, or by hand).

- **Conditions**: *it returns rows*, *it returns none* (freshness), or a *threshold* on the first row's column.
- **States**: ok, triggered or error. A change to *triggered* is always delivered, with a sample of the rows; *resolved* when it recovers (PagerDuty closes the incident); an error once.
- The query runs as the alert's author with read access only, under their access policies. **Test** runs it once without saving or sending.

## Scheduled snapshots

**Dashboards → Snapshots**: a dashboard (grid or Mosaic) or a data app rendered by a headless browser on a schedule, as a PNG or a PDF, and delivered to channels — to email as an inline image and attachment, to Slack, Teams and PagerDuty through a signed link. The image ships Chromium for this; a render that fails is delivered with the reason.

## Signed embeds

**Settings → Embedding** puts a dashboard or a notebook inside your own application, with no ZAANIX sign-in for its viewers.

1. A workspace owner creates an **embed key** (its secret shown once) and the sites allowed to frame it.
2. For each page view, **your server** signs a short-lived HS256 token naming the object, the viewer and attributes such as `{"tenant": "acme"}`. Settings shows a Node.js and a Python function that does it.
3. The iframe loads `https://<zaanix>/embed/view?token=…`.

An embed can load that one object and nothing else. Every request is checked again, so revoking a key stops all its embeds. An [access policy](governance.html#access-policies) that applies to embeds filters rows per viewer: `tenant = {{embed.tenant}}`.

## Queries as HTTP APIs

Publish a read-only SELECT at `GET /q/<slug>`: callers pass `{{name}}` parameters in the query string (typed and inserted as literals) and get rows back as JSON, or CSV with `?format=csv`. Each endpoint has its own key (shown once) unless it is public, runs as its owner under their access policies, is rate-limited per endpoint and audited. Agents publish them with `publish_endpoint`.

## Templates

**Home → Templates** installs ready-made analytics — saved queries, dashboards, notebooks, metrics and quality checks — in one step: E-commerce sales, SaaS subscriptions, Web analytics and Support tickets are built in, with sample data. Installing maps the template's tables to yours and checks the columns first; a failed install removes what it created. Publish your own from a workspace, or move them as `.zaanix-template.json` files.
