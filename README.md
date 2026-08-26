# Monta `changelog cli action`

Github action for running Montas changelog CLI tool.

This is used as part of our release to generate a change log from a git tag.

It will:

* generate a github release, containing the change log
* post the change log to a slack channel

If you have done your commit messages using [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) the changelog will be formatted nicely in sections.

## Job Information in Slack

When posting to Slack, the action automatically includes metadata in a threaded message:
- **Job URL**: Link to the GitHub Actions workflow run that generated the changelog
- **Triggered By**: GitHub username of the person who triggered the workflow run (linked to their GitHub profile)

This helps track which workflow generated each changelog and provides better traceability.

## Deployment Metadata

The action supports optional deployment metadata that can be included in Slack thread metadata:
- **Stage/Environment**: Deployment stage (e.g., dev, staging, production)
- **Docker Image**: Docker image repository URL
- **Image Tag**: Current deployed image tag (e.g., commit SHA)
- **Previous Image Tag**: Previous image tag for rollback reference
- **Deployment Start Time**: Timestamp when deployment started (ISO 8601 format recommended)
- **Deployment End Time**: Timestamp when deployment finished (ISO 8601 format recommended)
- **Deployment URL**: Link to the deployment system (ArgoCD, Cloudflare, etc.)
- **Deployed Systems** (`deployments`): JSON array of the systems/components shipped in this release — for multi-service releases (see below)

This provides operations teams with deployment information directly in the changelog, making it easier to track what was deployed to which environment, when it was deployed, and enabling quick rollbacks and regression debugging if needed.

### Deployed Systems (`deployments`)

For a release that ships several systems, the `deployments` input takes a JSON array describing each one — typically piped straight from the deploy pipeline's rollout wait. Each entry:

| Field | Required | Format | Example |
|---|---|---|---|
| `name` | yes | display name | `hub` |
| `revision` | no | git commit SHA (full or short) | `80aad1c` |
| `start` | no | ISO 8601 UTC timestamp | `2026-08-26T08:19:18Z` |
| `end` | no | ISO 8601 UTC timestamp | `2026-08-26T08:30:59Z` |
| `status` | no | rollout health | `healthy` |
| `url` | no | link (e.g. ArgoCD app) | `https://argocd.monta.app/applications/argocd/frontend-hub-production` |

Unknown fields are ignored and malformed JSON is dropped — it never fails the changelog. Non-ISO `start`/`end` values are shown verbatim.

```yaml
        # e.g. the JSON emitted by the argocd-wait-sync-multi rollout wait
        deployments: ${{ needs.wait-rollout.outputs.deployments }}
```

## PR and JIRA Commenting

The action can automatically comment on PRs and JIRA tickets when deploying to production:
- **Comment on PRs**: Posts deployment information on all PRs included in the release
- **Comment on JIRA**: Posts deployment information on all JIRA tickets referenced in commits

**Requirements:**
- Must set `stage` to `production` or `internal`
- Slack announcement must be posted (`output: slack`) — its link is included in the comments
- For JIRA commenting: Must provide JIRA credentials (`jira-email`, `jira-token`, `jira-app-name`)

Deployment start/end times are optional; when present they add a human-readable deploy window to the comment.

**Comment Format:**
Comments include:
- Release version
- Full changelog of what was included
- Deployment timing (human-readable format)
- Links to changeset, deployment system, and Slack announcement

This ensures stakeholders (PMs, designers, wider team) are notified when their work is deployed to production.

## Release Notifications

Independently of `output`, the action can post a short Slack message announcing the release, listing
dashboards to keep an eye on, and tagging the people who authored, co-authored, or approved the pull
requests included in it. Setting `release-notify-channel` is what turns this on - it reuses `slack-token`
to authenticate, so no separate token input is needed.

- **`release-notify-channel`**: Slack channel ID or name to post the notification to
- **`monitoring-urls`**: Comma-separated list of dashboard/monitoring URLs. Each entry is a bare URL or
  `Label|https://url` to give it a display label

Contributors are tagged with a real Slack mention when their public GitHub/commit email matches a Slack
account; otherwise they're linked to their GitHub profile instead. Anyone who only approved (or only
co-authored via a `Co-authored-by:` trailer) is suffixed with `(approver)` / `(co-author)`. Requires
`github-token` to resolve PR authors, approvers, and co-authors.

