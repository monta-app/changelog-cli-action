# Monta `changelog cli action`

Github action for running Montas changelog CLI tool.

This is used as part of our release to generate a change log from a git tag.

It will:

* generate a github release, containing the change log
* post the change log to a slack channel

If you have done your commit messages using [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) the changelog will be formatted nicely in sections.

## Architecture Support

This action automatically detects the runner architecture and downloads the appropriate binary:
- **x86_64** - Downloads `changelog-cli-x64`
- **aarch64/arm64** - Downloads `changelog-cli-arm64`

The action works on both standard x64 runners and ARM64 runners (e.g., `ubuntu-24.04-arm`).

## Example of Github workflow job

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

See further documentation of options in [action.yml](./action.yml)
