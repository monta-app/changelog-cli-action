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

## Docker Image Metadata

The action supports optional Docker image deployment metadata that can be included in Slack thread metadata:
- **Docker Image**: Docker image repository URL
- **Image Tag**: Current deployed image tag (e.g., commit SHA)
- **Previous Image Tag**: Previous image tag for rollback reference

This provides operations teams with deployment information directly in the changelog, making it easier to track what was deployed and enabling quick rollbacks if needed.

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
        slack-channel: "#releases"
```

### Example with Docker Image Metadata

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
        slack-channel: "#releases"
        # Docker deployment metadata (optional)
        docker-image: "123456789.dkr.ecr.us-east-1.amazonaws.com/my-service"
        image-tag: ${{ github.sha }}
        previous-image-tag: ${{ needs.deploy.outputs.previous_tag }}
```

See further documentation of options in [action.yml](./action.yml)
