# Mock Interview Round 1 (Checkout.com behavioral)

Format for each question: **My answer** (as I said it, unedited), then **Revised answer** (same facts, clearer structure), then short notes.

Rules for the revised answers:
- Only facts I confirmed. `[TODO]` means a fact I still need to fill in. Never invent it.
- Say "Checkout.com", not "CKO", in the interview.
- Story cards with more detail: `interview-cards.md`.

---

## Q1. Tell me about yourself, and why you want to join Checkout.com

### My answer

> Well, thanks for taking time to talk with me. My name is XXX, and I am a software engineer with over 7 years' working experience in eBay payments department. I worked across most of the payment system: the payment graph which handles the transactions at checkout, the payment gateway which sends the transactions to external processors like Adyen, CKO, Paypal etc., the data gateway that processes the reports and webhooks, and the accounting system which books journals for each transaction and is also the source of truth for accounting. In recent years, I've focused myself working on the fx platform. I led several FX projects from requirements to production, working closely with PMs, BUs and other stakeholders. One of them extended buyer fx to 42 additional countries and regions. In my spare time, I go to the gym regularly and enjoy playing table tennis with my friends. As to why I want to join Checkout.com, firstly, I have the payments working experience and this position suits for me. Secondly, I know CKO is a reliable partner because the collebration with CKO was very smooth when I was leading the CKO buyer fx project.

### Revised answer

