# UbatJelas

Medicine label & pharmacy jargon translator for elderly Malaysians. Built for
AWS CendekiAwan — GenAI Innovation Challenge.

**⚠️ This README replaces an earlier version that assumed a Docker + App
Runner deployment on a different timeline. That path is gone.** This app now
follows the competition's own required pipeline: **GitHub Actions → SAM →
Lambda → API Gateway**, with the frontend as a static site on S3 behind
CloudFront (HTTPS).

## Architecture

- `backend/lambda/app/` — the original FastAPI app (`main.py`, `models.py`,
  `i18n.py`, `bedrock_service.py`, `llamaparse_service.py`,
  `reminder_logic.py`). Routes and business logic are the same as the
  original design; the OCR layer was swapped from Textract to LlamaParse
  (Textract hit an account-level `SubscriptionRequiredException` that
  wasn't fixable on our timeline — see `llamaparse_service.py`'s docstring
  for the confidence-gate tradeoff that came with the swap).
- `backend/lambda/lambda_handler.py` — thin Mangum wrapper so API Gateway can
  invoke the FastAPI app as a Lambda function.
- `infra/template.yaml` — SAM template: one Lambda function behind an HTTP
  API, a private S3 bucket for the static frontend, and a CloudFront
  distribution that serves it over HTTPS (`*.cloudfront.net` default cert).
- `frontend/index.html` — a plain HTML/JS frontend replacing the Streamlit
  UI (Streamlit needs a running server; it can't be a static S3 file).
  Mirrors the original PartyRock widget flow: Photo Ubat → Extracted Label
  Text → Decoded Instructions/Penjelasan Mudah → Kad Ubat, plus a client-side
  reminder/alarm feature.
- `.github/workflows/deploy.yml` — builds, deploys, and uploads the frontend
  in one run.

## Things to double-check before you run the deploy

1. **Repository secrets — 4, not the guide's stock 3.** OCR runs through
   LlamaParse, not Textract, so no `AmazonTextractFullAccess` grant is
   needed on the `github-deployer` IAM user. But `deploy.yml` needs a 4th
   secret beyond the guide's usual three:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `SAM_DEPLOY_BUCKET`
   - `LLAMA_CLOUD_API_KEY` — from cloud.llamaindex.ai, passed to
     `template.yaml`'s `LlamaCloudApiKey` parameter at deploy time.
2. **Which AWS account/credentials you're actually using.** If you're on a
   personal AWS account with a permanent IAM user (`AKIA...` keys, per the
   guide's Step 2), the workflow as written works as-is with just
   `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` secrets. If you're on a
   **team sandbox account** with temporary credentials (`ASIA...` keys), you
   also need a session token — add an `AWS_SESSION_TOKEN` secret and an
   `aws-session-token:` line in the `configure-aws-credentials` step, and
   note that those credentials expire and will need refreshing.
3. **Confirm the exact Bedrock model ID** in your account (Bedrock console →
   Model catalog). `infra/template.yaml` defaults to
   `global.anthropic.claude-haiku-4-5-20251001-v1:0` — the cross-region
   inference profile ID our account needed (the bare
   `anthropic.claude-haiku-4-5-20251001-v1:0` 404s here). Some accounts
   need the bare ID instead. Override with
   `--parameter-overrides BedrockModelId=<your-id>` in the SAM Deploy step
   if the default 404s for you.
4. **File size limit.** `MAX_FILE_SIZE_MB` is set to 4 in the template, not
   10. Lambda has a 6MB synchronous payload limit, and API Gateway
   base64-encodes binary uploads (~33% overhead), so a 10MB photo would
   already be over budget. Don't raise this without also moving to a
   presigned-S3-upload flow.
5. **Notifications need the CloudFront URL.** The browser Notification API
   only works on HTTPS. Open the printed `https://xxxx.cloudfront.net` URL,
   not the old `http://...s3-website-...` endpoint (that hosting is gone;
   the bucket is private and only CloudFront can read it). The frontend
   still falls back to an in-page alert + audio beep if permission is
   denied.
6. **`github-deployer` IAM needs CloudFront.** If SAM deploy fails on
   `AWS::CloudFront::Distribution` / `OriginAccessControl`, or the
   invalidation step fails, add `cloudfront:*` (or at least create/update
   distribution, origin access control, and `CreateInvalidation`) to that
   user. The first CloudFront deploy can take 5–15 minutes while the
   distribution rolls out.

## Local test before deploying

```bash
cd backend/lambda
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt uvicorn
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=... AWS_REGION=ap-southeast-1
export BEDROCK_MODEL_ID=global.anthropic.claude-haiku-4-5-20251001-v1:0
uvicorn app.main:app --reload --port 8000
```

Quick smoke test without touching LlamaParse/Bedrock:
`curl -X POST http://localhost:8000/reminder -H "Content-Type: application/json" -d '{"frequency_code":"BD"}'`

For the frontend, replace `__API_URL_PLACEHOLDER__` in `frontend/index.html`
with `http://localhost:8000` temporarily and open the file directly in a
browser. **Revert it back to `__API_URL_PLACEHOLDER__` before you commit** —
`deploy.yml`'s `sed` step only replaces that exact placeholder string, so if
this edit is still in place when you push, the live CloudFront frontend will
keep pointing at your laptop's localhost instead of the real API Gateway URL,
and `sed` won't complain, it'll just silently do nothing.

## Deploying (follow the competition guide's Steps 5–8 as written)

Upload these files to GitHub exactly as the guide describes, add all 4
repository secrets (see checklist item 1 above — the guide's stock 3 plus
`LLAMA_CLOUD_API_KEY`), then run the workflow from the Actions tab. The last
step prints both URLs:

```
✅ API URL: https://xxxxx.execute-api.ap-southeast-1.amazonaws.com
✅ Frontend URL: https://xxxx.cloudfront.net
```

## Known MVP gaps (worth flagging to judges proactively)

- Reminders are held in browser memory only — closing the tab clears them.
  There's no backend persistence or push notification service.
- Tamil/Mandarin are offered in the language picker and passed to the
  translation prompt, but the UI chrome (buttons, warnings) only has full
  localized copy for English/Bahasa Malaysia — Tamil/Mandarin selections
  fall back to English UI text.
- PDF upload uses the first page only, rasterized in-Lambda — fine for a
  single-page pharmacy label.
- No automated tests were added in this pass; the guide's own error table
  (Step 7) is your main debugging reference if the first deploy fails.
