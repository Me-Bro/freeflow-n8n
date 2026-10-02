# Applicants sheet: prepare internship email

Workflow file: `workflows/lead-outreach-email.json` (already imported into n8n).

```
Start (form) ─> Read leads sheet ─> List leads ─> Pick a lead (form) ─> Build prompt ─> LLM: write email ─> Parse email ─> Create Gmail draft ─> Show draft (form)
```

You open a form in the browser and go through three pages:

1. **Start**: optionally type a filter (part of a name, college, course, program or email), then click *Load applicants*.
2. **Pick an applicant and the meeting**: choose the applicant, the meeting date, start time (15-minute slots, IST), duration and the Google Meet link, plus anything specific to mention.
3. **Done page**: shows the email and links to the draft in Gmail. Nothing is sent: you review and send it from Gmail.

The email follows the standard Rural Root Cloud format:

```
Subject: Regarding Your Application – Next Steps with Rural Root Cloud

Dear <first name>,

Greetings from Rural Root Cloud.

<3 paragraphs written by the LLM: we reviewed your request for <program> + please share your resume;
 a challenge someone with your background faces; why we'd like a short meeting>

Meeting Details
<first name> <> RR Cloud
Tuesday, March 17 · 10:00 – 10:30am
Time zone: Asia/Kolkata
Google Meet joining info
Video call link: <link>

Looking forward to your response.

Best Regards,
Team Rural Root Cloud
https://ruralrootcloud.com/internship/
```

Everything except the three paragraphs is fixed text. The paragraphs are written by the llm-gateway (`model: "json"`, free) from the applicant's college/company, course, year/experience and program. Phone number and address are not sent to the model.

## Sheet format

The Google Form responses sheet, with headers in row 1. The workflow uses *Name*, *Email*, *College / Previous Company*, *Course / Department*, *Semester / Passing Year / Experience* and *Program want to enroll*. Only an email column is required, and rows without one are skipped. If the form's questions are renamed, update `COLUMNS` at the top of the **List leads** node.

## One-time setup

### 1. LLM gateway credential
1. In n8n: **Credentials → Add credential → OpenAI**.
   - API Key: the `GATEWAY_API_KEY` value from `.env`
   - Base URL: `http://llm-gateway:8080/v1`
2. Open **LLM: write email** and choose that credential. (Already done on this machine: the credential is called *llm-gateway*.)

### 2. Google OAuth client (used by both Sheets and Gmail)
1. https://console.cloud.google.com → create a project (or reuse one).
2. **APIs & Services → Library**: enable **Google Sheets API**, **Google Drive API** (lets n8n list your sheets) and **Gmail API**.
3. **OAuth consent screen**: choose *External*, fill in the app name and your email, and add your Gmail address under **Test users**.
4. **Credentials → Create credentials → OAuth client ID** → *Web application*. Add the redirect URI `http://localhost:15678/rest/oauth2-credential/callback`, then copy the Client ID and Client Secret.

### 3. n8n Google credentials
1. **Add credential → Google Sheets OAuth2 API**: paste the Client ID and Secret, then **Sign in with Google**.
2. **Add credential → Gmail OAuth2 API**: same Client ID and Secret, then **Sign in with Google**.
3. Open **Read leads sheet**, select the Sheets credential, paste your sheet's URL under *Document* and pick the tab under *Sheet*.
4. Open **Create Gmail draft** and select the Gmail credential.

### 4. Settings
Open the **List leads** node and edit `SETTINGS` at the top:
- `subject`, `signature`: the fixed subject line and sign-off
- `defaultMeetLink`: pre-fills the Meet link field if you reuse one link
- `defaultStartTime`: the pre-selected time slot

The example email the model copies the style from, and its instructions, are in the **Build prompt** node.

### 5. Test, then turn it on
1. Click **Execute workflow**. n8n opens the test form. Pick a lead and check the draft that appears in Gmail.
2. **Publish/activate** the workflow, then open the **Start** node and copy its **Production URL**. Bookmark it: that's the form you use from now on.

## Good to know

- **Nothing is sent automatically.** The workflow only creates a Gmail draft.
- **Google tokens:** while the OAuth app is in *Testing* mode, Google expires the sign-in after 7 days. When a Google node fails with an auth error, open the credential and sign in again. Publishing the app (*OAuth consent screen → Publish app*) avoids this; for personal use you can skip Google's verification and click through the "unverified app" warning.
- **Up to 300 applicants** fit in the dropdown. Use the filter on the first page for bigger sheets.
- **Free models vary.** If the paragraphs read badly, delete the draft and run the form again; you'll often get a different provider. To prefer a provider, change its order in `llm-gateway/providers.json`.
- The Meet link is not created for you: create the meeting (or reuse a standing link) in Google Calendar and paste it in.
- The form only works on this computer, because n8n is on `localhost`.
- After editing the workflow in the UI, export it to keep git in sync:
  `docker compose exec n8n n8n export:workflow --id=SheetLeadEmail01 --output=/home/node/workflows/lead-outreach-email.json --pretty`
