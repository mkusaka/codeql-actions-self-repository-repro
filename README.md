# codeql-actions-self-repository-repro

Minimal reproduction: CodeQL for GitHub Actions does not resolve local reusable workflows and composite actions referenced with the [self-repository syntax](https://github.blog/changelog/2026-07-30-reference-same-repository-actions-with-self-repository-syntax/) (`uses: $/...`). The same files referenced with the workspace-relative form (`uses: ./...`) are resolved.

## Layout

Each case has two copies that differ only in their first comment line and `name`. One copy is called with `./`, the other with `$/`.

| Caller | Callee (`./`) | Callee (`$/`) |
| --- | --- | --- |
| [`call-reusable.yml`](.github/workflows/call-reusable.yml) (`pull_request`) | [`reusable-dot.yml`](.github/workflows/reusable-dot.yml) | [`reusable-self.yml`](.github/workflows/reusable-self.yml) |
| [`call-action.yml`](.github/workflows/call-action.yml) (`pull_request_target`) | [`echo-dot`](.github/actions/echo-dot/action.yml) | [`echo-self`](.github/actions/echo-self/action.yml) |

[`codeql.yml`](.github/workflows/codeql.yml) runs `github/codeql-action` v4.38.2 (CodeQL 2.27.1, `codeql/actions-queries` 0.6.36) with `languages: actions` and `queries: security-extended`. `call-reusable` and `call-action` are disabled in the repository settings; they exist only to be analyzed.

## Results

| File | `./` callee | `$/` callee |
| --- | --- | --- |
| Reusable workflow | No alert | 2 × `actions/code-injection/medium` on `github.event_name` and `github.head_ref` |
| Composite action | `actions/code-injection/medium` with a data flow from `github.event.pull_request.title` in `call-action.yml` | `actions/code-injection/medium` with no data flow; the source is the action's own `inputs.text` |

So:

1. **Reusable workflow — false positives.** With `$/`, the reusable workflow has no resolved caller. It is then treated as privileged and externally triggerable, although its only caller runs on `pull_request`.
2. **Composite action — caller context lost.** With `$/`, the call site in `call-action.yml` is not connected to the action. The alert loses the path from the caller's untrusted source, and the action is analyzed as if it had no caller.

## Cause

In `actions/ql/lib/codeql/actions/ast/internal/Ast.qll` (`github/codeql` main):

- A call is matched to a callable by name: `viableCallable(DataFlowCall c) { c.getName() = result.getName() }` (`DataFlowPrivate.qll`).
- The callable's name comes from `getResolvedPath()` of `ReusableWorkflowImpl` / `CompositeActionImpl`, which only produces `""` and `"./"` prefixed paths.
- The call's name comes from `getCallee()`. For steps it is the raw `uses:` value without `@ref`, and for jobs it only strips a leading `./` (`u.getValue().matches("./%")`).

A `$/...` callee therefore never equals any resolved path.

## Alerts

- [Code scanning alerts](https://github.com/mkusaka/codeql-actions-self-repository-repro/security/code-scanning)
- The SARIF file is attached to each `codeql` run as the `sarif` artifact.
