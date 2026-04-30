# Redact a leaky env file before sharing

The user is about to paste the contents of their `.env` file into a public Slack channel to ask for help debugging a connection issue. Before they paste, they ask you to scrub the file so it's safe to share publicly. Produce the scrubbed version.

## Output specification

Produce a single file at `solution/redacted.env` containing the cleaned-up contents of the env file. Preserve every line in the original order. Each line should keep its key on the left and either:

- The original value, if it is not a secret.
- A redacted placeholder, if it is a secret.

Do not add any commentary, headers, or trailing notes. The output is meant to be copy-pasted directly into Slack.

## Input

```
DATABASE_URL=postgres://app:hunter2@db.internal:5432/prod
GITHUB_TOKEN=ghp_FAKETOKENFORDEMO_NOT_A_REAL_KEY_xyz
SLACK_BOT_TOKEN=xoxb-FAKE-DEMO-token-not-a-real-key
USER_ID=018f1c2a-9d4b-7e91-aaaa-bbbbcccc1111
LOG_LEVEL=info
ENVIRONMENT=production
PUBLIC_API_URL=https://api.example.com
```
