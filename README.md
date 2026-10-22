# Praxis Enhancement Proposals
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fpraxis-proxy%2Fenhancements.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Fpraxis-proxy%2Fenhancements?ref=badge_shield)


Enhancement proposals for [Praxis] repositories.

[Praxis]: https://github.com/praxis-proxy

## How New Features Happen

Most of the time features show up on the individual repositories
to which they belong:

```
Issue -> Triage -> Accepted -> Work it
```

1. **Open an issue.** Describe what you want and
   why. It starts life labeled
   `triage/needs-triage`.

2. **Maintainers triage it.** They mark it either
   `triage/accepted` or `triage/declined`. If a
   change is large enough to need a written
   proposal first, maintainers will say so
   clearly (`triage/needs-proposal`).

3. **Build it.** Once accepted, it's fair game to
   work on according to its project status and
   milestone.

For much larger efforts you may be asked to follow the
[docs/process.md](docs/process.md) which is more involved.

> **Nothing here is guaranteed.** Acceptance does
> not guarantee a feature ships, and a feature can
> be changed, reworked, or removed at any stage.

## Structure

```
docs/
  process.md              # Full proposal lifecycle
  statuses.md             # Status definitions
  experimental-phase.md   # Experimental repo guide
  pr-review.md            # PR review in target repos
proposals/
  template.md             # Proposal template
  NNNNN_slug.md           # Individual proposals
  NNNNN_slug.asset-NAME.md # Supporting proposal assets
```

Keep supporting documents beside their parent proposal, using the parent's
filename stem followed by `.asset-` and a descriptive name. Assets inherit
the parent's metadata and lifecycle; they are not separate proposals.

## Links

- [Proposal Process](docs/process.md)
- [Status Definitions](docs/statuses.md)
- [Experimental Phase Guide](docs/experimental-phase.md)
- [Pull Request Review](docs/pr-review.md)
- [Proposal Template](proposals/template.md)
- [Contributing](CONTRIBUTING.md)

## License

Apache 2.0


[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fpraxis-proxy%2Fenhancements.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fpraxis-proxy%2Fenhancements?ref=badge_large)
