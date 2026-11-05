### Changed

- `CLAUDE.md`: the product owner reviews and merges every pull request into `main`, `dev` and `release/X.Y.Z`; Claude manages the rest of Git (branches, tags, pre-releases) and deletes `release/X.Y.Z` once its version is published; every decision made during the work stays with the product owner, and the file says where each kind of decision is recorded.

### Added

- `CLAUDE.md`: chains of pull requests, merged in order only, with drafts, [SEC] carried along the chain and no conflict resolved alone; rulesets for `main`, `dev`, `release/**` and `v*` tags; what a pull request proves while no CI exists; a CI job and secret scanning behind `.gitignore`.
