# Woodpecker CI Slack Plugin

Woodpecker plugin for sending Slack notifications cloned from: https://github.com/drone-plugins/drone-slack

There is no published Docker image for this plugin — build your own and push it to a
registry your Woodpecker agents can pull from (see [Docker](#docker)), then reference
that image in your pipeline.

The webhook URLs are provided once, globally, via the server's `WOODPECKER_ENVIRONMENT`
(see [Channel routing](#channel-routing)), so pipelines don't reference any secret:

```yaml
  slack-failure:
    image: <your-registry>/woodpecker-slack:latest
    pull: true
    settings:
      status: failure
      description: <description of your service or system>
    when:
      status: failure

  slack-release:
    image: <your-registry>/woodpecker-slack:latest
    pull: true
    settings:
      status: success
      description: <description of your service or system>
    when:
      event: push
      branch: <main branch, usually master>
      status: success
```

This ensures that a message is posted if a pull request or master commit fails and also that a message is posted if master succeeds and a new version is deployed.

## Channel routing

Instead of wiring a separate step per channel, the plugin decides where a message
goes from the build outcome and event:

| Situation                          | Webhook used            |
| ---------------------------------- | ----------------------- |
| Failure on a pull request          | `SLACK_WEBHOOK_NOTICE`  |
| Failure on the default branch      | `SLACK_WEBHOOK_ALERTS`  |
| Success (any event)                | `SLACK_WEBHOOK_NOTICE`  |

Provide the two URLs once, globally, on the Woodpecker server so no pipeline has to
reference them:

```ini
WOODPECKER_ENVIRONMENT=SLACK_WEBHOOK_NOTICE:https://hooks.slack.com/services/AAA,SLACK_WEBHOOK_ALERTS:https://hooks.slack.com/services/BBB
```

These land in every step's environment and the plugin reads them directly. Note that
`WOODPECKER_ENVIRONMENT` values are **not** redacted or image-filtered — they are visible
to every step of every pipeline (including untrusted PRs and third-party plugins), so only
use this for a trusted, internal forge. If you need the URLs protected, pass them per-step
instead as `webhook_notice` / `webhook_alerts` settings (env `PLUGIN_WEBHOOK_NOTICE` /
`PLUGIN_WEBHOOK_ALERTS`) from `from_secret`. If only one is set, the other falls back to it,
and the legacy single `webhook` setting still works (routing disabled — everything goes to it).

The success/failure split still has to come from each pipeline's `when: status:`
conditions — Woodpecker 3.x removed `CI_PIPELINE_STATUS`, so the plugin cannot detect
failure on its own. Only the channel choice lives in the plugin.

## Dynamic messages from a file (`message_file`)

Plugin `settings:` are static YAML, so a message whose content is computed at
runtime (e.g. "which versions did this deploy actually include?") can't be
passed via `message:`. Instead, have an earlier step write the message to a
file in the workspace and point the plugin at it:

```yaml
  announce-deploy:
    image: gowerstreet/slack:latest
    settings:
      message_file: deploy-announcement.txt
```

The file content is sent verbatim (Slack mrkdwn works, e.g. `<url|text>`
links). If the file is **missing or blank, the step succeeds silently and
sends nothing** — so a pipeline run that had nothing to announce (e.g. a no-op
deploy) stays quiet without any conditional logic in the pipeline. Channel
routing applies as above (successes go to `SLACK_WEBHOOK_NOTICE`).

## Build

Build the binary with the following commands:

```
mise build
```

This will produce a file called `drone-slack` in your root directory.

## Docker

There is no image published to a public registry, so you need to build and host your own.
Build the Docker image and test-run it:

```
mise docker:run
```

This will produce docker image `gowerstreet/slack:local` and run it with a few test variables. You should be able to see the output in the `#alerts` slack channel.

To use the plugin in your pipelines, tag the image for a registry your Woodpecker agents
can pull from and push it there:

```
docker tag gowerstreet/slack:local <your-registry>/woodpecker-slack:latest
docker push <your-registry>/woodpecker-slack:latest
```

Alternatively, build without mise:

```
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -tags netgo -ldflags '-w' .
docker build --rm -t <your-registry>/woodpecker-slack:latest .
docker push <your-registry>/woodpecker-slack:latest
```
