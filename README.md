# Panorama Read-Only MCP Server

A Model Context Protocol (MCP) server that lets Claude query a **Palo Alto Networks Panorama** management server in **read-only mode** using the PAN-OS XML API. It can retrieve system info, list managed firewalls, pull running or candidate configuration, view security policies, run operational "show" commands, retrieve logs, pull reports, and query individual firewalls through Panorama — all without making any changes.

---

## Two Versions Available

| Version | Location | Runtime | Secret storage |
|---------|----------|---------|----------------|
| **Docker** (this file) | `/` (root) | Docker Desktop + MCP Toolkit extension | `docker mcp secret set` |
| **Podman** | [`/podman`](./podman) | Podman 4.4+ (rootless, no extension needed) | `podman secret create` |

Both versions expose the same read-only tools. Choose the one that matches your container runtime.
The Podman version includes additional input validation (XPath injection protection, type whitelists) and does not expose API key generation as a tool — the key is generated out of band with `curl`.

---

## What It Does

This server exposes 20+ read-only tools to Claude via MCP:

| Tool | Description |
|------|-------------|
| `generate_api_key` | Generate a PAN-OS API key from admin credentials |
| `get_system_info` | Panorama/firewall system info (hostname, model, serial, version, uptime) |
| `get_panorama_status` | Panorama HA status and platform health |
| `list_managed_devices` | List all managed firewalls with details |
| `get_device_groups` | List device groups and their assigned firewalls |
| `get_templates` | List templates and template stacks |
| `get_running_config` | Retrieve active (running) config for any XPath |
| `get_candidate_config` | Retrieve candidate (uncommitted) config for any XPath |
| `get_security_rules` | Security policy rules from device groups or firewalls |
| `get_nat_rules` | NAT rules from device groups or firewalls |
| `get_address_objects` | Address objects (shared, device-group, or firewall) |
| `get_address_groups` | Address group objects |
| `get_service_objects` | Service objects (protocol/port definitions) |
| `get_security_profiles` | Security profiles (AV, anti-spyware, vulnerability, URL filtering, etc.) |
| `run_show_command` | Run any read-only "show" operational command |
| `get_logs` | Retrieve logs (traffic, threat, system, config, URL, WildFire, etc.) |
| `get_report` | Retrieve predefined, dynamic, or custom reports |
| `get_predefined_objects` | Retrieve predefined applications, services, or threats |
| `get_ha_status` | High-availability status |
| `get_job_status` | Check async job status |
| `export_device_state` | Export running config for backup/review |
| `get_config_audit` | Show uncommitted changes |
| `get_commit_locks` | Show active commit locks |
| `get_version_info` | PAN-OS version, serial, model |

---

## Prerequisites

