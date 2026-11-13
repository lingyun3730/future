# Interview Cards

One short card per project. Use it for "coding principles with example" and behavioral questions. Full stories live in `../project-coding-examples/`.

## EasyFin

| | |
|---|---|
| **Why** | A Finance user wrote SFX 1.3 instead of BFX 1.3 (seller vs. buyer journal type). It took several rounds to find. |
| **My action** | On my own initiative, I built EasyFin alone: one web UI to submit, review, and approve changes. |
| **Principle + example** | Fail early. Validate at submission, so a wrong journal type is caught right away. |
| **Result** | All 60 Finance users use it. Apart from project meetings, no extra communication about requirements. |

Keywords: `BFX/SFX typo -> build EasyFin -> validate at submit -> 60 users`

90-second version:
> Finance often sent requirements by Excel and email. Once, someone wrote SFX 1.3 instead of BFX 1.3, a seller journal type instead of a buyer one, and we needed several rounds to find it. I decided to fix it at the source. On my own initiative, I built EasyFin, a web tool where Finance submits and reviews changes. The key idea was to validate at submission, so an invalid journal type is caught immediately. Today all 60 Finance colleagues use it, and apart from project meetings, we have no extra communication about requirements.

Questions it can answer:
- Behavioral: take initiative / improve a process / work with non-technical users.
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

Use for: working with other teams, influence without authority, taking ownership. If asked about a "disagreement", say it was a priority conflict, not a direct argument.

| | |
|---|---|
| **Situation** | I led the in-house Buyer FX project. Part of the change was in the payment graph, owned by another team. The engineer there had other projects, so he was slow and our deadline was at risk. |
| **Action 1** | I talked to him one-on-one and asked to raise the priority of our work. He agreed, but progress was still slow. |
| **Action 2** | I saw the problem was his time, not his willingness. With my manager's approval, I handed my test preparation to a colleague and spent two days studying the code to work out the exact changes. He still made the change himself. |
| **Result** | With a clear plan he finished much faster. The change and testing were done on time. He thanked me afterwards. |
| **Lesson** | A request alone is often not enough. Lower the other team's cost: give specifics, so it is easy to say yes and finish. |

Keywords: `payment graph dependency -> 1:1 priority talk -> still slow -> I free two days, study code -> exact plan -> he executes -> on time`

90-second version:
> I led the in-house Buyer FX project. Part of the change was in the payment graph, which another team owned. The engineer there had other projects, so his progress was slow, and our deadline was at risk. First, I talked to him one-on-one and asked him to raise the priority. He agreed, but nothing really changed. I realized the problem was not willingness, it was his time. So, with my manager's approval, I handed my test preparation to a colleague and spent two days studying the code, to work out exactly what to change. He still did the change himself, but now he had a clear plan, and it went much faster. We finished the change and the testing on time, and he thanked me afterwards. What I learned is that asking is often not enough. If I make it easier for the other team to help, they can.

Notes:
- The owner stayed responsible for the change. I only gave the plan.
- I did not escalate. I first tried to remove the blocker myself.

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
