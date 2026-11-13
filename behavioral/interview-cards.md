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
