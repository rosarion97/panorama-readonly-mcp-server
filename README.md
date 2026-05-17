# Panorama Read-Only MCP Server

A Model Context Protocol (MCP) server that lets Claude query a **Palo Alto Networks Panorama** management server in **read-only mode** using the PAN-OS XML API. It can retrieve system info, list managed firewalls, pull running or candidate configuration, view security policies, run operational "show" commands, retrieve logs, pull reports, and query individual firewalls through Panorama — all without making any changes.

This server is designed to run inside a Docker container managed by the **Docker MCP Toolkit**. Secrets stay in Docker — they are never written into Claude Desktop's configuration file.

> Not affiliated with or endorsed by Palo Alto Networks. Use at your own risk.

---

## What It Does

This server exposes 23 read-only tools to Claude via MCP:

| Tool | Description |
|------|-------------|
| `get_system_info` | Panorama/firewall system info (hostname, model, serial, version, uptime) |
| `get_panorama_status` | Panorama HA status and platform health |
| `list_managed_devices` | List all managed firewalls with details |
| `get_device_groups` | List device groups and their assigned firewalls |
| `get_templates` | List templates and template stacks |
| `get_running_config` | Retrieve active (running) config for any XPath under `/config` |
| `get_candidate_config` | Retrieve candidate (uncommitted) config for any XPath under `/config` |
| `get_security_rules` | Security policy rules from device groups or firewalls |
| `get_nat_rules` | NAT rules from device groups or firewalls |
| `get_address_objects` | Address objects (shared, device-group, or firewall) |
| `get_address_groups` | Address group objects |
| `get_service_objects` | Service objects (protocol/port definitions) |
| `get_security_profiles` | Security profiles (AV, anti-spyware, vulnerability, URL filtering, etc.) |
| `run_show_command` | Run any read-only `<show>` operational command |
| `get_logs` | Retrieve logs (traffic, threat, system, config, URL, WildFire, etc.) |
| `get_report` | Retrieve predefined, dynamic, or custom reports |
| `get_predefined_objects` | Retrieve predefined applications, services, or threats |
| `get_ha_status` | High-availability status |
| `get_job_status` | Check async job status |
| `export_device_state` | Export running config for backup/review |
| `get_config_audit` | Show uncommitted changes |
| `get_commit_locks` | Show active commit locks |
| `get_version_info` | PAN-OS version, serial, model |

API-key generation is intentionally **not** exposed as a tool. Generating an API key requires admin credentials, and routing those through an LLM would put them in conversation context. Instead, generate the key once, out of band, using `curl` (see Step 0).

---

## Prerequisites

