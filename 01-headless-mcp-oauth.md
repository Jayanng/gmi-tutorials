# Running the GMI Cloud MCP Server on a Headless VPS

*How to connect an AI agent to GMI Cloud's remote MCP server when there is no browser on the box, plus the two bugs that will waste your evening.*

## What you get

GMI Cloud ships a remote MCP server at `https://mcp.gmicloud.ai/mcp`. Ten tools: `search_models`, `get_model`, `estimate_generation`, `submit_generation`, `get_generation`, `list_generations`, `upload_media`, `get_account_summary`, `search_docs`, `create_support_request`. Auth is OAuth 2.1 with PKCE. No API key to manage.

On a laptop this is a one-command setup. On a headless server it is a trap. Here is the working path.

## The trap

The standard CLI login flow for MCP OAuth opens a browser callback on `127.0.0.1:<port>`. On a remote box that page never loads. Worse, in our testing some CLI flows rotated their OAuth state roughly every 60 seconds, so by the time you approve on your phone and paste the callback back, the state has rotated and the exchange fails. Retrying in a loop makes it worse: in our runs each failed attempt registered a new OAuth client and burned another approval.

## The working method

1. **Kill stale login attempts.** One login flow at a time. Confirm the redirect port is free (`ss -ltnp | grep <port>`)
2. **Drive the flow manually.** Print ONE authorize URL and hold that state indefinitely. Do not rotate
3. **Approve in your phone browser.** The browser will fail to load `127.0.0.1:<port>/callback?code=...&state=...`. That failure is expected and fine
4. **Paste the full failed URL back to the server.** Your script validates the `state` parameter matches the one it issued, then exchanges the `code` at the token endpoint
5. **Store the token as the client expects it.** Then test: your MCP client should report a successful authenticated connection

Minimal exchange sketch:

```python
import requests

resp = requests.post(TOKEN_ENDPOINT, data={
    "grant_type": "authorization_code",
    "code": code_from_callback,
    "redirect_uri": redirect_uri,
    "client_id": client_id,
    "code_verifier": pkce_verifier,
})
tokens = resp.json()
# save access_token + refresh_token + expires_at
```

## Bug #1: `expires_at` must be a Unix float

This one is nasty. If you store the expiry as an ISO string, the token storage layer throws:

```
unsupported operand type(s) for -: 'str' and 'float'
```

...and the connection fails even though the token is perfectly valid. Store `expires_at` as `time.time() + data["expires_in"]`, a plain float.

## Bug #2: token expiry on the manual path

Native MCP tool integrations auto-refresh the access token. A hand-rolled driver does not, and starts returning 401 after about an hour (`expires_in` is 3600). Do not re-run OAuth. Refresh instead:

```python
resp = requests.post(TOKEN_ENDPOINT, data={
    "grant_type": "refresh_token",
    "refresh_token": saved_refresh_token,
    "client_id": client_id,
})
# rewrite the token file with the new access_token and a float expires_at
```

## Verify it works

Call a free tool first. `search_docs` or `get_account_summary` are read-only and cost nothing. If those answer, the connection is healthy and generation tools will work when your balance allows and the connection holds generation scope; a refusal on a funded account is often a scope problem, not a broken connection.

## Gotchas worth knowing

- MCP tool lists can go stale mid-session in some agent runtimes. If previously working tools suddenly report "not a deferrable tool", do not loop on it: reconnect or fall back to a direct HTTP driver
- Result objects come in two shapes: text (JSON status) and resource links (the output file URL). Branch on the type before assuming either
- In our testing the read calls stayed available at zero balance: `list_generations` and `get_generation` kept working when paid generation was refused, which makes them your recovery path when paid calls start failing

That is the whole setup. Once the token is stored correctly and refreshing, the agent gets ten GMI tools over one connection.