```yaml
    - name: Run changelog cli action
      uses: monta-app/changelog-cli-action@main
      with:
        service-name: "My Service"
        github-release: true
        github-token: ${{ secrets.GITHUB_TOKEN }}
        output: "slack"
        slack-token: ${{ secrets.SLACK_TOKEN }}
        slack-channel: "#info-releases"
        # Release notification (optional)
        release-notify-channel: "#my-service-releases"
        monitoring-urls: "Grafana|https://grafana.monta.app/d/my-service,Sentry|https://sentry.io/my-service"
```

## Architecture Support

This action automatically detects the runner architecture and downloads the appropriate binary:
- **x86_64** - Downloads `changelog-cli-x64`
- **aarch64/arm64** - Downloads `changelog-cli-arm64`

The action works on both standard x64 runners and ARM64 runners (e.g., `ubuntu-24.04-arm`).

## Example of Github workflow job

### Basic Example

```yaml
create-change-log:
  needs: deploy
  name: Create and publish change log
  runs-on: ubuntu-latest
  timeout-minutes: 5
  steps:
    - name: Run changelog cli action
      uses: monta-app/changelog-cli-action@main
      with:
        # the name of your service
        service-name: "<name of your service>"
        # true if a github release should be created - recommended
        github-release: true
        # the github token - use a reference to a secret
        github-token: <token>
        # name of the Jira app used for generating Jira issue links
        jira-app-name: <jira-app>
        # output mode - always 'slack'
        output: "slack"
        # the slack token - use a reference to a secret
        slack-token: <token>
        # the slack channel to post to
        slack-channel: "#info-releases"
```

### Example with Deployment Metadata

```yaml
create-change-log:
  needs: deploy
  name: Create and publish change log
  runs-on: ubuntu-latest
  timeout-minutes: 5
  steps:
    - name: Run changelog cli action
      uses: monta-app/changelog-cli-action@main
      with:
        service-name: "My Service"
        github-release: true
        github-token: ${{ secrets.GITHUB_TOKEN }}
        jira-app-name: "myapp"
        output: "slack"
        slack-token: ${{ secrets.SLACK_TOKEN }}
        slack-channel: "#info-releases"
        # Deployment metadata (optional)
        stage: "production"
        docker-image: "123456789.dkr.ecr.us-east-1.amazonaws.com/my-service"
        image-tag: ${{ github.sha }}
        previous-image-tag: ${{ needs.deploy.outputs.previous_tag }}
        deployment-start-time: ${{ needs.deploy.outputs.start_time }}
        deployment-end-time: ${{ needs.deploy.outputs.end_time }}
        deployment-url: "https://argocd.monta.app/applications/argocd/my-service-production"
```

### Example with PR and JIRA Commenting

```yaml
create-change-log:
  needs: deploy
  name: Create and publish change log
  runs-on: ubuntu-latest
  timeout-minutes: 5
  steps:
    - name: Run changelog cli action
      uses: monta-app/changelog-cli-action@main
      with:
        service-name: "My Service"
        github-release: true
        github-token: ${{ secrets.MONTA_BOT_TOKEN }}  # Use bot token for PR comments
        jira-app-name: "myapp"
        jira-email: ${{ secrets.JIRA_EMAIL }}
        jira-token: ${{ secrets.JIRA_TOKEN }}
        output: "slack"
        slack-token: ${{ secrets.SLACK_TOKEN }}
        slack-channel: "#info-releases"
        # Deployment metadata (stage is required for commenting; times are optional)
        stage: "production"
        docker-image: "123456789.dkr.ecr.us-east-1.amazonaws.com/my-service"
        image-tag: ${{ github.sha }}
        previous-image-tag: ${{ needs.deploy.outputs.previous_tag }}
        deployment-start-time: ${{ needs.deploy.outputs.start_time }}
        deployment-end-time: ${{ needs.deploy.outputs.end_time }}
        deployment-url: "https://argocd.monta.app/applications/argocd/my-service-production"
        # Enable PR and JIRA commenting (optional)
        comment-on-prs: true
        comment-on-jira: true
```

**Note:** For PR commenting to work properly, use a bot token (e.g., `MONTA_BOT_TOKEN`) instead of `GITHUB_TOKEN` for the `github-token` input. This allows the action to comment on PRs as the bot user.

See further documentation of options in [action.yml](./action.yml)