- **Docker Desktop** with the [MCP Toolkit](https://docs.docker.com/desktop/features/mcp/) extension installed and enabled
- **Palo Alto Networks Panorama** (PAN-OS 11.1 or newer)
- A Panorama admin account with a **read-only role** scoped as narrowly as your environment allows (see [Recommended Panorama Role](#recommended-panorama-role))
- A pre-generated **PAN-OS API key**

---

## Recommended Panorama Role

The server enforces read-only access at the application layer (only `<show>` operational commands and `action=show`/`action=get` config calls). However, an admin role with broad read access can still expose sensitive material such as **administrator password hashes** (`/config/mgt-config/users//phash`), **certificate private keys** (`/config/shared/certificate//private-key`), and shared secrets for RADIUS/TACACS/SNMP. The LLM can construct XPaths that target those nodes if RBAC permits it.

To minimize exposure:

1. Create a **custom Admin Role** under *Device > Admin Roles* (do not use the built-in "Superuser (readonly)").
2. On the **WebUI / XML API** tab, grant only:
   - XML API: Configuration (read), Operational Requests, Logs, Reports, Export
   - WebUI: read access scoped to the device groups, templates, objects, and policies you want Claude to see
3. Disable XML API access for: Commit, User-ID Agent.
4. Under *Configuration*, deny visibility into Mgt Config (admin users), Certificate Management, and any authentication/server profile nodes that contain shared secrets.
5. Set a finite **API key lifetime** under *Device > Setup > Management > Authentication Settings*.

---

## Step-by-Step Setup

### Step 0 — Generate Your Panorama API Key (out of band)

Run this from a trusted machine on a trusted network. Do **not** disable TLS verification when the admin password is on the wire.

```bash
curl -X POST 'https://<panorama-host>/api/?type=keygen' \
  --data-urlencode 'user=<admin-username>' \
  --data-urlencode 'password=<admin-password>'
```

If your Panorama uses a self-signed certificate, do this once instead of using `-k`:

```bash
# Save Panorama's cert to a local file (one-time)
echo | openssl s_client -connect <panorama-host>:443 -servername <panorama-host> 2>/dev/null \
  | openssl x509 > /tmp/panorama.pem

# Then call keygen with the cert pinned
curl --cacert /tmp/panorama.pem -X POST 'https://<panorama-host>/api/?type=keygen' \
  --data-urlencode 'user=<admin-username>' \
  --data-urlencode 'password=<admin-password>'
```

You'll get a response like:

```xml
<response status="success">
  <result>
    <key>LUFRPT1xxxxxxxxxxxxxxxxxxxxxxxxxx==</key>
  </result>
</response>
```

Copy the `<key>` value — you'll need it in Step 3. Do not paste this key into chat with Claude.

### Step 1 — Get the Project Files

Clone or download this repository so the following files are present in your working directory:

- `panorama_readonly_server.py`
- `Dockerfile`
- `.dockerignore`
- `requirements.txt`
- `custom-catalog.yaml`
- `.env.example` (reference only — not used by the container; secrets live in Docker)

### Step 2 — Build the Docker Image

```bash
docker build -t panorama-readonly-mcp-server .
```

### Step 3 — Store Secrets in Docker (not in Claude Desktop)

```bash
docker mcp secret set PANORAMA_HOST="panorama.example.com"
docker mcp secret set PANORAMA_API_KEY="LUFRPT1xxxxxxxxxxxxxxxxxxxxxxxxxx=="
docker mcp secret set PANORAMA_VERIFY_SSL="yes"
```

Use `PANORAMA_VERIFY_SSL="yes"` whenever you can. Only set it to `"no"` if Panorama uses a self-signed certificate and you accept the risk; the safer alternative is to mount Panorama's CA cert into the container and keep verification on.

Verify the secrets are stored:

```bash
docker mcp secret list
```

You should see `PANORAMA_HOST`, `PANORAMA_API_KEY`, and `PANORAMA_VERIFY_SSL` listed (values are not displayed).

### Step 4 — Install the Custom Catalog

```bash
mkdir -p ~/.docker/mcp/catalogs
cp custom-catalog.yaml ~/.docker/mcp/catalogs/custom.yaml
```

### Step 5 — Enable the Server in the Registry

`~/.docker/mcp/registry.yaml` lists which servers from your catalogs are active. The file has a single top-level `registry:` key. Add the `panorama-readonly` entry under it — **do not overwrite the file** if it already exists.

Final shape of the file:

```yaml
registry:
  panorama-readonly:
    catalog: custom
    enabled: true
  # ... any other servers you already had stay here
```

If `registry.yaml` does not exist yet, create it with exactly the snippet above.

### Step 6 — Connect Claude Desktop to the Docker MCP Gateway

Connect Claude Desktop to the Docker MCP gateway so it picks up servers from your registry. Either:

- In Docker Desktop: open **MCP Toolkit > Clients** and connect Claude Desktop, **or**
- From the CLI:
  ```bash
  docker mcp client connect claude-desktop
  ```

This wires Claude Desktop into the gateway. Claude Desktop's `claude_desktop_config.json` will reference the gateway only — it does **not** contain `PANORAMA_API_KEY` or any other secret.

Quit and reopen Claude Desktop after connecting.

### Step 7 — Verify

```bash
docker mcp server list
docker mcp tools list
```

You should see `panorama-readonly` listed as enabled and its tools in the second command's output. In Claude Desktop, the tools menu should now include the Panorama tools.

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

### 1. Application Layer (code validation)

- `run_show_command` requires the command to start with `<show>` and rejects any payload that contains a blocked prefix: `<request>`, `<set>`, `<delete>`, `<debug>`, `<load>`, `<save>`, `<revert>`, `<schedule>`, `<test>`, `<clear>`, `<edit>`, `<configure>`, `<import>`, `<clone>`.
- `get_running_config` and `get_candidate_config` require the XPath to start with `/config` and limit length.
- All other config tools build their XPath from a fixed template; the only variable parts (device-group names, profile types, predefined object types) are validated against a whitelist of safe characters or against an enum of allowed values, blocking attribute-quote breakouts.
- No tool ever constructs `action=set`, `action=edit`, `action=delete`, `type=commit`, `type=import`, or `type=user-id` API calls.

### 2. API Call Layer (XML API only)

- All requests go to `https://<host>/api/` (the XML API endpoint).
- The REST API (`/restapi/`) is never used.
- Only allowed call patterns: `action=show`, `action=get`, `type=op` with `<show>` commands, `type=log`, `type=report`, `type=export` (config only), and `type=version`.

### 3. Panorama RBAC Layer (defense in depth)

- The admin role used for the API key should be a custom read-only role scoped per the [Recommended Panorama Role](#recommended-panorama-role) section.
- Even if a write call somehow slipped through, Panorama rejects it with error 15 (Operation denied) or 16 (Unauthorized).

### Additional security notes

- **Read-only is not the same as harmless.** A read-only role with broad config visibility can still leak password hashes, certificate private keys, and shared secrets. Use a custom role that hides those nodes.
- **Rotate the API key** periodically and set an API key lifetime on Panorama.
- **Never store the API key in Claude Desktop's config file.** All credential storage goes through `docker mcp secret set`.
- The server runs as a **non-root user** (UID 1000) inside the Docker container.
- All logging goes to **stderr**, keeping stdout clean for the JSON-RPC protocol. Error messages returned to the LLM do not echo raw response bodies on authentication failures.

---

## Troubleshooting

### "Could not connect to Panorama"

- Verify `PANORAMA_HOST` is correct and reachable from your Docker container.
- Check that Panorama's management interface is accessible on HTTPS (port 443).
- If using Docker Desktop, confirm DNS resolution works inside the container.

### "HTTP 401" or "HTTP 403"

- The API key may be expired or invalid. Regenerate it (Step 0) and update the secret with `docker mcp secret set PANORAMA_API_KEY=...`.
- The admin account may not have XML API access enabled. Check *Device > Admin Roles > XML API* on Panorama.

### "Failed to parse XML response from Panorama"

- This usually means Panorama returned non-XML (e.g., a captive portal or proxy interstitial). Confirm `PANORAMA_HOST` resolves to Panorama directly.

### "SSL certificate verify failed"

- For production, install a CA-signed cert on Panorama or mount Panorama's CA into the container so verification can stay on.
- For lab use only, set `PANORAMA_VERIFY_SSL=no`.

### "Bad XPath" errors

- Double-check the XPath syntax. The Panorama API browser at `https://<panorama>/api/` (logged in as your admin) is the easiest way to find valid paths.
- Device group names and object names are case-sensitive.
- The server rejects XPaths that don't start with `/config` and any name with characters outside `[A-Za-z0-9_.\- ]`.

### "Job did not complete within timeout"

- Log and report queries on large datasets can take time. The default timeout is 120 seconds.
- Narrow your query with a more specific filter or a shorter time range.

### Server doesn't appear in Claude Desktop

- Verify the image built successfully: `docker images | grep panorama`.
- Confirm the registry entry: `docker mcp server list` should show `panorama-readonly` enabled.
- Confirm Claude Desktop is connected to the gateway (Step 6).
- Restart Claude Desktop after any change to the registry, secrets, or catalog.

---

## How to Add New Read-Only Tools

1. Add a new function in `panorama_readonly_server.py` following the pattern:

```python
@mcp.tool()
async def my_new_tool(param: str, target_serial: str = "") -> str:
    """Single-line description of what this tool does."""
    try:
        name = _validate_name(param, "param")
        root = await _panorama_request(
            {"type": "op", "cmd": f"<show><my><thing>{name}</thing></my></show>"},
            target_serial,
        )
        result = root.find(".//result") or root
        return f"Result:\n{_xml_to_text(result)}"
    except Exception as e:
        logger.error(f"Error in my_new_tool: {e}")
        return f"Error: {str(e)}"
```

2. Rebuild the Docker image: `docker build -t panorama-readonly-mcp-server .`
3. Restart Claude Desktop.

**Rules for new tools:**

- Only use `action=show`, `action=get`, `type=op` with `<show>` commands, `type=log`, `type=report`, `type=export`, or `type=version`.
- Never use `action=set/edit/delete/rename/clone/move/override`, `type=commit`, `type=import`, or `type=user-id`.
- Run any user-supplied value that ends up inside an XPath through `_validate_name()` or an enum check before interpolating.
- Single-line docstrings only.
- Default optional string params to `""`, never `None`.
- Always return strings.

---

## Architecture

```
Claude Desktop  ←→  Docker MCP Gateway  ←→  panorama-readonly container  ←→  HTTPS  ←→  Panorama XML API
                                              │
                                              └─ reads PANORAMA_HOST / PANORAMA_API_KEY / PANORAMA_VERIFY_SSL
                                                 from Docker-managed secrets at startup
```

Claude Desktop connects to the Docker MCP gateway. The gateway launches the `panorama-readonly-mcp-server` container, injecting your stored secrets as environment variables. The container speaks JSON-RPC over stdio with the gateway, and HTTPS to the Panorama XML API at `https://<host>/api/`. Secrets never appear in Claude Desktop's config file.

---

## License

This project is provided as-is for integrating Palo Alto Networks Panorama with Claude Desktop via MCP. Use at your own risk. Not affiliated with or endorsed by Palo Alto Networks.
