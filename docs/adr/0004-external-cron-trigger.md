# ADR-0004: Trigger the newsletter from an external cron every minute

- **Status**: Accepted
- **Date**: 2026-06-05 (commit "trigger email sender every minute via cron job"); recorded 2026-11-17
- **Deciders**: Ridwane777

## Context

Each user picks a delivery time. n8n ran on a free Render web service, which sleeps after inactivity, so an
internal Schedule Trigger does not fire while the instance is asleep.

## Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| n8n Schedule Trigger every hour | No external service | Misses runs while the host sleeps; hour precision only |
| n8n Schedule Trigger every minute | Minute precision | Same sleep problem |
| **External cron (cron-job.org) calling a webhook every minute** | Wakes the host and triggers B; minute precision; free | 1,440 executions a day, mostly doing nothing; unauthenticated public webhook |
| Paid always-on host + Schedule Trigger | Simple and reliable | Monthly cost |
| Queue per user (send at exact time with a delay job) | No polling | Needs a job queue n8n does not provide simply |

## Decision

Workflow B starts with `Webhook Trigger` on `POST /webhook/trigger-newsletter`, called every minute by
cron-job.org. `Check schedule` compares the current `HH:MM` in the user's time zone with `schedule_time`.

## Consequences

- ✅ Delivery at the minute chosen, and the free host stays awake.
- ❌ If one call fails or arrives late, that minute's users get no newsletter (no catch-up).
- ❌ Anyone can call the webhook; a shared secret header is needed.
- ❌ `Check schedule` maps `UTC` (the default) and `America/New_York` to `Africa/Lagos` (UTC+1), while the
  chatbot asks for a time in UTC: a user who asks for 08:00 UTC receives the email at 08:00 Lagos time, 07:00
  UTC. A New York user is off by five to six hours. This is a known bug.
