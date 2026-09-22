# hermes-plugin-freight-reconciliation

Hermes Agent portable plugin (Portable Agent Plugins v1) for the
**海纳·运费对账 / Haina Freight Reconciliation** expert — payer-side,
deterministic, local-first reconciliation of carrier bills against trip
ledgers and contracted rates.

The product ships in two host forms that share one engine
(`src/freight_core` + `adapters/` in
[freight-reconciliation](https://github.com/harrylabsj/freight-reconciliation)):

| Form | Package | Host |
| --- | --- | --- |
| WorkBuddy expert | `dist/host-plugin/freight-reconciliation/` (built by `scripts/build_host_package.py`) | WorkBuddy / CodeBuddy |
| Hermes plugin | this repository (`plugin.json` + `mcp.json` + `skills/`) | Hermes Agent |

## What the plugin does

- 14 MCP tools (server key `fr`): case creation, source registration, mapping /
  rate-rule preparation, deterministic re-calculation runs, paged issue lists,
  four-panel evidence, review/export preparation.
- The arithmetic and all business facts live in the local Core (SQLite, WAL);
  the model only sees redacted projections. Amounts are integer minor units;
  unknown is `null`, never `0`.
- **Never** approves payments, sends messages, debits accounts or writes back
  to ERP/TMS. `prepare_*` tools only draft candidates; confirmation happens on
  a local admin page the model cannot reach.
- A connector crash never strands a run: jobs left QUEUED/RUNNING are marked
  `FAILED/RUN_INTERRUPTED` on next startup with a `recovery_action`; inputs are
  frozen by digest so the re-run is deterministic.

## Layout

```text
plugin.json   manifest (license: MIT)
mcp.json      stdio MCP server, pinned: uvx --from git+.../freight-reconciliation@v0.3.0 freight-mcp
skills/       3 workflow skills, English SKILL.md authoritative + SKILL.zh-CN.md variants
LICENSE       MIT
```

The MCP server key is deliberately short (`fr`): Hermes tool line names
(`mcp__agent_plugin_<name>_<hash>__<key>__<tool>`) have a 64-character budget,
and the plugin name `freight-reconciliation` is long.

## Data root

Case library lives at `${PLUGIN_DATA}/freight-reconciliation` (injected via
`FREIGHT_RECON_ROOT`). Keep it on a local disk — WAL-mode SQLite must not sit
on a network share.

## Local admin page (human confirmations)

```bash
read -s -p 'admin token: ' FREIGHT_ADMIN_TOKEN; echo; export FREIGHT_ADMIN_TOKEN
export FREIGHT_RECON_ROOT=<same root as the connector>
uvx --from git+https://github.com/harrylabsj/freight-reconciliation@v0.3.0 freight-admin
# open http://127.0.0.1:8765/
```

## Self-test before a catalog PR

```bash
hermes plugins validate <this-dir>
# MCP probe from a neutral directory (no cwd dependency):
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | uvx --from git+https://github.com/harrylabsj/freight-reconciliation@v0.3.0 freight-mcp
# expect: 14 tools
```

An anonymized synthetic bill/trips/rates sample set lives in the engine repo at
`examples/anonymized/` (with an engine-generated golden summary) and is covered
by `tests/integration/test_anonymized_samples.py`; crash-recovery behavior by
`tests/integration/test_job_recovery.py`.
