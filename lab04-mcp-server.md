# Lab 4: Stand up and integrate an MCP server

| | |
|---|---|
| **Module** | 2 – Tool Design & MCP Integration (Domain 2) |
| **Duration** | 35–40 minutes |
| **Level** | Intermediate |
| **Prerequisite** | Lab 0 completed (Claude Code and `mcp>=2` installed) |
| **Cost** | A few cents |

## What you will build

An MCP server that exposes two tools and one resource, tested from Python, then connected to Claude Code twice: over **stdio** (local process) and over **Streamable HTTP** with a **bearer token**.

```
                   +---------------------------+
 Claude Code  -->  |  orders_server.py         |
   (client)        |  tools:    get_order_status
                   |            list_customer_orders
  stdio  : Claude Code starts the process      |
  HTTP   : http_server.py + token check (401 otherwise)
                   |  resource: policy://refunds
                   +---------------------------+
```

You will learn: tools vs resources, input validation on the server, structured errors, local vs remote transports, authentication, and how Claude Code registers servers.

## Steps

### 1. Create the lab folder

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
mkdir lab04
cd lab04
```

### 2. Write the MCP server

```powershell
notepad orders_server.py
```

```python
"""Lab 4: a small MCP server exposing order tools and a policy resource."""
from typing import Annotated

from pydantic import Field
from mcp.server import MCPServer

mcp = MCPServer("orders-demo")

ORDERS = {
    "ORD-10001": {"customer_id": "C-1", "status": "shipped", "carrier": "BlueDart", "eta": "2026-10-14", "total": 49.99},
    "ORD-10002": {"customer_id": "C-1", "status": "processing", "carrier": None, "eta": None, "total": 129.00},
    "ORD-10003": {"customer_id": "C-2", "status": "delivered", "carrier": "DTDC", "eta": "2026-10-02", "total": 15.50},
}

OrderId = Annotated[str, Field(pattern=r"^ORD-\d{5}$", description="Order ID such as ORD-10001")]


@mcp.tool()
def get_order_status(order_id: OrderId) -> dict:
    """Look up the status of ONE order by its ID. Use when a customer asks where an
    order is. Do not use for refunds or for listing orders. Returns status, carrier, ETA."""
    order = ORDERS.get(order_id)
    if order is None:
        return {"is_error": True, "error_category": "not_found", "retryable": False,
                "message": f"No order {order_id}", "hint": "Ask the customer to confirm the ID"}
    return {"order_id": order_id, **order}


@mcp.tool()
def list_customer_orders(customer_id: Annotated[str, Field(pattern=r"^C-\d+$", description="Customer ID such as C-1")]) -> list[dict]:
    """List all orders for ONE customer. Returns order_id, status and total for each."""
    return [{"order_id": oid, "status": o["status"], "total": o["total"]}
            for oid, o in ORDERS.items() if o["customer_id"] == customer_id]


@mcp.resource("policy://refunds")
def refund_policy() -> str:
    """Refund policy text."""
    return "Refunds are allowed within 30 days of delivery. Orders above 100 USD need manager approval."


if __name__ == "__main__":
    mcp.run()  # stdio by default
```

*Why, piece by piece:*
- `MCPServer("orders-demo")` creates the server. The package must be version 2 (`pip show mcp`).
- `@mcp.tool()` turns a function into a tool. The **docstring becomes the tool description**, and the type hints become the input schema. So the docstring says when to use it and when not to.
- `OrderId` adds a regex pattern. Bad input is rejected by the server before your code runs.
- A missing order returns a **structured error** dict (category, retryable, hint) instead of crashing.
- `@mcp.resource("policy://refunds")` exposes read-only context. Tools are actions the model chooses to call. Resources are data the application can load.
- `mcp.run()` serves over stdio, which is how Claude Code launches local servers.

### 3. Test the server from Python

```powershell
notepad test_stdio.py
```

```python
import asyncio, sys
from mcp import Client, StdioServerParameters


async def main():
    params = StdioServerParameters(command=sys.executable, args=["orders_server.py"])
    async with Client(params) as client:
        tools = await client.list_tools()
        print("tools:", [t.name for t in tools.tools])
        r = await client.call_tool("get_order_status", {"order_id": "ORD-10001"})
        print("result:", r.structured_content or r.content)
        r = await client.call_tool("get_order_status", {"order_id": "bad"})
        print("bad id:", r.is_error, r.content)
        res = await client.read_resource("policy://refunds")
        print("resource:", res.contents[0].text)

asyncio.run(main())
```

```powershell
python test_stdio.py
```

Expected output (shortened):

```
tools: ['get_order_status', 'list_customer_orders']
result: [TextContent(type='text', text='{ "order_id": "ORD-10001", "customer_id": "C-1", "status": "shipped", ...')]
bad id: True [TextContent(type='text', text="Error executing tool get_order_status: ... String should match pattern '^ORD-\\d{5}$' ...")]
resource: Refunds are allowed within 30 days of delivery. Orders above 100 USD need manager approval.
```

*Why:* the test client starts your server as a child process and talks to it the way Claude Code will. The `bad id: True` line shows validation failing with a clear message (`is_error` is true).

### 4. Connect the stdio server to Claude Code

Get the full paths, then register the server:

```powershell
$py  = (Get-Command python).Source
$srv = (Resolve-Path .\orders_server.py).Path
claude mcp add --transport stdio orders -- $py $srv
claude mcp list
```

*Why:* Claude Code must start the server with the **virtual environment's** Python, because that is where `mcp` is installed. The `--` separates Claude's options from the command to run. `claude mcp list` should show `orders` and its status.

### 5. Use the tools from Claude Code

```powershell
claude
```

Inside Claude Code, type `/mcp` to see the server, then ask:

```
What is the status of order ORD-10001, and which orders does customer C-1 have?
```

Approve the tool calls when asked. Exit with `/exit`.

*Why:* Claude reads the descriptions you wrote, picks `get_order_status` and `list_customer_orders`, and shows the results. Try `ORD-99999` too and watch it handle the structured `not_found` error.

### 6. Write the HTTP version with authentication

```powershell
notepad http_server.py
```

```python
"""Lab 4: serve the same MCP server over Streamable HTTP with bearer-token auth."""
import hmac
import os

