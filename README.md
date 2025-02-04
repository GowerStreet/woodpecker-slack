# Woodpecker CI Slack Plugin

Woodpecker plugin for sending Slack notifications cloned from: https://github.com/drone-plugins/drone-slack

Use in `.woodpecker.yaml` as follows:

```yaml
  slack-failure:
    image: gowerstreet/slack:latest
    settings:
      status: failure
      description: "Slack plugin for Woodpecker"
      webhook:
        from_secret: slack_webhook
    when:
      status: failure

  slack-release:
    image: gowerstreet/slack:latest
    settings:
      status: success
      description: "Slack plugin for Woodpecker"
      webhook:
        from_secret: slack_webhook
    when:
      event: push
      branch: <main branch, usually master>
      status: success
```

This ensures that a message is posted if a pull request or master commit fails and also that a message is posted if master succeeds and a new version is deployed.

## Build

Build the binary with the following commands:

```
mise build
```

This will produce a file called `drone-slack` in your root directory.

## Docker

Build the Docker image and test-run it:

```
mise docker:run
```

This will produce docker image `gowerstreet/slack:local` and run it with a few test variables. You should be able to see the output in the `#alerts` slack channel.
