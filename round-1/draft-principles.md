# Draft: Coding principles answers (not committed)

## Order
1. TasCreed: design for failure, idempotent, DRY
2. FXCenter: protect downstream, observable (if time)
3. EasyFin: fail early, KISS, YAGNI

## TasCreed: design for failure

> A good example is the task framework we used to run our daily financial reports. I noticed that when a task failed, it kept retrying again and again. That could put a lot of pressure on whatever it depended on, and I couldn't stop it in time.
>
> I didn't want to just work around it in our own reports, so I fixed it in the framework. I made the retries wait longer each time, so they slow down instead of piling up. And I added an API to kill a task, so a person can always stop one that keeps retrying.
>
> What I learned is that slowing down the retries only limits the damage. It doesn't fix the cause. So I always want to leave people a way to step in.

## TasCreed: idempotent (if asked about re-runs)

> Tasks can run more than once, so each shard writes a fixed file name. If a task is re-run, it overwrites the same file, so we never get duplicates.

## TasCreed: DRY (keep it light, say "in a way")

> The retry problem was in the framework, so every report would have needed its own fix. I fixed it once in the framework, so every team benefits and nobody repeats the same workaround. That's DRY, in a way.

## How to answer "How do you make a system reliable?"

> I start from how it fails: limit the damage, retry safely, and make it visible.

| Point | Story | One line |
| --- | --- | --- |
| Limit the damage | TasCreed | backoff + kill API |
| Safe to retry | TasCreed | fixed file name, re-run overwrites |
| Visible | FXCenter | completeness metric + alert |
| Protect downstream | FXCenter | local cache, fall back to the previous full set |

## EasyFin: fail early, KISS, YAGNI

KISS = choose the simplest way (keep Excel, simple page check-out).
YAGNI = don't build what is not needed yet (no real-time collaboration).

> I built EasyFin so Finance could submit accounting configuration changes without all the emails. Three principles guided it.
>
> First, **fail early**. We validate when the user submits. So a typo, an invalid account, or a journal type that doesn't exist is caught right there, before anyone reviews it or an engineer sees it.
>
> Second, **keep it simple**. Finance was used to Excel, so I let them keep working in Excel and just import the file. And for editing, I used a simple page check-out: one person edits at a time.
>
> Third, **don't build what you don't need yet**. I could have built real-time collaboration, but we only had a small group of users, and they rarely edit the same thing at the same time. So I didn't build it. If more people need it later, we can add it then.
>
> The result was that changes were clearer, and we had much less back and forth.

Follow-ups:
- Why check-out? Few users, low chance of editing at the same time, simple to build. The cost is that others wait, so we added a timeout in case someone forgets to release it.
- What if you need real-time later? Build it then, with a real reason.
- To confirm: number of users (60 or under 50?). The script says "a small group of users".

## FXCenter: protect downstream (if time)

> Rate lookups were about ten times Checkout traffic, so the client reads from a local cache. If the data is incomplete, it falls back to the previous full set, with no retry loop, and a metric and an alert tell us quickly.

Note: the incomplete-data case only happened in staging. Say "we designed for it", not "it happened in production".

## Outline opening

> I focus on a few things: design for failure and idempotency, protecting downstream services, and failing early. I can give an example for each.

## Q: "What coding principles do you know and how do you put them into practice? Give examples."

Structure: name them (10 s) -> pick the ones I really use -> 3 short examples -> offer to go deeper.

> There are some well-known ones. KISS means keep it simple: choose the simplest design that fits the real need. YAGNI means don't build something until you actually need it. DRY means don't repeat yourself: keep one place for each piece of logic, so a fix happens once. There are also the SOLID principles for object-oriented design. On top of those, from payments work, I care about two things: fail early, and design for failure. Let me share how I used some of them.
>
> First, **fail early, KISS and YAGNI**. I built a tool called EasyFin for our Finance team to submit accounting configuration changes. We validate when the user submits, so a typo or an invalid account is caught right there, not after review. To keep it simple, I kept Excel, because Finance was used to it, and I used a page check-out so one person edits at a time. And I didn't build real-time collaboration, because we only had a small group of users who rarely edit the same thing at once.
>
> Second, **design for failure and DRY**. In our task framework, a failing task kept retrying again and again, and I couldn't stop it. Instead of working around it in our own reports, I fixed it once in the framework: retries wait longer each time, and there is an API to kill a task. Tasks can also run more than once, so each shard writes a fixed file name, and a re-run just overwrites it.
>
> Third, **protect the downstream**. For FX rate lookups, which had about ten times the traffic of checkout, the client reads from a local cache. If the data is incomplete, it falls back to the previous full set, and an alert tells us.
>
> I can go deeper into any of these.

Notes:
- Say SOLID only as a name. Don't list all five unless asked (see the SOLID table in the chat: O and D are the ones with something to say).
- If time is short, drop the third example.
- No FX decimals story here; keep it for the production incident question.

## Notes
- TasCreed: say "I fixed it in the framework", not "I designed the framework". The portal was built by the team.
- Don't claim a max retry count. It is not confirmed.
- EasyFin: don't say the whole process is automated. The handoff and SQL generation are automated. Engineers still approve.
