# Interview Cards

One short card per project. Use it for "coding principles with example" and behavioral questions. Full stories live in `../project-coding-examples/`.

## EasyFin

Values: **Aspire** (main: initiative, beyond my scope), **Excel** (fail early, quality).

| | |
|---|---|
| **Why** | A Finance user wrote SFX 1.3 instead of BFX 1.3 (seller vs. buyer journal type). It took several rounds to find. |
| **My action** | On my own initiative, nobody asked me to, I built EasyFin alone: one web UI to submit, review, and approve changes. |
| **Adoption** | Finance still edits in Excel. Only the place to submit changed: EasyFin instead of Google Drive and email. They had a reason to switch, because mistakes are caught at submission. After launch they file a ticket for new needs, and we add reasonable ones as enhancements. |
| **Principle + example** | Fail early. Validate at submission, so a wrong journal type is caught right away. |
| **Result** | All 60 Finance users use it. Apart from project meetings, no extra communication about requirements. |

Keywords: `BFX/SFX typo -> build EasyFin alone -> keep Excel, change submit place -> validate at submit -> tickets -> 60 users`

90-second version:
> Finance used to send requirements by Excel, Google Drive and email. Once, someone wrote SFX 1.3 instead of BFX 1.3, a seller journal type instead of a buyer one, and we needed several rounds to find it. Nobody asked me to fix the process, but I saw it would keep happening. So, on my own initiative, I built EasyFin, a web tool where Finance submits and reviews changes. The key idea was to validate at submission, so a wrong journal type is caught right away. I didn't ask Finance to change how they work. They still edit in Excel, and only the place to submit changed. After launch, they file a ticket for new needs, and we add reasonable ones. Today all 60 Finance colleagues use it, and apart from project meetings, we have no extra communication about requirements.

Follow-up answers:
- **How did you get Finance to use it?** I kept their habit: they still edit in Excel, they only submit to EasyFin instead of Google Drive and email. They also saw the benefit at once, because typos and invalid data were caught at submission. After launch, they file tickets and we add reasonable requests.
- **Link to the Payment Graph story:** same idea, make it easy for others by lowering their cost.
- **Why not ask a team to build it?** It was not a formal PM requirement, so I could not assign the work to others, and waiting might mean it never started. It was a quiet period at the end of the year, so I could build a first version without affecting my regular projects. I also wanted to build a product from scratch. *(Not confirmed: that the team helped maintain it after the first version worked, and whether I demoed a prototype to Finance.)*
- **Hardest part?** Not technical: understanding what Finance really needed and making the workflow easy to use. There was no PM, so I talked to Finance users directly and confirmed the workflow with them. Example: my first design made users check out the page to lock it and check in after submitting. Some forgot to check in, so nobody else could submit. After their feedback, I added a timeout so the page unlocks automatically. Technical core, only if asked: keep Excel and the database in sync (users mark changed rows, everything else must match, compare and validate before submit).
- **Link:** a lock needs an expiry, because the holder may disappear (same idea as a lease).

Careful:
- Integration automated the handoff and SQL generation, not the approval. Engineers still review the SQL before staging. Do not say "the whole process is automated".
- After launch, say "we" for maintenance. The first version was built by me alone.
- No numbers except 60 users. If asked for time saved, say we did not measure it.
- I have no example of a specific ticket yet. Add one if I can recall it.

Questions it can answer:
- Behavioral: take initiative / improve a process / work with non-technical users / get adoption.
- Principle: fail early, make changes auditable.
- Details if asked: five features (UI, history, recon, validation, integration), see `../project-coding-examples/easyfin-story.md`.

## Failure: changed a shared function

| | |
|---|---|
| **Situation** | First year at eBay. I was asked to enhance a database query function. It looked like a simple task. |
| **Mistake** | I modified the existing function in place instead of adding a new one. I did not check who else used it. |
| **Result** | Another application using the same function raised an alert after release. It was a false alert, but my change caused it, and I wrote the RCA. |
| **Fix** | Rolled back the release, then added a new function and released again. |
| **Lesson** | Think about backward compatibility first. For shared code, add a new function and leave the old one unchanged. Never change something I don't fully understand. |

