# Review instructions — amplifier-module-tool-mcp

Commit and range reviews read this file from the default branch; dirty
reviews and local runs read the working copy.

An Amplifier tool module that connects to MCP servers over stdio and
Streamable HTTP and mounts their tools, resources and prompts as
Amplifier tools (README "Overview"). This repo is Joi's fork of
`microsoft/amplifier-module-tool-mcp`; the module is loaded through the
`amplifier.modules` entry point `tool-mcp` and `behaviors/mcp.yaml`.

Already enforced (do not repeat): nothing. There is no CI and no git
hook, and nothing runs ruff (`pyproject.toml` only sets its target
version). Tests run by hand with `uv run pytest tests/ -v`.
`tests/test_integration.py` skips when `npx` is missing;
`tests/test_http_integration.py` connects to the live
`https://mcp.deepwiki.com/mcp` and does not skip.

Always flag:
- a wrapper error path that returns `ToolResult(success=False)` without
  the error text in `output`, which is what makes the error visible to
  the model (commit 94aec17).
- a tool, resource or prompt name that is not `mcp_<server>_...` or that
  keeps characters outside `[a-zA-Z0-9_-]`. All three wrappers sanitize
  with the same `re.sub`, and the provider API rejects other names
  (commit a543d84).
- a Pydantic object such as `AnyUrl` passed through to a definition or
  output instead of a plain `str`; tool definitions are serialized to
  JSON (commit a543d84).
- in either transport's `_connection_task_impl`, `except Exception`
  where `except BaseException` is used, or any exit path that does not
  set `_ready_event`; `connect()` then hangs. Also flag removal of the
  `CONNECTION_TIMEOUT` wait in the HTTP client (commits 21a4207,
  b32b7ce).
- a change that breaks the 2- or 3-tuple unpack of
  `streamable_http_client`; downstream pins use both SDK shapes
  (commit b32b7ce).
- wrapper content returned without `truncate_content_if_needed` and
  `max_content_size`, or a dropped `content_size_chars` /
  `content_truncated` field (README "Configuration",
  `CONTENT_SIZE_PROTECTION.md`).
- one server's failure, including `asyncio.CancelledError`, stopping
  `MCPManager.start()` from starting the others.
- the visibility hook registering, or startup warning or failing, when
  no servers are configured. The module is meant for always-on
  inclusion with `--app` (commit 7247430, README "Installation").
- server stderr shown in the console when `verbose_servers` is false;
  by default it goes to `server_log_dir` (README "Configuration").
- a change to the server config priority without matching docs. The
  code order is inline, then `AMPLIFIER_MCP_CONFIG`, then project
  `.amplifier/mcp.json`, then user `~/.amplifier/mcp.json` (`config.py`,
  commit 2031343); the README and `behaviors/mcp.yaml` still list the
  environment variable last.
- a logged or returned value after `${VAR}` substitution in server
  `env` or `headers`; those carry connection strings and auth headers.
- an async test without `@pytest.mark.asyncio` (`asyncio_mode =
  "strict"`), or a new test that needs `npx` or a server without a skip
  when it is missing.
- a version bump that does not change `pyproject.toml`,
  `amplifier_module_tool_mcp/__init__.py` and the README status line
  together.
