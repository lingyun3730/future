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
