# GitHub Copilot API Reports and OpenTelemetry Demo

This repository contains a practical demonstration of two complementary
GitHub Copilot observability workflows:

1. Exporting enterprise AI credit usage reports through the GitHub REST API.
2. Receiving GitHub Copilot agent telemetry through an OpenTelemetry
   Collector and identifying custom agents with `copilot_chat.mode_name`.

The repository includes separate step-by-step guides for Linux/macOS and
Windows. The examples use a test enterprise and synthetic custom agents.

## Contents

- [`demo-guide.md`](./demo-guide.md): Linux and macOS guide.
- [`demo-guide-windows.md`](./demo-guide-windows.md): Windows guide.
- [`compose.yaml`](./compose.yaml): Docker Compose configuration for the
  OpenTelemetry Collector.
- [`collector.yaml`](./collector.yaml): OTLP receiver, filtering, privacy
  transformation, and file exporters.
- [`workspace/.github/agents/`](./workspace/.github/agents/): synthetic
  custom agents used by the telemetry demo.

## Requirements

- A GitHub enterprise account authorized to request billing reports.
- A token exposed as `GITHUB_BILLING_TOKEN` with the
  `manage_billing:enterprise` scope.
- GitHub Copilot access and an enabled model for the telemetry walkthrough.
- Visual Studio Code with GitHub Copilot Chat.
- Docker Desktop or a compatible Docker Engine with Compose.
- `curl` and `jq` for the billing examples. On Windows, these commands are
  run from Git Bash; the OpenTelemetry walkthrough uses PowerShell.

## Quick start

Create the local output directory and start the Collector:

```bash
mkdir -p output
docker compose up -d
docker compose port collector 4318
```

Compose publishes the OTLP HTTP port on an available loopback port. Use the
reported port when configuring the Copilot Chat OpenTelemetry endpoint in
Visual Studio Code.

Follow the appropriate guide for the complete billing and telemetry
walkthrough.

## Data handling

The Collector writes:

- `output/received.jsonl`: raw OTLP data received by the demo.
- `output/sanitized.jsonl`: filtered invocation spans containing only the
  attributes required by the walkthrough.

Raw output is enabled only to demonstrate the before/after transformation.
Review retention, access controls, transport security, and privacy
requirements before adapting this configuration for production.
