### Fixed

- `.gitignore`: the `data/` and `models/` rules are anchored to the repository root, so they no longer ignore C# source folders named `Data/` or `Models/`.

### Added

- `.gitignore`: data, model and experiment files are ignored wherever they are, except the synthetic test fixtures in `test/**/fixtures/`; key files, `.env` and real `*.conf` files are ignored, and only `*.conf.example` files without secrets are versioned.
- `.gitattributes`: systemd timers (`*.timer`) and configuration files (`*.conf`, `*.conf.example`) keep LF line endings, like the other files read on Linux.
- `CLAUDE.md`: where data, models, test fixtures and configuration examples live.
