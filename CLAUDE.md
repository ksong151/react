# React

The React monorepo: the `react` runtime (Fiber reconciler, DOM/Native/ART renderers, Fizz streaming SSR, Flight/RSC), DevTools, and the React Compiler. Runtime code is Flow. `compiler/` is TypeScript plus an in-progress Rust port, a separate Yarn + Cargo workspace with its own `compiler/CLAUDE.md`. The tree is mirrored into Meta's internal monorepo.

## Architecture
- `packages/react-dom` is thin public wrappers; the real DOM code is the private `packages/react-dom-bindings`. `packages/react-reconciler` is Fiber. `packages/react-server` is Fizz + the Flight server; `packages/react-client` is the Flight client; `packages/react-server-dom-{webpack,turbopack,parcel,esm}` are bundler bindings.
- Renderers plug in via forks: `packages/react-reconciler/src/ReactFiberConfig.js` throws "must be shimmed" and `scripts/rollup/forks.js` swaps in `src/forks/ReactFiberConfig.<renderer>.js` at build time. Renderer names live in `scripts/shared/inlinedHostConfigs.js` and degrade `dom-node-webpack` → `dom-node` → `dom`.
- Entry variants sit beside each `index.js`: `.stable.js`, `.experimental.js`, `.fb.js`, `*.development.js`, picked by release channel. `*.react-server.js` is the `react-server` export condition. `npm/` dirs hold hand-written ES5 CJS shims.
- Flags: `packages/shared/ReactFeatureFlags.js` plus one fork per target in `packages/shared/forks/`. Test helpers (`act`, `assertLog`, `waitForAll`, `assertConsoleErrorDev`) come from `packages/internal-test-utils`.
- Compiler pipeline: `compiler/packages/babel-plugin-react-compiler/src/Entrypoint/Pipeline.ts`; fixtures in `src/__tests__/fixtures/compiler/`; Rust crates in `compiler/crates/`.

## Commands
All lines run from the repo root. Node 20.19.0, Yarn 1.
```bash
yarn install --frozen-lockfile && yarn --cwd compiler install --frozen-lockfile   # compiler has its own lockfile
yarn test ReactHooks                       # -r stable|www-modern|xplat, --prod, --variant=false, --build
yarn linc && yarn prettier                 # lint + format changed files; yarn lint / yarn prettier-all for everything
yarn flow dom-node                         # Flow is per renderer; yarn flow-ci dom-node for a full check
yarn build react/index,react-dom/index --type=NODE_DEV -r=experimental   # subset build; bare yarn build is CI-sized
yarn --cwd fixtures/dom install && yarn --cwd fixtures/dom dev   # run; needs a full yarn build first
yarn extract-errors                        # after a build; regenerates scripts/error-codes/codes.json
yarn generate-inline-fizz-runtime          # regenerates the Fizz inline-code strings file
yarn --cwd compiler snap:build && yarn --cwd compiler snap -p simple-alias.js   # compiler fixtures; -u updates .expect.md, -d dumps HIR, --rust
yarn --cwd compiler workspace babel-plugin-react-compiler lint
cargo test --manifest-path compiler/Cargo.toml
```
Pre-commit order: `yarn prettier` → `yarn linc` → `yarn flow dom-node` → `yarn test <pattern>` → `yarn test-www <pattern>`.

## Conventions
- Runtime source (`packages/*/src`, not tests/scripts/devtools/compiler) is ES5-shaped Flow in `.js`: no `class`, `async`, `for…of`, `?.`, `??`; `shared/assign`, `shared/isArray`, `shared/hasOwnProperty` over the builtins. A closure over a loop `let` fails the build (`throwIfClosureRequired`) but passes Jest.
- Import across packages by package path (`'react-reconciler/src/…'`), never `../../`. Flow suppressions carry a code: `$FlowFixMe[incompatible-type]`.
- `console.error`/`warn` only inside a bare `if (__DEV__) {`; first arg a string literal whose `%s` count matches the args. `console['error']` is the sanctioned escape hatch.
- Every `new Error('…')` in `packages/` must already be in `scripts/error-codes/codes.json`, or be preceded by the literal comment `// eslint-disable-next-line react-internal/prod-error-codes`.
- Prod string coercion is `'' + x` right after `if (__DEV__) { checkXxxStringCoercion(x); }`; non-prod code uses `String(x)`.
- Naming: DEV-only identifiers end `DEV`/`InDev`, DEV fiber fields start `_debug`, warn-once sets are `didWarnAbout…`; `enableX` adds behavior, `disableX` removes legacy; `unstable_` exports live only in `.experimental.js`/`.fb.js` entries.
- A new flag goes in `ReactFeatureFlags.js` and every fork; the Flow cast at the bottom of each fork fails `yarn flow` otherwise. Branch on the named constant so the build can strip it.
- Tests: `__tests__/Foo-test.js`; suffix `-test.internal.js` if you mutate flags or import internals. `jest.resetModules()` then `require()` inside `beforeEach`. Scheduler is always mocked: `Scheduler.log('x')` in components, `assertLog([...])` in tests. Assert every console call with `assertConsoleErrorDev([...])`. A `// @gate flag` test must *fail* when the gate is false, so delete stale gates.
- PR body needs `## Summary` and `## How did you test this change?`. Squash merge; compiler subjects are `[compiler] …` or `[rust-compiler] …`.
- Compiler: `noInlineConfig` so `eslint-disable` comments do nothing. Fixture names are enforced: `error.*` must throw, others must not; `error.todo-*` is unsupported, `error.bug-*` is a known bug. Export `FIXTURE_ENTRYPOINT = {fn, params}` so Sprout can diff original vs compiled.

## Guardrails
- Never run `jest`, `eslint` or `flow` directly; the root jest config exists only to throw. Use the `yarn` scripts.
- Generated, do not hand-edit: `scripts/error-codes/codes.json` (append-only), `packages/react-dom-bindings/src/server/fizz-instruction-set/ReactDOMFizzInstructionSetInlineCodeStrings.js` (edit the sibling `ReactDOMFizzInline*.js`), `scripts/flow/*/.flowconfig` (edit `scripts/flow/config/flowconfig`), `flow-typed/**`, `compiler/**/*.expect.md`, `packages/react-devtools-shared/src/hooks/__tests__/__source__/__compiled__/**`.
- `yarn build` overwrites `packages/shared/ReactVersion.js` with a canary placeholder and never restores it; `git checkout` it before committing. Never touch `ReactVersions.js` or `version`/`peerDependencies` in `packages/*/package.json`; release tooling owns them.
- Root `yarn lint` and `yarn install` skip `compiler/` entirely; it has its own ESLint config and lockfile.
- Keep upstream shape in vendored code: `packages/eslint-plugin-react-hooks/src/code-path-analysis/`, `packages/react-server/src/createFastHashJS.js`, `fixtures/legacy-jsx-runtimes/`. Files marked `@noformat`/`@nolint` are copied verbatim into React Native.
- Rewording a warning or error is an API change: `scripts/print-warnings` exports every message to Meta. A new public entry point needs `index.*` variants, an `npm/` shim, a `scripts/rollup/bundles.js` entry and an `inlinedHostConfigs.js` line.
- `.claude/skills/feature-flags/SKILL.md` says gated tests are skipped. They are not; they must fail.
