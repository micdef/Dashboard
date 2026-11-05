### Fixed

- `.gitignore`: the `data/` and `models/` rules are anchored to the repository root, so they no longer ignore C# source folders named `Data/` or `Models/`.

### Changed

- `.gitattributes`: systemd timers (`*.timer`) and configuration files (`*.conf`) keep LF line endings, like the other files read on Linux.
