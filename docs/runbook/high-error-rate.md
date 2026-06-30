# High Error Rate

## What this alert means

QuickNotes returned more than 5% 4xx and 5xx responses for at least 5 minutes.

## Triage steps

1. Open the Grafana golden-signals dashboard and confirm the error ratio, traffic level, and current note count.
2. Check the QuickNotes container logs with `docker compose logs --tail=200 quicknotes` and look for repeated `400`, `404`, or `500` patterns.
3. Query Prometheus for `quicknotes_http_responses_by_code_total` and confirm whether the errors are mostly client-side `4xx` or server-side `5xx`.
4. Reproduce one failing request locally with `curl` so you know whether the failure is still active and which endpoint is affected.

## Mitigations

- If malformed client traffic is driving the alert, rate-limit or temporarily block the bad caller while healthy traffic keeps flowing.
- If the service is unhealthy, restart the Compose stack with `docker compose restart quicknotes` and verify `/health` and `/metrics` before sending more traffic.
- If a recent config change caused the issue, roll back to the last known-good commit and redeploy the stack.

## Post-incident

Write a blameless postmortem using the Lecture 1 format: what happened, why it happened, and what system changes will prevent the same page next time. Start from [lectures/lec1.md](/Users/tatyana/Documents/DevOps-Intro/lectures/lec1.md:332).
