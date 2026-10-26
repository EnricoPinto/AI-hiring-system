# AI Hiring System

An end-to-end recruitment pipeline built in **n8n**. A candidate fills a form, an LLM reads and scores their resume, and the system sorts them into tiers, sends the right email, generates interview questions, handles scheduling, and reports on the pipeline, all without anyone touching a spreadsheet.

The whole thing lives in one workflow: 61 nodes, 8 entry points.

## What it does

**1. Application intake**
- Public n8n form collects name, contact, job ID, experience, salary expectations, notice period and a resume PDF.
- Candidate data is normalised into a single record with a generated `candidate_id`.
- Duplicate check on `email` + `job_id` against Google Sheets. Repeat applicants are handled separately instead of creating a second row.
- Acknowledgment email goes out via Gmail.

**2. Resume parsing**
- PDF text is extracted, then **Llama 3.3 70B (Groq)** turns it into structured JSON: current title/company, experience, education, skills, certifications, career gaps, achievements.
- Parsed fields are written back to the candidate's row.

**3. Screening and scoring**
- The matching job description is pulled from the `Job_Openings` sheet.
- An **anti-bias anonymizer** strips name, email, phone, links and other identifying fields before the AI sees the candidate. Only skills, experience, education, certifications, title and salary expectation go in.
- Two AI chains run on the anonymized profile:
  - **Skills evaluator** (Llama 3.3 70B) scores skills, experience, education, salary fit and a bonus.
  - **Red flag detector** (Llama 3.1 8B) flags things like a large experience gap, salary far over budget, or missing core skills.
- Scores are summed to a 0-100 total and bucketed into a tier:

| Tier | Score | What happens |
|------|-------|--------------|
| A | 75+ | Fast track: interview questions generated, scheduling email sent |
| B | 55-74 | Human review email |
| C | 35-54 | Hold pool email |
| D | below 35 | Auto-rejection email with a reason |

**4. Interviews**
- Tier A candidates get a tailored set of technical and behavioural questions (Llama 3.3 70B) and a Calendly booking link.
- The interview is logged in the `Interviews` sheet.
- An hourly check sends reminders to the interviewer about 24 hours and 1 hour before the slot.
- A Calendly webhook catches cancellations and sends the candidate a reschedule offer.
- Interviewers submit a scorecard through a separate feedback form, which updates the interview row.

**5. Offers and onboarding**
- A manual trigger generates a formal offer letter with the LLM, emails it, and logs it in `Offers`.
- A separate chain writes a personalised 30-60-90 day plan, emails it as a welcome message, and logs it in `Onboarding`.

**6. Analytics**
- Daily at 8:45 AM: tier counts and funnel numbers are logged to `Analytics_Daily`.
- Every Monday at 9 AM: the week's numbers are summarised by the LLM into an executive briefing and emailed to leadership.

## Stack

- **n8n** for orchestration (form triggers, webhook, schedules, code nodes)
- **Groq API**: `llama-3.3-70b-versatile` for parsing, scoring, questions, offers and summaries; `llama-3.1-8b-instant` for red flags
- **Google Sheets** as the database
- **Gmail** for all outbound email
- **Calendly** for interview booking and cancellation webhooks

## Repo layout

```
.
├── README.md
├── .gitignore
├── workflow/
│   └── ai-hiring-system-master-workflow.json   # import this into n8n
└── docs/
    └── sheet-schema.md                          # tabs and columns to create
```

The workflow JSON is sanitized: credential IDs, the Google Sheet ID and the n8n instance ID have been replaced with placeholders, and the workflow is exported as inactive.

## Setup

1. **Create the Google Sheet.** Make a spreadsheet with six tabs (`Candidates`, `Job_Openings`, `Interviews`, `Offers`, `Onboarding`, `Analytics_Daily`). Column headers are listed in [`docs/sheet-schema.md`](docs/sheet-schema.md). Add at least one row to `Job_Openings`.
2. **Import the workflow.** In n8n: Workflows > Import from File > pick `workflow/ai-hiring-system-master-workflow.json`.
3. **Add credentials** and attach them to the nodes that need them:
   - Google Sheets OAuth2
   - Gmail OAuth2
   - Groq API key (free tier works for testing)
4. **Point the sheet nodes at your spreadsheet.** Replace `YOUR_GOOGLE_SHEET_ID` in the Google Sheets nodes with your spreadsheet's ID (the long string in its URL).
5. **Edit the placeholders:**
   - `leadership@company.com` in *Send Executive Briefing Gmail*
   - `interviewer@company.com` and the Calendly link in *Format Interview Payload*
6. **Calendly webhook (optional).** Point a Calendly `invitee.canceled` webhook at the n8n production URL ending in `/webhook/calendly-cancellation`.
7. **Activate** the workflow and open the application form URL from the *n8n Form Trigger* node.

## Testing it

1. Add a job to `Job_Openings` with a `job_id` like `JOB_2026_001`.
2. Submit the application form with that job ID and a sample resume PDF.
3. Watch the execution: a new row should appear in `Candidates`, get parsed and scored, and an email should go to the address you entered. Use your own email while testing.

## Known limitations

- **Onboarding has no trigger node.** *Send Onboarding Welcome Email* references a node called `Trigger Onboarding Flow` that is not in the exported workflow. Add a manual trigger with that name, or the onboarding chain won't start.
- **Offer letters** default to the candidate's *expected* salary (or a placeholder) because there is no approved-offer field yet.
- **Daily metrics are simplified.** It counts every row in `Candidates` rather than only today's applications, and the offer/hire counters are hard-coded to 0.
- **LLM output is parsed as JSON.** If the model returns something malformed, scoring falls back to default values instead of failing, so spot-check scores early on.
- **Tier A auto-schedules; B, C and D only send emails.** There is no follow-up automation after a human reviews a Tier B candidate.
- The anonymizer removes personal identifiers, but free-text fields like the cover letter can still contain them.