> Thanks for taking the time to talk with me. I'm Lintao, a backend engineer with over seven years of experience in Payments at eBay.
>
> I've worked across most of the payment system: the Payment Graph that handles transactions at checkout, the gateway that sends them to external processors like Adyen and Checkout.com, the data gateway that processes webhooks and reports, and the accounting system, which books every transaction as journals and is the source of truth.
>
> In recent years I've focused on the FX platform. I led several FX projects from requirements to production, working closely with product managers and business partners. One of them extended Buyer FX to 42 additional countries and regions. Outside work, I go to the gym and play table tennis.
>
> I'd like to join Checkout.com for three reasons. First, at eBay I worked on managed payments, and Checkout.com is a real processor, so I'd see the other side of payments, and my experience would be complementary. Second, I've worked with you before: I led the Buyer FX project that routes payments through Checkout.com, so I know the integration from the merchant side, and the collaboration was smooth. Third, `[TODO: one real fact about Checkout.com's product, tech or culture]`. I'd like to work in a team like that.

### Notes

- Reason 1 ("my experience suits this role") is too generic. Use "the other side of payments".
- Put the hobby at the end of the intro, then use a clear transition into "why".
- The processor list in my card is Adyen, Checkout.com, Alipay. Use PayPal only if it is true.
- Fill in reason 3 from the website or blog before the interview.

---

## Q2. Tell me about a time you disagreed with a stakeholder (the currency expansion)

### My answer

> I'd take buyer fx add 42 new currencies as an example. The PM wanted to skip the E2E testing because he thought the buyer fx was there and this was just a currency extension which required only adding the currency configuration. I pointed out the real risk if we skip the E2E testing based on my knowledge and understanding of the current implementation. Some components might hardcoded the currencies and some currencies might be missing in their configuration. I performed a batch of tests very quickly in staging, it turned out 25 new currencies were not configured in a component so that the payments could not go through. This was just the part of the FMM testing, and we also had RMM flows. I showed the results to the PM and he agreed to conduct the E2E testing. The testing results proved my judgement - three components had not configured the full currency list.

### Revised answer

> When we expanded Buyer FX to more countries, the PM thought E2E testing could be skipped. The feature already existed, so it looked like only a currency configuration change.
>
> I disagreed. From my knowledge of the system, I was worried that some components might have the supported currencies hard-coded, or be missing some currencies in their configuration.
>
> I didn't argue about opinions. I ran a batch of tests in staging, and quickly found that 25 currencies were not configured in one component, so payments couldn't go through. That was only the payment flow. The refund flow was still untested.
>
> I showed the results to the PM, and he agreed to add E2E testing. We spent two weeks on it. The refund flow had the same problem, and in total three components had to be upgraded to support the full currency list.
>
> What I learned: when I disagree about risk, evidence works better than opinion.

### Notes

- Do not say "FMM / RMM". Say "payment flow / refund flow".
- Do not say "it proved my judgement". Say "the results showed the risk was real".
- Fix the number: is it 42 countries and regions, or 42 currencies? Use the same number as in Q1. `[TODO: confirm]`
- Do not mix: 25 currencies were not configured (payment flow); three components were upgraded (payment and refund).
- If the PM did not literally say "skip", use "thought it was simple, so E2E might not be needed".

---

## Q3. Tell me about a time you made a mistake at work

### My answer

> When I was a junior engineer, I was assigned a task to enhance a db query function. It looked simple so I modified the existing function directly instead of creating a one. It worked well on the new API so I released it. But another application reported an alert right after the release. It turned out a false alert but I still rolled back the change to stop the alerting. Because it was my change which caused the alert, I was asked to write an RCA. Then I realized I had missed the check who else also referenced the changed function. I should create a new function rather than directly make the change to the old one before I fully understand the full risk. Furthermore, I learned I should always think about the backward compatibility when I make any change.

### Revised answer

> In my first year at eBay, I was asked to enhance a database query function. It looked simple, so I modified the existing function directly, instead of adding a new one. It worked for the new API, and I released it.
>
> Right after the release, another application that shared the function raised an alert. It turned out to be a false alert, but my change had caused it. I rolled back the release to stop the alerts, and I was asked to write the RCA. The cause was clear: I hadn't checked who else used the function.
>
> We released again with a new function and the old one untouched. Since then, before I change shared code, I check who else calls it, and I prefer adding a new function over changing an old one. `[TODO: optional, one later example where this habit prevented a problem]`

### Notes

- Own the mistake clearly. Do not blame others.
- I don't remember why exactly the alert fired. Say so honestly if asked.
- The lesson must be a specific habit, not just "think about backward compatibility".

---

## Q4. Tell me about something you improved that nobody asked you to (EasyFin)

### My answer

> When I was working in the accounting engineering team, I found the whole feature delivery process involved many manual effort. Finance submitted the changed requirement file to google drive and emailed the engineer, and there was no check and validation on the changes. Upon receiving the requirement change, engineers were supposed to check and understand the changes first. Sometimes we detected the typos or invalid data, this required a few rounds of communication with finance. I noticed this would continue if nothing got changed, so I designed and developed easyfin - a web UI for finance folks to submit and review their changes. Firstly, it preserved the finance's working habit on excel edit. Second, it helped detect any typos and invalid data when they submit the changes to easyfin. We also added customized validations per finance needs. Thirdly, it helped on the compliance audits as well. Nobody asked me to do that, I just wanted to make the work easier to improve the efficiency. This was not a formal requirements from PM end, and I completed the first version during the end of the year when there were not so many business projects.

### Revised answer

> In the accounting engineering team, Finance sent us requirements as Excel files through Google Drive and email, with no checks. Once, someone wrote SFX 1.3 instead of BFX 1.3, a seller journal type instead of a buyer one, and it took several rounds to find. I saw it would keep happening.
>
> Nobody asked me to fix it, and it wasn't a formal PM requirement. In a quieter period at the end of the year, I built the first version of EasyFin on my own: a web tool where Finance submits and reviews changes.
>
> Three design choices. First, I kept their habit: they still edit in Excel, and only the place to submit changed. Second, it validates at submission, so mistakes like BFX versus SFX are caught immediately, and we added custom validations for Finance's needs. Third, every change is tracked, which helps audits. `[TODO: confirm what the audit point means: history of changes?]`
>
> Today all 60 Finance colleagues use it, and apart from project meetings, we have almost no extra communication about requirements.

### Notes

- The result (60 users, almost no extra communication) was missing in my answer. Always say it.
- Use one concrete incident (BFX vs SFX) to open.
- Shorten the description of the old process to two sentences.
- Don't say "the whole process is automated". The integration automated the handoff and SQL generation, not the approval.

---

## Q5. Tell me about a time you delivered under a tight deadline (Payment Graph)

### My answer

> I'd take the inhouse buyer fx project as an example. When I was leading the inhouse buyer fx, a critical part was in the payment graph. However the engineer was overloaded with his projects and he had not prioritied the fx project. We were stuck and the deadline was at risk. I noticed that his problem was not willingness but time. I wanted to complete the dev work and kick off the E2E on time. So I spent 2 days working out the change plan to be made to the payment graph. I booked a 1:1 with the engineer and he comfirmed the plan. The change was still made by him but with a clear plan, he completed the task only within three days. This actually helped both of us and the delivery of the dev work was not delayed. So in this project, I just realized the risk in advance and quickly took the action to help resolve the blockers.

### Revised answer

> I led the in-house Buyer FX project. A critical part was in the Payment Graph, owned by another team. The engineer there was overloaded with his own projects, so FX was not his priority. E2E testing could not start until his change was done, so our deadline was at risk.
>
> First, I talked to him one-on-one and asked him to raise the priority. He agreed, but after a sprint, nothing had really changed. I saw that the problem was not willingness, it was his time.
>
> So, with my manager's approval, I handed my test preparation to a colleague and spent two days studying the code, to work out exactly what to change. Then I met him again, and he confirmed the plan. He still made the change himself, but with a clear plan he finished in three days. Development was done on time, and E2E started as planned. It helped both of us.
>
> What I learned is that asking is often not enough. If I make it easy for the other team to help, they can.

### Notes

- Do not say "I realized the risk in advance and quickly acted". The real order was: ask for priority, wait a sprint, then act. Say "when the priority talk wasn't enough, I took action myself."
- Mention the cost: test automation was two days late, but we tested manually first.
- Start from his situation (overloaded), not only from my deadline.
- Use this story once per interview round. It also answers priority conflict and helping a colleague.

---

## Q6. Which of Aspire, Excel and Unite resonates most with you?

### My answer

> I'd choose aspire, I am willing to try the new tools or tech in the real work. Last year when AI became powerful, I did not just use it for vibe coding, but also wanted it to handle more complicated tasks. Some of our data gateway tasks just processed the reports and webhooks. They were simple and repetitive, but still need to go through the steps like reading the jira ticket, learning the input and output event spec mapping for existing processors, doing the implementation as well as automation tests. So I learned to use skills + mcp server to fully handle such kind of tasks. After a few features, it became stable and I shared it to the teamates and they also started using it when they found it was really powerful.

### Revised answer

> Aspire resonates with me most, because I don't like waiting to be asked. I like to learn new tools and use them on real work.
>
> For example, last year, when AI tools became powerful, I didn't only use them for vibe coding. Many of our Data Gateway tasks are about reports and webhooks: simple and repetitive, but each one needs the same steps, such as reading the Jira ticket, learning the event mapping of existing processors, writing the code and the tests. So I taught myself to write skills and a local MCP server to handle these tasks. After a few features it became stable, and now I only review the result. I shared it on GitHub and in a team meeting, and my teammates all use it now.
>
> I did the same with EasyFin, a tool I built on my own initiative. For me, Aspire means not waiting to be asked, but pushing to do better.

### Notes

- The question has two parts: why it resonates, and an example. My answer only had the example.
- Check the real Aspire / Excel / Unite wording on their website before the interview, and use their words. `[TODO]`
- No numbers on time saved. Say "clearly less repetitive work, little rework".

---

## Q7. Do you have any questions for us?

### My answer

> What does a typical project look like in your team and what are the hardest technical challenges right now?

### Revised answer

Ask one at a time, then follow up on the answer.

> I've worked on payments and FX at eBay, so I'm curious: what are the hardest technical challenges your team faces right now?
>
> *(Follow-up, depending on the answer)* How does the team handle that today?

More to pick from (choose 2 or 3, by who is in the room):
- Engineers: How does the team handle reliability for payments, for example when a bank or scheme is unavailable?
- Manager: What would success look like in the first 6 months for this role?
- My background: I worked with Checkout.com as a merchant at eBay. How does the team use merchant feedback to shape the product?

### Notes

- Ask one question at a time. "Typical project" and "hardest challenge" are two questions.
- Listen, then ask a follow-up. It shows you are listening.
- Do not ask about salary or things on the website.

---

## Summary: which story for which question

| Question type | Story |
| --- | --- |
| Disagreement, pushing back with evidence | Currency expansion |
| Mistake | Shared function + RCA |
| Initiative, improved something | EasyFin |
| Tight deadline, cross-team blocker, helping a colleague | Payment Graph |
| Learning quickly, Aspire, influence without authority | AI skills + MCP |
| Competing priorities | Four steps + Buyer FX (two sentences) |
| Fix at the root | TasCreed retries |

Use each full story once per interview round.

## Still to do

- [ ] One real fact about Checkout.com for "Why" reason 3.
- [ ] Check the official wording of Aspire, Excel, Unite.
- [ ] Decide the number: 42 countries and regions, or 42 currencies. Use the same one everywhere.
- [ ] Add a later example for the shared-function habit, only if true.