import uvicorn
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

from orders_server import mcp

TOKEN = os.environ.get("MCP_TOKEN")
if not TOKEN:
    raise SystemExit("Set MCP_TOKEN first, e.g. $env:MCP_TOKEN = 'lab-secret-123'")


class BearerAuth(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        supplied = request.headers.get("authorization", "")
        expected = f"Bearer {TOKEN}"
        if not hmac.compare_digest(supplied, expected):
            return JSONResponse({"error": "unauthorized"}, status_code=401)
        return await call_next(request)


app = mcp.streamable_http_app()
app.add_middleware(BearerAuth)

if __name__ == "__main__":
    uvicorn.run(app, host="127.0.0.1", port=8000)
```

*Why:* the same `mcp` object is served over HTTP, so one set of tools works for local and remote use. A small middleware checks the `Authorization: Bearer ...` header with `hmac.compare_digest` (safe against timing attacks) and returns **401** otherwise. The token comes from an environment variable, never from code.

### 7. Run the HTTP server

Open a **second** PowerShell window, activate the environment, then:

```powershell
cd C:\cca-labs\lab04
$env:MCP_TOKEN = "lab-secret-123"
python http_server.py
```

Leave it running. The endpoint is `http://127.0.0.1:8000/mcp`.

### 8. Prove the authentication works

Back in the first window:

```powershell
notepad test_http.py
```

```python
import os
import httpx

URL = "http://127.0.0.1:8000/mcp"
HEADERS = {"Accept": "application/json, text/event-stream", "Content-Type": "application/json"}
INIT = {"jsonrpc": "2.0", "id": 1, "method": "initialize",
        "params": {"protocolVersion": "2025-03-26", "capabilities": {},
                   "clientInfo": {"name": "lab-test", "version": "0.1"}}}

r = httpx.post(URL, json=INIT, headers=HEADERS)
print("no token   ->", r.status_code)

r = httpx.post(URL, json=INIT, headers={**HEADERS, "Authorization": "Bearer wrong"})
print("wrong token->", r.status_code)

r = httpx.post(URL, json=INIT, headers={**HEADERS, "Authorization": f"Bearer {os.environ['MCP_TOKEN']}"})
print("right token->", r.status_code)
print(r.text[:200])
```

```powershell
$env:MCP_TOKEN = "lab-secret-123"
python test_http.py
```

Expected output:

```
no token   -> 401
wrong token-> 401
right token-> 200
```

*Why:* only the request with the correct token gets through. This is the minimum for any server reachable over a network.

### 9. Register the HTTP server with Claude Code

```powershell
claude mcp add --transport http orders-http http://127.0.0.1:8000/mcp --header "Authorization: Bearer lab-secret-123"
claude mcp list
```

Start `claude`, run `/mcp`, and ask for the status of ORD-10002.

*Why:* the header is stored in your local Claude Code config. For a shared team setup you would put the server in a project `.mcp.json` file and reference the secret as `${MCP_TOKEN}` so it is not committed.

### 10. Know the scopes

```powershell
claude mcp add --help
```

Find the `--scope` option.

*Why:* **local** (default) is private to you in this project, **project** writes `.mcp.json` for the team to share through git, **user** applies to all your projects. Pick the narrowest scope that works.

## Verify

- `python test_stdio.py` lists both tools and rejects `bad`.
- `test_http.py` shows 401, 401, 200.
- In Claude Code, both `orders` and `orders-http` appear under `/mcp` and answer questions about ORD-10001.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ImportError: cannot import name 'MCPServer'` | You have MCP v1. Run `pip install -U "mcp>=2"`. |
| `claude mcp list` shows `orders` as failed | The Python path is wrong. Re-run the `$py` and `$srv` lines with the environment active, then `claude mcp remove orders` and add again. |
| `Set MCP_TOKEN first` | Set `$env:MCP_TOKEN` in the **same** window that runs the script. |
| Connection refused in step 8 | The HTTP server window must still be running. |
| Port 8000 already in use | Change `port=8000` in `http_server.py` and in the URLs. |

## Stretch goals

1. Add a `cancel_order` tool that is marked as destructive in its description and takes a `confirm: bool` parameter.
2. Add a second resource `orders://{order_id}` and compare using it versus the tool.
3. Create a project-scope `.mcp.json` that uses `${MCP_TOKEN}`, and check it with `claude mcp list`.

## Exam takeaways

- **Tools** are actions the model chooses; **resources** are data the app loads.
- Docstrings and types become the tool contract. Write them for the model.
- Validate inputs on the server and return **structured errors**.
- **stdio** for local servers, **Streamable HTTP** for remote ones, and always authenticate remote servers.
- Scope matters: local, project, user.

## Cleanup

Stop the HTTP server with Ctrl+C in the second window, then:

```powershell
claude mcp remove orders
claude mcp remove orders-http
```
