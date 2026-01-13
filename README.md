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

This provides operations teams with deployment information directly in the changelog, making it easier to track what was deployed to which environment, when it was deployed, and enabling quick rollbacks and regression debugging if needed.

## PR and JIRA Commenting

The action can automatically comment on PRs and JIRA tickets when deploying to production:
- **Comment on PRs**: Posts deployment information on all PRs included in the release
- **Comment on JIRA**: Posts deployment information on all JIRA tickets referenced in commits

**Requirements:**
- Must set `stage: "production"`
- Must provide `deployment-start-time` and `deployment-end-time`
- Slack announcement must be posted (provides link in comments)
- For JIRA commenting: Must provide JIRA credentials (`jira-email`, `jira-token`, `jira-app-name`)

**Comment Format:**
Comments include:
- Release version
- Full changelog of what was included
- Deployment timing (human-readable format)
- Links to changeset, deployment system, and Slack announcement

This ensures stakeholders (PMs, designers, wider team) are notified when their work is deployed to production.

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
        # Deployment metadata (required for commenting)
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
