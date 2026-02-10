---
title: Cloudflare Pages Webhook Builds
description: Trigger Cloudflare Pages rebuilds on a schedule using GitHub Actions and deploy hooks.
tags:
- github actions
- cloudflare pages
---

Cloudflare Pages rebuilds your site on every commit, but scheduled content won’t trigger a build automatically. Deploy hooks let you trigger a build via a webhook. his guide shows how to set up a deploy hook and use GitHub Actions to run it on a schedule, keeping your site always up to date.

??? info "Prerequisites"
    - A **Cloudflare Pages** project linked to a Git repository.
    - A **GitHub** repository for the same project.

## Create a Deploy Hook

Cloudflare Pages provides deploy hooks, unique URLs that trigger a new build when they receive an HTTP POST request.

1. Log in to the [Cloudflare Dashboard].
2. Navigate to **Compute & AI** -> **Workers & Pages**.
3. Open the project for which you want to set up the webhook.
4. Go to the **Settings** tab and click the **+** icon next to **Deploy Hooks**.
5. Provide a name for your webhook.
6. Click **Save**.
7. Copy the URL under **Test by sending a POST request**.

!!! warning "Keep your webhook URL secure"
    The deploy hook URL can trigger builds without authentication. Never commit it directly to your repository.

## GitHub Secrets

To keep the webhook URL secure, store it as an encrypted GitHub Secret:

1. Go to your repository in GitHub.
2. Navigate to **Settings** -> **Secrets and variables** -> **Actions**.
3. Click **New repository secret**.
4. Set the **Name** to `CLOUDFLARE_WEBHOOK`.
5. Set the **Value** to your deploy hook URL wrapped in quotes: `"https://api.cloudflare.com/client/v4/pages/webhooks/deploy_hooks/..."`.
6. Click **Add secret**.

## Create the GitHub Action

Create a workflow file at `.github/workflows/pages-deployment.yaml`:

```yaml title=".github/workflows/pages-deployment.yaml" hl_lines="5"
name: Scheduled Cloudflare Pages Build
on:
  workflow_dispatch:
  schedule:
    - cron: "0 9 * * *" # (1)!

jobs:
  webhook:
    name: Trigger Deploy Hook
    runs-on: ubuntu-latest
    steps:
      - name: Send POST request to Cloudflare
        run: |
          curl -X POST {% raw %}${{ secrets.CLOUDFLARE_WEBHOOK }}{% endraw %}
```

1. :material-clock-outline: **Schedule** - Runs daily at 09:00 UTC. Adjust the cron expression to match your publishing schedule.

This workflow:

- Runs **daily at 09:00 UTC** via the `schedule` trigger
- Can also be triggered **manually** via `workflow_dispatch` in the GitHub Actions UI
- Sends a POST request to the Cloudflare deploy hook, which starts a new build

??? tip "Adjust the schedule"
    The `cron` expression `0 9 * * *` means every day at 09:00 UTC. Adjust it to match your publishing schedule. See [Crontab Guru] for help with cron syntax.

[cloudflare dashboard]: https://dash.cloudflare.com/
[Crontab Guru]: https://crontab.guru/
