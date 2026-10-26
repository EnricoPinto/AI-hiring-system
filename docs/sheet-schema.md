# Google Sheet schema

Create one spreadsheet with these six tabs. Row 1 of each tab must contain the column headers exactly as written (the workflow maps by header name). Matching is case-sensitive.

## Candidates

```
candidate_id, job_id, position, full_name, email, phone, linkedin_url, portfolio_url,
source, referral_by, application_date, status, stage, overall_score, tier,
skills_score, experience_score, education_score, salary_score, culture_fit_score,
red_flags, rejection_reason, current_title, current_company, years_experience,
technical_skills, education, certifications, current_salary, expected_salary,
notice_period, available_date, resume_file_url, cover_letter,
interview_1_date, interview_1_score, interview_1_feedback,
interview_2_date, interview_2_score, interview_2_feedback,
offer_amount, offer_sent_date, offer_status, hire_date, rejection_date,
notes, last_updated
```

## Job_Openings

The screening prompt reads these fields:

```
job_id, title, required_skills, preferred_skills, min_experience, must_have,
salary_min, salary_max
```

Add any extra columns you like (department, location, status). Example `job_id`: `JOB_2026_001`.

## Interviews

```
interview_id, candidate_id, job_id, interview_type, scheduled_date, scheduled_time,
duration_minutes, interviewer_name, interviewer_email, format, video_link, status,
overall_recommendation, technical_score, communication_score, problem_solving_score,
culture_fit_score, strengths, concerns, specific_examples, salary_recommendation,
next_step, feedback_submitted_at
```

## Offers, Onboarding, Analytics_Daily

These tabs use auto-mapped appends, so the headers need to match the fields the workflow writes. Easiest way: run the workflow once with the sheet empty, open the failed or successful node output, and copy its field names into row 1. For `Analytics_Daily` the fields are:

```
date, new_applications, screened, tier_a, tier_b, tier_c, tier_d,
interviews_scheduled, interviews_completed, offers_sent, offers_accepted,
hires_made, by_source_json, by_position_json
```