Keywords: `shared query function -> changed in place -> other app alerted -> rollback -> add new function`

90-second version:
> In my first year at eBay, I was asked to enhance a database query function. It looked simple, so I modified the existing function directly instead of adding a new one. It worked for the new API, but I didn't check who else used it. After the release, another application that shared the function raised an alert. It turned out to be a false alert, but my change had caused it, and I was asked to write an RCA. We rolled back, then released again with a new function and the old one untouched. Since then, I always think about backward compatibility first. For shared code, I add a new function rather than change the old one, and I never change something I don't fully understand.

Notes:
- Say "my mistake" clearly. Do not blame others.
- I don't remember why exactly the alert fired. Say so honestly if asked; focus on the fix and the lesson.
- Still to add if I can recall: how I check callers today, and a later time this habit prevented a problem.

## Cross-team dependency: slow owner of the payment graph

Use for: priority conflict between teams, working with other teams, influence without authority, taking ownership, a project at risk of delay.
Values: **Unite** (main), **Excel** (ownership, trade-off).
If asked about a "disagreement", say it was a priority conflict, not a direct argument.

| | |
|---|---|
| **Situation** | I led the in-house Buyer FX project. Part of the change was in the payment graph, owned by another team. The engineer there had other projects, so FX was not his priority, and our deadline was at risk. |
| **Action 1** | I talked to him one-on-one and asked to raise the priority. He agreed to do it next sprint, but after a sprint progress was still slow. |
| **Action 2** | I saw the problem was his time, not his willingness. With my manager's approval, I handed my test preparation to a colleague (one hour of requirement sync plus wikis) and spent two days studying the code to work out the exact changes. I met him again and he confirmed the plan. He still made the change himself. |
| **Result** | With a clear plan he finished in three days. Development was done before E2E testing and E2E started on time. He thanked me afterwards. |
| **Cost** | Test automation was ready two days later than planned. Testing was not on the critical path, and we ran the first tests manually. |
| **Lesson** | A request alone is often not enough. Lower the other team's cost: give specifics, so it is easy to say yes and finish. |

Keywords: `payment graph dependency -> 1:1 priority talk -> still slow after a sprint -> I free two days, study code -> exact plan -> done in 3 days -> E2E on time`

90-second version:
> I led the in-house Buyer FX project. Part of the change was in the payment graph, which another team owned. The engineer there had other projects, so FX was not his priority, and our deadline was at risk. First, I talked to him one-on-one and asked him to raise the priority. He agreed, but after a sprint, progress had not really changed. I realized the problem was not willingness, it was his time. So, with my manager's approval, I handed my test preparation to a colleague, and spent two days studying the code to work out exactly what to change. Then I met him again, and he confirmed the plan. He still made the change himself, but with a clear plan he finished in three days, and we completed development before E2E testing. What I learned is that asking is often not enough. If I make it easier for the other team to help, they can.

Follow-up answers:
- **Why not escalate?** Escalation is a last resort. His problem was lack of time, not refusal, so escalating would not give him more time, and it could hurt the relationship. I tried to remove the blocker myself first. If it was still blocked close to the deadline, I would raise it openly with my manager. *(Confirm this is how I would really act.)*
- **What would you do differently?** Act earlier. I waited a full sprint before I saw nothing changed. Next time I would check in after a few days, go to the first conversation with a concrete plan, and align priorities with other teams at project kickoff.
- **Did handing off the test work cause problems?** A small cost. My colleague needed time to learn the requirements, so test automation was two days late. I chose this on purpose, because testing was not on the critical path while the payment graph change was. I spent one hour on a requirement sync and shared wikis. E2E started on time, and we tested manually until automation caught up.

Notes:
- The owner stayed responsible for the change. I only gave the plan.
- I did not escalate. I first tried to remove the blocker myself.
- "After a sprint" and "three days" come from my own recollection. Keep them consistent.

