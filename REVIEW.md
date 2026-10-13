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
version). Tests run by hand with `uv run pytest tests/ -v`. In
`tests/test_integration.py` the tests marked `skipif` skip when `npx`
is missing, and `test_connection_lifecycle` then checks only the
disconnected state; `tests/test_http_integration.py` connects to the live
`https://mcp.deepwiki.com/mcp` and does not skip.

Always flag:
- a wrapper error path that returns `ToolResult(success=False)` without
  the error text in `output`, which is what makes the error visible to
  the model (commit 94aec17).
- a generated tool, resource or prompt name that is not
  `mcp_<server>_...` or does not match `^[a-zA-Z0-9_-]{1,128}$`, the
  provider rule cited in `tests/test_resources.py`. The wrappers replace
  other characters with `re.sub` but do not cap length (commit a543d84).
- a Pydantic object such as `AnyUrl` passed through to a definition or
  output instead of a plain `str`; tool definitions are serialized to
  JSON (commit a543d84).
- in either transport, a change to the background-task lifecycle: the
  transport context and its `ClientSession` are entered and exited by
  the same task, and `disconnect()` signals that task and awaits it
  (anyio's same-task rule, commit cb92cb4).
- in either transport's `_connection_task_impl`, `except Exception`
  where `except BaseException` is used, or a path that exits before
  readiness without setting `_ready_event`; `connect()` then hangs.
  In either transport, also flag a cancellation recorded as
  `_last_error` or a circuit-breaker failure. In the HTTP client, flag
  removal of the `CONNECTION_TIMEOUT` wait or of the
  `_extract_root_cause` unwrap of `ExceptionGroup` (commits 21a4207,
  b32b7ce).
- a change that breaks the 2- or 3-tuple unpack of
  `streamable_http_client`; downstream pins use both SDK shapes
  (commit b32b7ce).
- a stdio server started without the parent environment: `client.py`
  merges `{**os.environ, **self.env}`, so server config overrides
  inherited variables and `PATH` and API keys still reach the server.
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
- a change to the server config priority. The order is inline, then
  `AMPLIFIER_MCP_CONFIG` as an explicit override, then project
  `.amplifier/mcp.json`, then user `~/.amplifier/mcp.json` (`config.py`,
  commit 2031343, following amplifier-foundation). The README and
  `behaviors/mcp.yaml` still list the environment variable last; a
  change to either should bring it in line with the code.
- a server `env` or `headers` value logged or put in a tool result,
  whether literal or from `${VAR}` substitution; these fields carry
  connection strings and auth headers (README "Configure MCP Servers").
- an async test that is not marked for asyncio, at function, class or
  module level (`asyncio_mode = "strict"`), or a new test that needs
  `npx` or a live server without a skip when it is missing.
- a version bump that does not change `pyproject.toml`,
  `amplifier_module_tool_mcp/__init__.py` and the README status line
  together. That line also states a test count (52), which is already
  stale; do not let a change make it more so.
