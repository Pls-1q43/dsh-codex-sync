# Live Skills and MCP compatibility

## Scope

The compatibility update admits the **0.2.1-rc.2 MCP client** explicitly, without a version exemption. It keeps the existing package identity, commands, UI response shapes, per-item state schema and within-layer Codex skill rank (800). The import/export and reverse-installer implementations are not redesigned by this update.

Home resolution is `config.codexHome`, then a nonempty `CODEX_HOME`, then `~/.codex`. Directory overrides are generic configuration, not application-installation or user-specific paths. No runtime inventory, credentials, personal acceptance record or profile configuration belongs in the package or PR.

## Skills

- Discover one level of `<home>/skills/<entry>/SKILL.md` or `<home>/skills/<entry>.md`; hidden roots and nested example skills are not recursively imported.
- YAML descriptions and invocation controls are parsed as YAML. Empty/invalid metadata, invalid names/policies, empty bodies and files larger than 1 MiB are omitted; malformed invocation policies never default to granting access. A missing name retains the directory/filename fallback when it is a valid skill name.
- Changes call the owning registration's `control.invalidate()`, which clears the current DSH registry cache. There is no call to a nonexistent shared `skills.invalidate()` method.
- Watch additions, deletion, rename and frontmatter changes. A fixed-root probe recovers missing/recreated roots, and a two-second metadata-only signature fallback covers watcher startup races or lost OS events; resource subtrees are not recursively watched. Bodies are reread on load. The Node-20-compatible watcher does not require native addons.
- Keep master/default/per-item/select-all preferences and raw entry identities. A renamed directory is a new item. Registration unload aborts observation and awaits watcher closure. Flat Markdown resources resolve relative to the skills directory, not the Markdown file.
- Within one registration layer, rank 800 remains below DSH's own providers; nearer Agent/preset layers still win regardless of rank. Already-logged skill content is not rewritten.

## MCP

The optional native client is loaded through the Host Loader when available, with the npm peer fallback for other deployments. No application or executable path is embedded. Explicit clients retain dependency waiting and native reconnect after initial errors. Mirrored clients report `mounted` only after their initial activation/discovery succeeds, then rely on native reconnect; initial failures retry at a bounded interval.

A complete TOML parser supports quoted keys, comments, literal strings, multiline/comma-containing arrays and inline/subtable maps. A read/parse failure keeps the last-good generation and reports a redacted diagnostic. An authoritative valid configuration that removes/disables a server revokes it.

| Codex field | Mapping / constraint |
| --- | --- |
| `command`, `args`, `env` | stdio; env values are literal, matching Codex rather than interpolating `${VAR}` |
| `env_vars` | whitelist: copy only named variables present in the Host environment; explicit env wins |
| `cwd` | absolute or `~/` path; relative paths are rejected instead of guessing a cross-application base |
| `url` | HTTP/HTTPS Streamable HTTP, without URL userinfo |
| `http_headers`, `env_http_headers` | static/environment headers with case-insensitive override semantics |
| `bearer_token_env_var` | explicit in-memory bearer header; missing references fail closed |
| `startup_timeout_sec` / `startup_timeout_ms` | full executable-lookup + activation deadline; default 20 seconds, maximum 5 minutes |
| `tool_timeout_sec` | native tool timeout; default 60 seconds, maximum 5 minutes |
| `enabled_tools`, nonempty `disabled_tools`, `tool_approval` | unsupported by the native client: skip/report rather than broaden tool access |
| `enabled = false` | never start |

A native server namespace must match `[A-Za-z0-9_-]{1,32}`; invalid names are reported per server, not normalized into ambiguous aliases. Reverse-bridge exclusions check known bridge identities and launch entrypoints, not arbitrary data path substrings. Existing deny/only, explicit-wins and per-item policies remain in force. POSIX stderr silencing preserves argument boundaries; on Windows it is not applied, so the plugin never assumes `/bin/sh` exists.

Parent-directory observation plus a bounded 2-second fallback detects atomic replacement, repeated edits and late-created homes. Configure `mcpMirrorPollIntervalMs` (50–60000 ms) or `mcpStartupTimeoutMs` (1–300000 ms) when needed. Each generation owns an AbortController and its actual Cordis Fiber. New configuration admission is independent of unrelated handshakes **and teardowns**. Only same-namespace replacement startup waits for its prior generation's closure; global unload joins all active/retiring generations.

**Trust boundary:** enabling mirroring trusts enabled commands/endpoints in the current and future Codex configuration. MCP commands run with plugin privileges and may invoke package managers; this is not per-tool approval. The bridge does not read Codex authentication/OAuth stores or inherit every secret-shaped environment variable. Logged status can contain the configured local path and server names, so treat those existing user-requested status outputs as private. The plugin creates no persistent runtime-status artifact in its source or install tree.

## Reproducible verification

All new fixtures are synthetic; tests never read a personal Codex directory or use a real service/account. Keep the historical migration fixture dependency graph separate from current DSH peers.

```sh
npm ci --ignore-scripts
npm test
npm run test:current
# Use Node 24 for the locked current SDK/transitive engine requirements:
npm run test:native
npm run test:package
```

- Main suite: existing session/client/command regressions plus actual Cordis lifecycle, full-TOML mapping, full-pipeline cancellation, independent revocation and atomic-save tests.
- `test:current`: isolated DSH SkillRegistry 0.2.1-rc.2 / Cordis 4.0.5; populate real caches, then test commands/HTTP toggles and external file changes, missing roots and unload.
- `test:native`: isolated native MCP client 0.2.1-rc.2. A hermetic stdio fixture answers initialize/list/call; verify discover/call/dispose/re-add, recovery after initial failure and late arrival of the tools service. No network endpoint is used.
- Packaging gate: inspect the **actual tarball**, require runtime/schema/locale/icon/license resources, and reject runtime artifacts, private paths, credential-shaped literals and local deployment documents.

CI exercises the portable contract suites on Node 20/22 and current native SDK fixtures on Node 24. Composition checks are separate from lifecycle/protocol tests; a successful `--dump-config` is not a boot or visual acceptance claim. No full Desktop GUI, restart, or all-host/all-platform certification follows merely from these tests. The unchanged legacy peer range is not a claim that every historical/future runtime has been newly verified.
