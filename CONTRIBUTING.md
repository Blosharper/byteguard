# Contributing

Thanks for looking. byteguard is a build tool for casual source code protection:
it raises the cost of reading a shipped bundle and is **not encryption or DRM**.
Read the opening of [README.md](README.md) before a first patch, because that scope
decides most review questions.

## Issues first

Open an issue before a substantial change, so the decision and its reasons are written
down before the code. A good defect report names:

1. the versions of `byteguard`, `vite-plugin-byteguard`, Vite and Node, and the browser
   or Electron version if the failure is at runtime;
2. the plugin options you used, with secrets replaced by placeholders;
3. the smallest input that fails, what happened, verbatim, and what you expected.

⚠️ **Never post a real key or a protected bundle from a product you ship.** This
repository is public. Reproduce with a throwaway key and a small test bundle.

Recovering source from a protected bundle with enough effort is within the stated
limits, not a defect. A report is welcome when the output is weaker than the
documentation says, or when a claim in the documentation is wrong.

## Before you open a pull request

CI's `test` job runs these commands on Node 22:

```bash
npm ci
npm run build
npm test
```

The build comes before the tests on purpose. `vite-plugin-byteguard` generates its
types against the built core, and `packages/byteguard/test/dist.test.ts` checks the
built package, because the loader inlines bundled code that `src` tests cannot see.
That test is skipped when `dist` is absent, so run the build first.

To work on one package, run `npm test` inside `packages/byteguard` or
`packages/vite-plugin-byteguard`.

## What a change has to clear

- **The core stays bundler-agnostic.** `packages/byteguard` has no `vite` or `rollup`
  imports. Bundler behaviour belongs in an adapter.
- **The binary format is a contract.** A format change updates the *Binary format*
  section of [`packages/byteguard/README.md`](packages/byteguard/README.md) and
  `test/format-fixture.test.ts`, and keeps older containers readable. Read the note at
  the end of that section before bumping the version byte.
- **Claims stay honest.** Do not describe the output as encrypted, secure or
  tamper-proof beyond what the code does.
- **Behaviour and documentation move together.** A new option updates the package
  README and has a test.

## Commits and pull requests

[Conventional Commits](https://www.conventionalcommits.org/), with a scope where one
fits: `feat(vite):`, `fix(runtime):`, `docs(readme):`. Subject and body in English,
one unit of work per commit.

Every change reaches `main` as a pull request and is **squash-merged**, so:

- **The pull request title is the commit subject on `main`** and must itself be a
  Conventional Commit subject. For a single-commit branch, GitHub uses that commit's
  subject, so keep the two the same. release-please reads the subject to pick the next
  version: `feat` bumps the minor, `fix` and `perf` the patch. While a package is below
  1.0.0, a breaking change (`!` or a `BREAKING CHANGE:` footer) bumps the minor. Other
  types do not cut a release on their own. See [RELEASING.md](RELEASING.md).
- **The squash body is made of the branch's commit messages**, not of the pull request
  description. Footers that must reach `main`, such as `Closes #N` or
  `BREAKING CHANGE:`, go in a commit message.
- **The required check is `test`.** It has to be green before the merge.

## Licence

Contributions are accepted under the MIT licence in [LICENSE](LICENSE).
