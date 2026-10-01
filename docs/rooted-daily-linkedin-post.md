# RootEd: daily LinkedIn post

Workflow file: `workflows/rooted-daily-linkedin-post.json` (already imported into n8n).

```
Every day at 9:00 ─┐
                   ├─> Build prompt ─> Claude: write post ─> Parse draft ─> Email me the draft ─> Approved? ─> Post to LinkedIn ─> Remember post
Test run ──────────┘                                                         (waits up to 12h)
```

1. **Build prompt** picks today's angle (17 rotating topics, such as fee collection, data security or the free core) and format (story, list, myth vs. reality…). It also lists the last 14 posts so Claude doesn't repeat itself. All product facts come from a brief taken from rooted.ruralrootcloud.com. Claude is told not to invent stats, customers or features.
2. **Claude: write post** calls `claude-opus-5-5`. Server-side refusal fallback is turned on: if the model declines a request, Anthropic automatically retries it on another model.
3. **Email me the draft** sends you the draft with a **Review & post** button. The form lets you edit the text and choose *Post it* or *Skip today*. If you don't answer within 12 hours, nothing is posted.
4. **Post to LinkedIn** publishes the post to your personal profile.

## One-time setup

### 1. n8n account
Open http://localhost:5678 and create the owner account.

### 2. Anthropic API key
1. https://console.anthropic.com → API Keys → **Create key**.
2. In n8n: **Credentials → Add credential → Anthropic**, then paste the key.
3. Open the **Claude: write post** node and choose that credential under *Credential for Anthropic*.

### 3. Email (to receive drafts): Gmail example
1. Turn on 2-Step Verification for your Google account, then create an **App password** at https://myaccount.google.com/apppasswords.
2. In n8n: **Add credential → SMTP**
   - User: your Gmail address · Password: the app password
   - Host: `smtp.gmail.com` · Port: `465` · SSL/TLS: on
3. Open **Email me the draft** and select the credential.

### 4. LinkedIn app
1. Go to https://www.linkedin.com/developers/apps → **Create app**. LinkedIn requires the app to be linked to a LinkedIn Page, so use the Rural Root Cloud page or create one.
2. **Products** tab: add **Share on LinkedIn** and **Sign In with LinkedIn using OpenID Connect**. Both are approved instantly.
3. **Auth** tab: add the redirect URL `http://localhost:5678/rest/oauth2-credential/callback`, then copy the Client ID and Client Secret.
4. In n8n: **Add credential → LinkedIn OAuth2 API**
   - Turn **off** *Organization Support* and turn **off** *Legacy*. Both default to on and will break sign-in.
   - Paste the Client ID and Secret → **Connect my account**.
5. Open **Post to LinkedIn**, select the credential, then pick yourself under **Person**.

### 5. Settings
Open the **Build prompt** node and edit `SETTINGS` at the top:
- `approverEmail`: where drafts go
- `fromEmail`: your SMTP address
- `author`: who the posts are written as (e.g. "the founder of RootEd")

The product brief, topic list and writing rules are in the same node. Edit them as RootEd changes.

### 6. Test, then turn it on
1. Click **Execute workflow** to run the *Test run* trigger. Within about a minute you should get the email.
2. Click **Review & post**. For the first test, choose **Skip today** so nothing is published. (*Post it* really posts.)
3. When you're happy, **Publish/activate** the workflow. From then on it runs every day at 9:00 IST.

## Good to know

- **Your computer must be on** with Docker running at 9:00. Missed days are skipped, not caught up.
- **The approval link only works on this computer**, because n8n is on `localhost`. To approve from your phone, n8n needs a public URL (e.g. Cloudflare Tunnel).
- **LinkedIn tokens expire after about 60 days.** When posting fails with an auth error, open the LinkedIn credential and click *Reconnect*.
- **Cost:** about one Claude request per day, which is a few US cents.
- **Post history** (used to avoid repeats) is saved only for scheduled runs, not test runs.
- After editing the workflow in the UI, export it to keep git in sync:
  `docker compose exec n8n n8n export:workflow --id=RootEdLinkedIn01 --output=/home/node/workflows/rooted-daily-linkedin-post.json --pretty`
