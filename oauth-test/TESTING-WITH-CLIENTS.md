# Testing with Real MCP Clients

## ✅ What's New

I've created **MCP-compliant** OAuth servers that implement:

### Required MCP Standards
- ✅ **RFC 8414** - Authorization Server Metadata
- ✅ **RFC 9728** - Protected Resource Metadata  
- ✅ **RFC 7591** - Dynamic Client Registration
- ✅ **OAuth 2.1** - Authorization Code Flow
- ✅ **PKCE** - Code Challenge/Verifier (S256)
- ✅ **JWT** - JSON Web Tokens with RS256
- ✅ **JWKS** - JSON Web Key Set endpoint
- ✅ **MCP Protocol** - Initialize, List Tools, Call Tools

## Files

### New MCP-Compliant Servers
- `mcp-auth-server.js` - Full OAuth 2.1 + JWT implementation
- `mcp-resource-server.js` - MCP server with tool execution
- `test-mcp-compliance.sh` - Automated compliance test

### Original Simple Servers (for learning)
- `auth-server.js` - Simple version (random tokens)
- `resource-server.js` - Simple version
- `client.js` - Browser-based client

## Quick Test

### 1. Test MCP Compliance
```bash
cd /Users/fiberta/work/DesktopCommanderMCP/oauth-test
./test-mcp-compliance.sh
```

This will:
- ✅ Start both servers
- ✅ Test all OAuth endpoints
- ✅ Register a client dynamically
- ✅ Get authorization code
- ✅ Exchange for JWT token
- ✅ Initialize MCP connection
- ✅ List tools
- ✅ Call tools
- ✅ Verify auth is required

## Testing with MCP Inspector

### Step 1: Start Servers

**Terminal 1:**
```bash
node mcp-auth-server.js
```

**Terminal 2:**
```bash
node mcp-resource-server.js
```

### Step 2: Run MCP Inspector

**Terminal 3:**
```bash
npx @modelcontextprotocol/inspector
```

This opens a web UI at `http://localhost:6274`

### Step 3: Connect to Your Server

In the MCP Inspector UI:

1. **Transport Type**: Select "Streamable HTTP"

2. **Server URL**: Enter `http://localhost:3002/mcp`

3. **Click "Connect"**

4. Inspector will:
   - Detect 401 response
   - Fetch `/.well-known/oauth-protected-resource`
   - Fetch `/.well-known/oauth-authorization-server`
   - Dynamically register a client
   - Open browser for you to login

5. **Login Page Opens**: Enter `admin` / `password123`

6. **Authorize**: Consent to the permissions

7. **Redirected Back**: Inspector gets the token

8. **Connected!**: Now you can:
   - See available tools
   - Call `get_user_info` tool
   - Call `echo` tool with a message

## Testing with Claude.ai (Requires Public URL)

Claude.ai **cannot** connect to `localhost`, so you need to:

### Option 1: Cloudflare Tunnel (Easiest)

```bash
# Install cloudflared
brew install cloudflared

# Start auth server tunnel
cloudflared tunnel --url http://localhost:3001

# Start resource server tunnel (in another terminal)
cloudflared tunnel --url http://localhost:3002
```

You'll get two URLs like:
- `https://random-words-123.trycloudflare.com` → Auth Server
- `https://random-words-456.trycloudflare.com` → Resource Server

**Update baseUrl in code:**
```javascript
// In mcp-auth-server.js, change:
const baseUrl = 'https://random-words-123.trycloudflare.com';

// In mcp-resource-server.js, update:
app.get('/.well-known/oauth-protected-resource', (req, res) => {
  res.json({
    resource: 'https://random-words-456.trycloudflare.com',
    authorization_servers: ['https://random-words-123.trycloudflare.com'],
    // ...
  });
});
```

### Option 2: ngrok

```bash
# Install ngrok
brew install ngrok

# Terminal 1: Auth server tunnel
ngrok http 3001

# Terminal 2: Resource server tunnel  
ngrok http 3002
```

### Option 3: Deploy to a Server

Deploy both servers to a real server with a public IP/domain.

### Then Add to Claude.ai

1. Go to Claude.ai
2. Settings → Integrations
3. Add Custom Integration
4. Enter your **resource server URL**: `https://your-tunnel.com/mcp`
5. Click "Connect"
6. Browser opens for login
7. Login with `admin` / `password123`
8. Authorize
9. ✅ Connected!

## Testing with ChatGPT (Not Yet Supported)

ChatGPT/OpenAI **does not yet support MCP protocol** as of now. They may add support in the future.

## Expected Behavior

### Without Token (401):
```bash
curl -X POST http://localhost:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

Response:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Authorization required"
  },
  "id": null
}
```

