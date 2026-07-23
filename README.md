# CrushForYou

<p align="center">Crush for you. This repo's code is created and maintained by crushforyou, and the feature is decided by you.</p>

[Full Documentation](docs/README.md)

## Installation

```bash
go install github.com/sdgumdam/crushforyou/cmd/crushfy@latest
```

Or build from source:

```bash
git clone https://github.com/sdgumdam/crushforyou.git
cd crushforyou
go build -o crushfy .
```

## Quick Start

```bash
# Run interactively
crushfy

# Run a single prompt
crushfy run "explain this codebase"

# Continue a previous session
crushfy run --continue "what else did we find?"
```

## PRD Roadmap

- [ ] **Multi-model sub-agent dispatch**: per-agent pinned models + runtime model override
- [ ] **Three-layer context compaction**: MicroCompact → AutoCompact → preserve recent N messages
- [ ] **Reliable tool calls**: error isolation, cancel cleanup, duplicate detection
- [ ] **Fine-grained permissions**: per-command allowlist, per-session override
- [ ] **Background sub-agent execution**: async dispatch, session branching
- [ ] **Cost visibility**: real-time token speed, per-step cost breakdown
- [ ] **Provider friendliness**: config validation, auto-discover models, clear errors

## Contributing

We welcome contributions. Please read our [contributing guide](CONTRIBUTING.md) before opening a PR.

- Issues: [github.com/sdgumdam/crushforyou/issues](https://github.com/sdgumdam/crushforyou/issues)
- Discussions: [github.com/sdgumdam/crushforyou/discussions](https://github.com/sdgumdam/crushforyou/discussions)
