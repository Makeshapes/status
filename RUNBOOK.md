# Status Page Runbook

How to use [https://status.makeshapes.com](https://status.makeshapes.com) and this repository during incidents and maintenance windows.

Upstream docs: [upptime.js.org](https://upptime.js.org/docs/) — refer for anything not covered here.

---

## What this repo is

- **Source of truth for the public status page** at `https://status.makeshapes.com`.
- **Automated uptime probes** run from GitHub Actions every 5 minutes against 5 production endpoints (see `.upptimerc.yml`).
- **Incidents are GitHub Issues** in this repo. Open an issue → it renders on the public page. Close it → it moves to history.
- **Notifications fire to Slack `#status`** on state change (down / back up).

This repo is **separate from `Makeshapes/infrastructure`** by design — it must keep working when prod is down. Only the DNS record (`status.makeshapes.com` CNAME → `makeshapes.github.io`) lives in the infra repo.

---

## Posting an unplanned incident

Use this when something is broken and customers should know.

```sh
gh issue create \
  -R Makeshapes/status \
  -t "<short customer-facing title>" \
  -l "incident,<service-slug>" \
  -b "<initial customer-facing description>"
```

`<service-slug>` is the same slug used in the auto-rendered status table. Options today:
- `api-apiv2`
- `yjs-collab`
- `socket-io-sessions`
- `experience-app`
- `creator`

You can omit the service label if the issue is global / multi-service.

### Body conventions

Write for customers, not engineers. No internal hostnames, no ARNs, no stack traces.

```markdown
We're currently investigating an issue affecting <service>. Users may experience <symptom>. We'll update this issue with progress.

**Started:** YYYY-MM-DD HH:MM UTC
```

### Posting updates

Add a comment to the issue. Each comment shows on the public page in chronological order.

```markdown
**Update (YYYY-MM-DD HH:MM UTC):** Identified the cause as <high-level cause>. Engineering is deploying a fix.
```

### Resolving

When the customer-visible problem is gone:

1. Post a final comment with a resolution time and short summary.
2. Close the issue: `gh issue close <n> -R Makeshapes/status`.

Upptime auto-moves closed incidents to the history section of the page.

---

## Scheduled maintenance

For planned windows where a service will be intentionally degraded or offline.

Open an issue with **label `maintenance`** and a fenced YAML block in the body declaring the window:

```sh
gh issue create \
  -R Makeshapes/status \
  -t "Scheduled maintenance: <what>" \
  -l "maintenance,<service-slug>"
```

Body:

````markdown
We will be performing scheduled maintenance on <service>. <Short description of what changes and what users should expect.>

```yaml
- name: <Service display name>
  url: https://<endpoint>
  date: YYYY-MM-DD HH:MM
  duration: 30m
```
````

Upptime parses the fenced block and surfaces the window on the page. Multiple sites can be listed in the YAML block.

Full schema and worked examples: <https://upptime.js.org/docs/scheduled-maintenance>.

---

## Severity guidance

| Severity | When | Who pages | Open status incident? |
|---|---|---|---|
| **SEV-1** | Customer-facing outage of `api`, `app`, or `creator`. Multiple services down. | PagerDuty (New Relic alert path) | **Yes**, immediately. |
| **SEV-2** | One service degraded but mostly working. `yjs` or `socketio` down. | PagerDuty | Yes, once confirmed not transient. |
| **SEV-3** | Single endpoint slow or intermittent. Internal-only impact. | Slack `#alerts` | Optional — only if customers will notice. |
| **Maintenance** | Planned change with announced window. | n/a | Yes, with `maintenance` label and YAML schedule. |

The status page is **customer comms**, not engineering visibility. CloudWatch and New Relic remain the source of truth for internal observability. Don't open status incidents for things customers can't see.

---

## Operating tips

- **Don't add internal endpoints** (NLB hostnames, RDS endpoints, bastion) to `.upptimerc.yml`. This repo is public.
- **Don't rename the `master` branch.** Upptime workflows hard-code it.
- **PAT rotation:** the `GH_PAT` secret backing the upptime workflows is owned by an individual GitHub account. Rotate before expiry; move to a machine user or GitHub App when convenient.
- **Slack webhook rotation:** the `NOTIFICATION_SLACK_WEBHOOK_URL` secret can be re-issued from the Slack app at `https://api.slack.com/apps` → Incoming Webhooks. Recreate the webhook for `#status`, update the secret, no other changes needed.
- **Re-trigger after manual edits:** `gh workflow run "Uptime CI" -R Makeshapes/status`. State changes auto-trigger Static Site CI which republishes the page.
- **Adding a new monitored site:** edit `.upptimerc.yml`, commit, push. Setup CI regenerates workflows automatically; Uptime CI picks up the new site on the next run.
- **Maintenance-mode integration** (auto-open issue when the maintenance flag flips in our app stack): not yet built. Tracked in `Makeshapes/infrastructure#520` and `#521`. Until then, scheduled maintenance is fully manual here.

---

## Related references

- Public page: <https://status.makeshapes.com>
- Design doc: [`Makeshapes/infrastructure/docs/tech-debt/status-page.md`](https://github.com/Makeshapes/infrastructure/blob/main/docs/tech-debt/status-page.md)
- Cloudflare DNS record: [`live/cloudflare/cloudflare.tf`](https://github.com/Makeshapes/infrastructure/blob/main/live/cloudflare/cloudflare.tf) (resource `github_pages_status`)
- Upstream upptime docs: <https://upptime.js.org/docs/>
- Upptime incident reports: <https://upptime.js.org/docs/incident-reports>
- Upptime scheduled maintenance: <https://upptime.js.org/docs/scheduled-maintenance>
