# How to Sell an MCP Tool with x402 + Pyrimid

A practical step-by-step guide for AI agents and developers to monetize MCP servers using HTTP 402 Payment Required and the Pyrimid protocol.

## Overview

This guide walks through building a paid MCP tool that:
1. Offers a free `preview_*` tool (no payment needed)
2. Requires payment for the `buy_*` tool via x402 (HTTP 402)
3. Registers in the Pyrimid catalog for agent discovery
4. Collects USDC automatically on Base

## Step 1: Design Your Tool Pattern

Every paid MCP tool follows a preview/buy pattern:

| Tool | Auth | Purpose |
|------|------|---------|
| `preview_search` | Free | Returns schema, price, sample output |
| `buy_search` | x402 | Returns full paid result |

**Free tool** (`preview_*`): Shows the agent what's available, how much it costs, and what the output looks like. No payment required.

**Paid tool** (`buy_*`): Returns HTTP 402 with x402 payment requirements until the agent pays. After payment, returns the full result.

## Step 2: Implement the MCP Server

Create an MCP server with both tools:

```python
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio
import mcp.types as types
import json

app = Server("paid-mcp-server")

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="preview_search",
            description="Preview paid search - shows price and sample",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                }
            }
        ),
        types.Tool(
            name="buy_search",
            description="Execute paid search (requires x402 payment)",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                }
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "preview_search":
        return [types.TextContent(
            type="text",
            text=json.dumps({
                "price_usdc": 50000,  # $0.05 in micro USDC
                "sample": "Sample result for: " + arguments["query"],
                "schema": {"fields": ["title", "url", "relevance_score"]},
                "payment_required": True,
                "payment_method": "x402",
                "endpoint": "https://your-server.com/api/buy/search"
            })
        )]
    
    if name == "buy_search":
        # Check x402 payment header
        # If unpaid, return 402 with payment requirements
        raise types.Error(
            code=402,
            message="Payment Required",
            data={
                "payment": {
                    "network": "base",
                    "asset": "USDC",
                    "amount": "50000",
                    "receiver": "0xYourWallet...",
                    "metadata": {"product": "paid_search"}
                }
            }
        )
        # After payment verified, return actual results
        return [types.TextContent(
            type="text",
            text=json.dumps({"results": [...], "paid": True})
        )]

async def main():
    async with mcp.server.stdio.stdio_server() as (read, write):
        await app.run(read, write, InitializationOptions(
            server_name="paid-mcp-server",
            server_version="1.0.0"
        ))
```

## Step 3: Handle x402 Payment Flow

The x402 protocol works as follows:

1. Agent calls `buy_search` → Server returns **HTTP 402** with payment requirements
2. Agent reads the `X-Payment-Required` header containing:
   - Network (e.g., `base`)
   - Asset (e.g., `USDC`)
   - Amount (e.g., `50000` = $0.05)
   - Receiver address
3. Agent sends USDC on Base to the receiver
4. Agent retries the request with `X-Payment-Proof` header containing the tx hash
5. Server verifies the payment and returns the result

**402 Response Example:**

```json
{
  "error": "payment_required",
  "payment": {
    "network": "base",
    "asset": "USDC",
    "amount": "50000",
    "receiver": "0xYourWalletHere",
    "metadata": {
      "product_id": "paid_search",
      "vendor_id": "your-mcp-server"
    }
  }
}
```

**Client-side (agent) payment flow:**

```python
import httpx
from web3 import Web3

# Step 1: Try the paid tool
response = httpx.post("https://your-server.com/api/buy/search", json={"query": "AI agents"})

# Step 2: If 402, parse payment requirements
if response.status_code == 402:
    payment = response.json()["payment"]
    
    # Step 3: Send USDC
    w3 = Web3(Web3.HTTPProvider("https://base-rpc.publicnode.com"))
    usdc_contract = w3.eth.contract(
        address="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",  # USDC on Base
        abi=[{"constant":False,"inputs":[{"name":"_to","type":"address"},{"name":"_value","type":"uint256"}],"name":"transfer","outputs":[{"name":"","type":"bool"}],"type":"function"}]
    )
    tx = usdc_contract.functions.transfer(
        payment["receiver"],
        int(payment["amount"])
    ).transact({"from": agent_wallet})
    
    # Step 4: Retry with proof
    response = httpx.post(
        "https://your-server.com/api/buy/search",
        json={"query": "AI agents"},
        headers={"X-Payment-Proof": tx.hex()}
    )
    
    # Step 5: Get results
    results = response.json()
```

## Step 4: Register in Pyrimid Catalog

To make your paid MCP tool discoverable by agents, register it in the Pyrimid catalog:

**Product Metadata (pyrimid-product.json):**

```json
{
  "vendor_id": "your-mcp-server",
  "product_id": "paid_search",
  "name": "Enriched Web Search",
  "description": "Paid MCP search returning enriched results with citations and relevance scoring",
  "category": "search-scraping",
  "tags": ["mcp", "search", "x402", "paid-tools", "enriched-data"],
  "price_usdc": 50000,
  "affiliate_bps": 3000,
  "endpoint": "https://your-server.com/api/paid/search",
  "network": "base",
  "asset": "USDC",
  "documentation_url": "https://your-server.com/docs",
  "icon_url": "https://your-server.com/icon.png",
  "free_tool": "preview_search",
  "paid_tool": "buy_search"
}
```

Submit via Pyrimid's catalog API or GitHub:

```bash
# Option A: API submission
curl -X POST "https://api.pyrimid.ai/v1/catalog/products" \
  -H "Content-Type: application/json" \
  -d @pyrimid-product.json

# Option B: GitHub PR to pyrimid-ai/pyrimid
# Add your product to examples/mcp-paid-tool/
```

## Step 5: Agent Discovery

Make your paid tool discoverable:

1. **Server card**: Add MCP server card to your domain at `/.well-known/mcp.json`
2. **llms.txt**: Add `https://your-server.com/llms.txt` with tool descriptions
3. **agents.txt**: Add discovery hints for AI agents
4. **Pyrimid catalog**: Your product appears in `https://pyrimid.ai/catalog`

Example `llms.txt`:
```
# Paid MCP Server
/docs/api
/docs/tools
https://your-server.com/pyrimid-product.json
```

## Complete Working Example

Here's a minimal end-to-end example you can run:

**Server** (`server.py`): Implements preview/free and buy/paid tools with x402
**Client** (`client.py`): Shows how an AI agent discovers, pays, and consumes

[Full source at examples/mcp-paid-tool/](https://github.com/pyrimid-ai/pyrimid/tree/main/examples/mcp-paid-tool)

## Why This Works

- **For agents**: Standard x402 payment flow works with any agent framework
- **For vendors**: Set your price, get paid in USDC, no billing infrastructure needed
- **For affiliates**: Earn 3000 bps (30%) commission routing buyers to your tool
- **For the ecosystem**: All fees visible onchain, no gatekeepers

## Next Steps

1. Deploy your MCP server with preview/buy tools
2. Test the x402 flow with a Base wallet
3. Register in Pyrimid catalog
4. Add discovery files (llms.txt, agents.txt)
5. Start earning USDC from agent customers
