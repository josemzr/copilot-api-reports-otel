# Informes de API y demostración OpenTelemetry de GitHub Copilot

Este repositorio contiene una demostración práctica de dos flujos
complementarios de observabilidad de GitHub Copilot:

1. Exportar informes empresariales de consumo de AI credits mediante la API
   REST de GitHub.
2. Recibir telemetría de agentes de GitHub Copilot mediante un Collector de
   OpenTelemetry e identificar custom agents con `copilot_chat.mode_name`.

El repositorio incluye guías paso a paso separadas para Linux/macOS y Windows.
Los ejemplos utilizan una enterprise de pruebas y custom agents sintéticos.

## Contenido

- [`demo-guide.md`](./demo-guide.md): guía para Linux y macOS.
- [`demo-guide-windows.md`](./demo-guide-windows.md): guía para Windows.
- [`compose.yaml`](./compose.yaml): configuración de Docker Compose para el
  Collector de OpenTelemetry.
- [`collector.yaml`](./collector.yaml): receptor OTLP, filtrado, transformación
  de privacidad y exportadores a ficheros.
- [`workspace/.github/agents/`](./workspace/.github/agents/): custom agents
  sintéticos utilizados en la demostración de telemetría.

## Requisitos

- Una cuenta de GitHub Enterprise autorizada para solicitar informes de
  facturación.
- Un token expuesto como `GITHUB_BILLING_TOKEN` con el permiso
  `manage_billing:enterprise`.
- Acceso a GitHub Copilot y a un modelo habilitado para la demostración de
  telemetría.
- Visual Studio Code con GitHub Copilot Chat.
- Docker Desktop o un Docker Engine compatible con Compose.
- `curl` y `jq` para los ejemplos de facturación. En Windows, estos comandos
  se ejecutan desde Git Bash; la demostración OpenTelemetry utiliza PowerShell.

## Inicio rápido

Seleccionar la guía correspondiente al sistema operativo:

- **Linux y macOS:** [guía completa](./demo-guide.md).
- **Windows:** [guía completa](./demo-guide-windows.md).

Ambas guías incluyen los requisitos, la exportación de informes de facturación,
el arranque del Collector y la configuración manual de la telemetría de
GitHub Copilot en Visual Studio Code.

## Tratamiento de datos

El Collector escribe:

- `output/received.jsonl`: datos OTLP originales recibidos durante la
  demostración.
- `output/sanitized.jsonl`: spans de invocación filtrados que contienen
  únicamente los atributos necesarios.

La salida original está habilitada únicamente para demostrar la transformación
antes/después. Se deben revisar la retención, los controles de acceso, la
seguridad del transporte y los requisitos de privacidad antes de adaptar esta
configuración para producción.