Header: `WWW-Authenticate: Bearer resource_metadata="..."`

### With Valid Token (200):
```bash
# (Get token first via OAuth flow)
curl -X POST http://localhost:3002/mcp \
  -H "Authorization: Bearer ***REMOVED***" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

Response:
```json
{
  "jsonrpc": "2.0",
  "result": {
    "tools": [
      {
        "name": "get_user_info",
        "description": "Get information about the authenticated user",
        "inputSchema": {...}
      },
      {
        "name": "echo",
        "description": "Echo back a message",
        "inputSchema": {...}
      }
    ]
  },
  "id": 1
}
```

## Available Tools

### get_user_info
Returns info about the authenticated user.

```json
{
  "method": "tools/call",
  "params": {
    "name": "get_user_info",
    "arguments": {}
  }
}
```

Response:
```json
{
  "username": "admin",
  "client_id": "abc-123",
  "scope": "mcp:tools",
  "message": "You are successfully authenticated with OAuth!"
}
```

### echo
Echoes back a message with your username.

```json
{
  "method": "tools/call",
  "params": {
    "name": "echo",
    "arguments": {
      "message": "Hello from OAuth!"
    }
  }
}
```

Response:
```json
{
  "content": [{
    "type": "text",
    "text": "Echo from admin: Hello from OAuth!"
  }]
}
```

## Architecture

```
┌─────────────────┐
│  MCP Inspector  │
│  or Claude.ai   │
└────────┬────────┘
         │
         │ 1. POST /mcp (no token)
         │ ← 401 + WWW-Authenticate
         │
         │ 2. GET /.well-known/oauth-protected-resource
         │ ← { authorization_servers: [...] }
         │
         │ 3. GET /.well-known/oauth-authorization-server
         │ ← { authorization_endpoint, token_endpoint, ... }
         │
         │ 4. POST /register (dynamic client registration)
         │ ← { client_id, redirect_uris }
         │
         │ 5. Opens browser → GET /authorize
         │
┌────────▼────────┐
│   User Login    │ 6. POST /authorize (username/password)
│   (Browser)     │ ← Redirect with code
└────────┬────────┘
         │
         │ 7. POST /token (exchange code for JWT)
         │ ← { access_token: "eyJ..." }
         │
         │ 8. POST /mcp + Authorization: Bearer eyJ...
         │ ← { result: {...} } ✅
         │
         ▼
┌─────────────────┐
│  MCP Resource   │
│     Server      │
│   (port 3002)   │
└─────────────────┘
```

## What's Different from Simple Version?

| Feature | Simple | MCP-Compliant |
|---------|--------|---------------|
| Tokens | Random strings | JWT (RS256) |
| PKCE | ❌ No | ✅ Yes (S256) |
| Client Registration | ❌ Hardcoded | ✅ Dynamic (RFC 7591) |
| Metadata Endpoints | ✅ Basic | ✅ Full (RFC 8414, 9728) |
| JWKS | ❌ No | ✅ Yes |
| Token Validation | Simple check | JWT signature verify |
| MCP Protocol | ❌ No | ✅ Full support |
| Real Client Support | ❌ Learning only | ✅ Works with Inspector |

## Next Steps

### ✅ Done
- MCP-compliant OAuth server
- JWT token generation
- PKCE support
- Dynamic client registration
- Full MCP protocol support

### 🎯 To Test with Claude.ai
1. Deploy to public URL (Cloudflare Tunnel)
2. Update URLs in code
3. Add to Claude.ai integrations
4. Test OAuth flow
5. Test tool execution

### 🚀 To Integrate with Desktop Commander
1. Merge auth + resource servers
2. Replace mock tools with real DC tools
3. Add user management UI
4. Add SQLite for persistence
5. Deploy with existing DC infrastructure

## Troubleshooting

**Inspector can't connect:**
- Check both servers are running
- Verify URLs are correct
- Check browser console for errors

**Login page doesn't open:**
- Clear browser cache
- Try incognito mode
- Check server logs

**Token validation fails:**
- Check JWT signature
- Verify JWKS endpoint is accessible
- Check token hasn't expired

**Tools don't execute:**
- Verify token has correct scope
- Check server logs for errors
- Ensure MCP protocol version matches

## Success! 🎉

If `./test-mcp-compliance.sh` passes all tests, you have a **working MCP OAuth server** that:

✅ Implements OAuth 2.1 correctly
✅ Uses JWT tokens
✅ Supports PKCE
✅ Has dynamic client registration
✅ Implements full MCP protocol
✅ Can work with MCP Inspector
✅ Ready for Claude.ai (with public URL)

This is the **minimal viable MCP OAuth implementation**!