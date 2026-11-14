# AI-Assisted Recruitment Screening Bot (n8n)

An n8n workflow that automatically screens job applications submitted through Google Forms. It de-duplicates applicants, reads each resume, gets an AI-generated evaluation, logs everything to a master Google Sheet, and emails the hiring team. **A human always makes the final decision.**

## Features

- Runs on a schedule (every 30 minutes) and picks up new applications from two Google Form response sheets: **Software Engineer (SWE)** and **Business Development Manager (BDM)**
- Detects duplicate applicants by email, both against the master sheet and within the same batch, and marks them `Duplicate`
- Downloads each applicant's PDF resume from Google Drive and extracts its text
- Sends the CV text and application details to an LLM (Groq, `openai/gpt-oss-120b`) for a job-related evaluation: score (1-10), classification, summary, strengths, concerns and interview focus
- Appends the result to a `Hiring Master` sheet
- Updates the original response row to `Processed - Pending Human Review`
- Sends an email notification with the candidate's details and the AI evaluation

## Workflow overview

```
Schedule Trigger
 ├─ SWE responses ─────┐
 ├─ BDM responses ─────┴─ Merge ─ Filter (Status empty/"New") ─┐
 └─ Hiring Master ─────────────────────────────────────────── merge
                                                                 │
                                              Duplicate check (by email)
                                               ├─ duplicate ─ Mark row "Duplicate"
                                               └─ new ─ Drive download ─ PDF extract
                                                        ─ Build prompt ─ Groq ─ Parse
                                                        ─ Normalize ─ Write to Hiring Master
                                                        ─ Update response row ─ Email
```

## Requirements

- An [n8n](https://n8n.io) instance (cloud or self-hosted)
- Google account with:
  - Two Google Forms with linked response sheets (SWE and BDM)
  - A `Hiring Master` Google Sheet
  - Resumes uploaded as **PDF** files through the forms (stored in Google Drive)
- A [Groq](https://console.groq.com) API key
- An SMTP account for sending notification emails

## Sheet setup

**SWE (Responses)** columns:
`Timestamp, Full Name, Email, Phone number, Resume, LinkedIn Profile URL, Years of Experience, Primary Programming Language, Github Profile URL, Status`

**BDM (Responses)** columns:
`Timestamp, Name, Email, Phone Number, Resume/CV, LinkedIn Profile URL, Experience, Industry Experience, Largest Deal Closed in USD, Status`

**Hiring Master** columns (header row only to start):
`Candidate ID, Name, Email, Phone, Role, Resume URL, LinkedIn URL, Experience Years, Extra Field 1, Extra field 2, AI Score, AI Classification, AI Summary, AI Strengths, AI Concerns, Interview Rec, Status, Processed At`

The `Status` column must be added manually to both response sheets. Leave it empty for new applications.

## Setup

1. In n8n, go to **Workflows → Import from file** and select the workflow JSON.
2. Set up and select credentials on these nodes:
   - Google Sheets (all sheet nodes)
   - Google Drive (`Download file`)
   - SMTP (`Send Hiring Notification`)
3. In each Google Sheets node, re-select your document and sheet tab from the dropdown.
4. In the `HTTP Request` node, add your Groq API key. Use an n8n credential rather than pasting the key into the node.
5. In `Send Hiring Notification`, set your sender and recipient addresses.
6. Click **Execute workflow** to test, then activate the workflow to enable the schedule.

## Testing

Sample data is provided as CSV files (`SWE (Responses).csv`, `BDM (Responses).csv`, `Hiring Master.csv`).

1. Upload a sample PDF to Google Drive and put its link in the Resume column of each test row.
2. Leave `Status` empty on the test rows.
3. Run the workflow. You should see:
   - New applicants processed and added to `Hiring Master`
   - An email per processed candidate
   - Repeated emails marked `Duplicate`

## How duplicates are handled

An application is a duplicate if its email (case-insensitive) already appears in `Hiring Master` or earlier in the same run. Response rows are updated by sheet row number, so a duplicate submission never changes the status of the original row.

## Responsible use

- The AI output is a **preliminary aid only**. The prompt tells the model not to infer protected characteristics and not to make hiring or rejection decisions.
- Every application is left as `Pending Human Review`. A person must review each one.
- Applicant data is personal data. Restrict access to the sheets and handle it according to your local privacy rules.

## Limitations

- Only **PDF** resumes are supported. Other formats will fail at the extraction step.
- Resumes must be accessible to the Google account connected in n8n.
- AI scores are not calibrated and can be inconsistent. Treat them as a starting point.

## Security

Never commit API keys, credentials or applicant data to the repository. If you export the workflow JSON, remove any hardcoded keys first and rotate any key that was exposed.
