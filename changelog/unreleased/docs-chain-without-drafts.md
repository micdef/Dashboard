### Changed

- `CLAUDE.md`: every pull request of a chain is ready for review when opened and targets the branch of the previous one, so that the product owner can merge the chain in order without Claude and an out-of-order merge never reaches the release branch; the place in the chain is in the title, and the next branch is updated with "Update with merge commit".
- `CLAUDE.md`: the repository settings and rulesets now in place are described (merge commits only, automatic deletion of merged branches, rulesets for main and dev, release branches and version tags); required checks are added with the first CI; Claude Code's project permissions deny `gh pr merge`.
