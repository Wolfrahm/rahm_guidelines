# Rahm Guidelines

Contracts for the Rahm workflow — Wolfrahm's shared conventions for backend development and infrastructure.

Each guideline defines a single concern (logging, configuration, transport, etc.) with rules that all Rahm-built libraries and services follow. Apps describe what they do; the libraries handle transport, formatting, and enforcement.

## Guidelines

- [Logging](logging.md) — structured JSON logging to stdout, shipped to Loki via collector. Defines wire format, schema, log call signature, scope-based context, and exception handling.

More guidelines are in progress.

## License

[MIT](LICENSE) © 2026 David Van De Walle