- **Docker Desktop** with the [MCP Toolkit extension](https://hub.docker.com/extensions/docker/labs-ai-tools-for-devs) installed
- **Palo Alto Networks Panorama** (PAN-OS 11.1+)
- A Panorama admin account with a **read-only role** (recommended: "Superuser (readonly)" or "Device admin (readonly)")
- A pre-generated **API key** from Panorama

---

## Step-by-Step Setup

### Step 0 — Generate Your Panorama API Key

Run this command to generate an API key (replace the host, username, and password):

```bash
curl -k -X POST 'https://<panorama-host>/api/?type=keygen' \
  -d 'user=<admin-username>&password=<admin-password>'
```

You'll get an XML response like:

```xml
<response status="success">
  <result>
    <key>LUFRPT1xxxxxxxxxxxxxxxxxxxxxxxxxx==</key>
  </result>
</response>
```

Copy the `<key>` value — you'll need it in Step 3.

> **Tip:** Use an admin account with a readonly role. This provides defense-in-depth: even if a write call slipped through, Panorama would reject it.

### Step 1 — Save the Project Files

```bash
mkdir panorama-readonly-mcp-server && cd panorama-readonly-mcp-server
```

Save all files from this project into that folder:
- `panorama_readonly_server.py`
- `Dockerfile`
- `.dockerignore`
- `requirements.txt`
- `.env.example`
- `custom-catalog.yaml`

### Step 2 — Build the Docker Image

```bash
docker build -t panorama-readonly-mcp-server .
```

### Step 3 — Set Secrets

```bash
docker mcp secret set PANORAMA_HOST="panorama.example.com"
docker mcp secret set PANORAMA_API_KEY="LUFRPT1xxxxxxxxxxxxxxxxxxxxxxxxxx=="
docker mcp secret set PANORAMA_VERIFY_SSL="no"
```

> Set `PANORAMA_VERIFY_SSL` to `"no"` if your Panorama uses a self-signed TLS certificate (common in lab/dev environments). Set to `"yes"` for production with proper certificates.

Verify your secrets are stored:

```bash
docker mcp secret list
```

### Step 4 — Create the Custom Catalog

Copy the `custom-catalog.yaml` file to:

```bash
mkdir -p ~/.docker/mcp/catalogs
cp custom-catalog.yaml ~/.docker/mcp/catalogs/custom.yaml
```

### Step 5 — Update the Registry

Add an entry in `~/.docker/mcp/registry.yaml`:

```yaml
panorama-readonly:
  catalog: custom
  enabled: true
```

### Step 6 — Configure Claude Desktop

Edit your Claude Desktop configuration file:

| Platform | Path |
|----------|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

Add the MCP server entry:

```json
{
  "mcpServers": {
    "panorama-readonly": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-e", "PANORAMA_HOST",
        "-e", "PANORAMA_API_KEY",
        "-e", "PANORAMA_VERIFY_SSL",
        "panorama-readonly-mcp-server"
      ],
      "env": {
        "PANORAMA_HOST": "panorama.example.com",
        "PANORAMA_API_KEY": "your-api-key-here",
        "PANORAMA_VERIFY_SSL": "no"
      }
    }
  }
}
```

### Step 7 — Restart Claude Desktop

Quit and reopen Claude Desktop. The Panorama Read-Only server should appear in the MCP tools list.

### Step 8 — Verify

```bash
docker mcp server list
```

You should see `panorama-readonly` in the output.

---

## Usage Examples

Once connected, try these natural-language prompts in Claude:

- **"Show me all firewalls managed by Panorama"**
- **"What security rules are in the 'branch-offices' device group?"**
- **"Pull the last 50 threat logs from the past 24 hours"**
- **"Show me the running config for the firewall with serial 0123456789"**
- **"What's the HA status of Panorama?"**
- **"List all address objects in the shared location"**
- **"Get the top-applications report for the last 7 days"**
- **"Show me the system info for firewall serial ABC123"**
- **"Are there any uncommitted changes on Panorama?"**
- **"What device groups are configured and which firewalls are in each?"**
- **"Show me the NAT rules in the 'datacenter' device group"**
- **"Run `show interface all` on firewall serial XYZ789"**
- **"What commit locks are active right now?"**

---

## Security Design

This server enforces read-only access at **three layers**:

### 1. Application Layer (Code Validation)
- The `run_show_command` tool validates that commands start with `<show>` and rejects blocked prefixes (`<request>`, `<set>`, `<delete>`, `<debug>`, `<load>`, `<save>`, `<revert>`, `<schedule>`, `<test>`, `<clear>`, `<edit>`, `<configure>`, `<import>`, `<clone>`).
- No tool constructs `action=set`, `action=edit`, `action=delete`, `type=commit`, `type=import`, or `type=user-id` API calls.

### 2. API Call Layer (XML API Only)
- All requests go to `https://<host>/api/` (the XML API endpoint).
- The REST API (`/restapi/`) is never used.
- Only allowed API call patterns: `action=show`, `action=get`, `type=op` with `<show>` commands, `type=log`, `type=report`, `type=export` (config only), and `type=version`.

### 3. Panorama RBAC Layer (Defense-in-Depth)
- The admin account should use a "Superuser (readonly)" or "Device admin (readonly)" role.
- Even if a write call somehow slipped through, Panorama rejects it with error 15 (Operation denied) or 16 (Unauthorized).

### Additional Security Notes
- **Rotate your API key** periodically. Set an API key lifetime on Panorama.
- **Never store the API key in plain text** — use Docker secrets or environment variables.
- The server runs as a **non-root user** (UID 1000) inside the Docker container.
- All logging goes to **stderr**, keeping stdout clean for the JSON-RPC protocol.

---

## Troubleshooting

### "Could not connect to Panorama"
- Verify `PANORAMA_HOST` is correct and reachable from your Docker container.
- Check if Panorama's management interface is accessible on HTTPS (port 443).
- If using Docker Desktop, ensure DNS resolution works inside the container.

### "HTTP 403" or "Unauthorized"
- Your API key may be expired or invalid. Regenerate it with `type=keygen`.
- The admin account may not have API access enabled. Check Device > Admin Roles > XML API tab on Panorama.

### "SSL certificate verify failed"
- Set `PANORAMA_VERIFY_SSL=no` if Panorama uses a self-signed certificate.
- Or replace Panorama's self-signed cert with one from a trusted CA.

### "Bad XPath" errors
- Double-check your XPath syntax. Use the Panorama API browser at `https://<panorama>/api/` to explore valid paths.
- Device group names and object names are case-sensitive.

### "Job did not complete within timeout"
- Log and report queries on large datasets can take time. The default timeout is 120 seconds.
- Try narrowing your query with a more specific filter or shorter time range.

### Server doesn't appear in Claude Desktop
- Verify the Docker image built successfully: `docker images | grep panorama`
- Check the Claude Desktop config JSON syntax is valid.
- Restart Claude Desktop after making config changes.

---

## How to Add New Read-Only Tools

1. Add a new function in `panorama_readonly_server.py` following the pattern:

```python
@mcp.tool()
async def my_new_tool(param: str, target_serial: str = "") -> str:
    """Single-line description of what this tool does."""
    if not param.strip():
        return "❌ Error: param is required"
    try:
        root = await _panorama_request(
            {"type": "op", "cmd": "<show><my><command></command></my></show>"},
            target_serial,
        )
        result = root.find(".//result")
        if result is None:
            result = root
        return f"✅ Result:\n{_xml_to_text(result)}"
    except Exception as e:
        logger.error(f"Error in my_new_tool: {e}")
        return f"❌ Error: {str(e)}"
```

2. Rebuild the Docker image: `docker build -t panorama-readonly-mcp-server .`
3. Restart Claude Desktop.

**Rules for new tools:**
- Only use `action=show`, `action=get`, `type=op` with `<show>` commands, `type=log`, `type=report`, `type=export`, or `type=version`.
- Never use `action=set/edit/delete/rename/clone/move/override`, `type=commit`, `type=import`, or `type=user-id`.
- Single-line docstrings only.
- Default optional string params to `""`, never `None`.
- Always return strings.

---

## Architecture

```
Claude Desktop  ←→  JSON-RPC (stdio)  ←→  MCP Server (Docker)  ←→  HTTPS  ←→  Panorama XML API
```

The server runs inside a Docker container, communicates with Claude Desktop over stdin/stdout using the JSON-RPC protocol, and makes HTTPS POST requests to the Panorama XML API at `https://<host>/api/`.

---

## License

This project is provided as-is for integrating Palo Alto Networks Panorama with Claude Desktop via MCP. Use at your own risk. Not affiliated with or endorsed by Palo Alto Networks.
