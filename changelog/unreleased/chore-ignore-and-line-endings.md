### Fixed

- `.gitignore`: the `data/` and `models/` rules are anchored to the repository root, so they no longer ignore C# source folders named `Data/` or `Models/`.

### Added

- `.gitignore`: data, model, log and database files are ignored wherever they are, except the synthetic test fixtures in `test/**/fixtures/` and `tests/**/fixtures/`; experiment outputs are ignored at the root; key files, certificates, `.env` files, local secrets (`/secrets/`, `*.secret`) and real `*.conf` files are ignored, and only `*.conf.example` files without secrets are versioned.
- `.gitattributes`: systemd timers (`*.timer`) and configuration files (`*.conf`, `*.conf.example`) keep LF line endings, like the other files read on Linux.
- `CLAUDE.md`: where data, models, test fixtures, local secrets and configuration examples live; tests generate their keys and certificates instead of versioning them.
