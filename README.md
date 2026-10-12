# Copilot telemetry

Local OpenTelemetry collection for GitHub Copilot CLI with Grafana's
`otel-lgtm` image. The container includes an OpenTelemetry Collector, telemetry
storage, and Grafana.

## Start

Docker with Compose and mise must be installed.

```sh
mise trust
docker compose up -d
```

Open <http://localhost:4567>. The default username and password are both `admin`.
The Grafana and collector ports accept local connections only.

## Collect Copilot sessions

Run Copilot from this directory:

```sh
mise exec -- copilot
```

If mise is activated in your shell, it loads the environment variables from
`mise.toml` when you enter this directory, so `copilot` also works directly.
The OTLP endpoint automatically enables Copilot telemetry. The collector
receives OTLP over HTTP on port `4318`.

In Grafana, use **Explore > Tempo** for traces from service `github-copilot`,
and **Explore > Prometheus** for metrics.

Copilot excludes prompts, responses, and tool arguments by default. Do not
enable content capture unless you trust the destination. Only sessions launched
with these environment variables export telemetry; past sessions are not imported.

## AI-credit metric

The collector converts model-call spans into the Prometheus counter
`github_copilot_ai_credits_total`. It divides `github.copilot.nano_aiu` by
1,000,000,000 and sums the result by session and model. Turn-level summary spans
are excluded to prevent double counting. Traces still go to Tempo, and existing
metrics still go to Prometheus.

After a collector configuration change, apply it with:

```sh
docker compose up -d
```

In Grafana, select **Explore > Prometheus** and query:

```promql
sum by (gen_ai_conversation_id) (github_copilot_ai_credits_total)
```

To select one session:

```promql
github_copilot_ai_credits_total{gen_ai_conversation_id="<session-id>"}
```

The `gen_ai_request_model` label identifies the model when available. Credits
appear as model-call spans arrive; the session does not need to end. Only spans
received after this configuration is enabled contribute to the counter. Existing
Tempo traces are not imported.

Counter state is held in collector memory. It resets when the collector restarts
or a series receives no new spans for 24 hours. Prometheus keeps the stored
history. For completed sessions, use a time range that covers their activity.

## Manage the container

```sh
docker compose ps
docker compose logs --tail=100
docker compose down
```

The named volume keeps telemetry data when the container is replaced or stopped
with `docker compose down`. Adding `--volumes` to `down` deletes that data.
The container restarts with Docker unless you explicitly stop it.

This stack is intended for local development, not production.