## Why am I leaving / Why Checkout.com

**Why leaving** (one sentence, calm, no complaints):
> "eBay closed its China team because of geopolitical reasons, so my whole team was dissolved. It was not about performance, and it gave me a chance to choose what I want to do next."

**Why Checkout.com** (3 reasons, all real):
1. **Complementary experience.** At eBay I worked on managed payments. Checkout.com is a real processor, so I would see the other side of payments.
2. **I have worked with you.** I led the Buyer FX implementation where payments went through Checkout.com, as a complement to Adyen. I know the integration from the merchant side.
3. **Strong impression.** Across several processors I worked with, Checkout.com support was always fast and effective at solving problems. I want to work in a team with that standard.

Keywords: `layoff (China team) -> processor side -> led CKO Buyer FX -> fast, effective support`

60-second version:
> Unfortunately, eBay closed its China team for geopolitical reasons, so my whole team was dissolved. It was not about performance, and it let me think about what I want next. I'd like to join Checkout.com for three reasons. First, at eBay I worked on managed payments, and Checkout.com is a real processor, so my experience would be complementary. Second, I've worked with you before. I led the Buyer FX implementation that routes payments through Checkout.com, alongside Adyen. Third, I've worked with several processors, and Checkout.com's support was always fast and good at solving problems. That made a strong impression on me, and I'd like to work in a team like that.

Notes:
- Keep the layoff answer short and neutral. Move on to the positives quickly.
- Do not compare or criticize other processors by name.
- Be ready for: "What do you know about our product / tech?" Add 2 or 3 facts from the company website or blog before the interview.

## Questions to ask the interviewer

Ask what you really want to know. Pick 3 per round. Listen, then ask one follow-up. Do not ask about salary or things on the website.

**Engineers / technical round**
1. What does a typical project look like for this team, and what are the hardest technical challenges right now?
2. How does the team handle reliability for payments, for example when a bank or scheme is unavailable?
3. How do engineers decide between building something new and improving an existing system?

**Manager / HR round**
4. How do teams work across time zones, and how do you keep communication smooth?
5. What would success look like in the first 6 months for this role?
6. What do you like most about working here?

**Use when it fits (my background)**
7. I worked with Checkout.com as a merchant at eBay. How does the team use merchant feedback to shape the product?

## Fix it at the root: endless retries in a shared framework

Use for: initiative, ownership, improving a shared tool, impact on other teams.

| | |
|---|---|
| **Situation** | At eBay, we used an internal task framework (TasCreed) to run daily financial reports. While debugging, I noticed some tasks retried endlessly and I could not stop them in time. |
| **Action** | I did not just work around it in our reports. I fixed it in the framework: added exponential backoff, and an API to kill a task that keeps retrying. |
| **Result** | Runaway retries were limited and could be stopped quickly. As more teams used the framework, the team built a portal with permission control to see running jobs and kill stuck ones in one place. I have no numbers for this. |
| **Lesson** | Fix problems at the root so everyone benefits. Backoff only limits the damage, so leave people a way to step in. |

Keywords: `endless retry -> fix framework, not workaround -> backoff + kill API -> portal for all teams`

90-second version:
> At eBay, we used an internal task framework to run our daily financial reports. While debugging, I noticed that some tasks retried endlessly, and I had no way to stop them in time. I could have worked around it in our own reports, but the problem was in the framework, so I fixed it there. I added exponential backoff, so retries slow down instead of hammering the dependency. I also added an API to kill a task that keeps retrying, so people always have a manual override. As more teams started using the framework, the team built a portal with permission control, to see running jobs and kill stuck ones in one place. What I learned is to fix problems at the root when I can, and to always leave humans a way to step in.

Notes:
- Say "the team built a portal". My exact role in the portal is not confirmed, so do not claim it.
- I was a heavy user, not the original designer of the framework (the open-source author is Lionel Liu). Say this if asked.
- No impact numbers. If asked, say honestly that we did not track them.
- Optional: add one concrete example of a task that retried endlessly, if I can recall one.
