Zabbix MCP Integration for Claude Code

Give an AI coding assistant (Claude Code, or any MCP-compatible client) read-only, live access to Zabbix monitoring data — so it can answer questions about infrastructure health directly, instead of a human first opening the Zabbix UI.

Why

When working on infrastructure automation with an AI assistant, constantly switching context to check the Zabbix dashboard for "what's currently broken?" breaks flow. This integration lets Claude Code query Zabbix directly and surface problem hosts as part of the conversation.

Architecture
Claude Code  <-->  zabbix-mcp (MCP server)  <-->  Zabbix API  <-->  Zabbix instance
                                                    (read-only user: home-lab-mcp)
MCP server: zabbix-mcp — a lightweight Python MCP server that exposes the Zabbix JSON-RPC API as MCP tools.
Zabbix access: a dedicated Zabbix user (home-lab-mcp) with read-only permissions and its own API token, kept separate from any admin credentials — the assistant can query state but cannot modify monitoring configuration, acknowledge problems, or change data.
Zabbix instance: self-hosted, running as part of my homelab platform (Proxmox + K3s + Zabbix/Grafana).
Current status

🟡 Early-stage / first round of testing

Working today:

Claude Code connects to the Zabbix API through the MCP server
Can query and return currently problematic hosts
Verified the read-only permission boundary holds: a write attempt (creating a test host) was correctly rejected by the Zabbix API

Not yet done / planned:

 Creating/managing tags and host groups, and unifying host descriptions, to make the Zabbix inventory more effectively searchable (see Roadmap)
Setup

Values below are placeholders — replace with your own instance details. Never commit real tokens to a public repo.

json
{
  "mcpServers": {
    "zabbix": {
      "command": "uvx",
      "args": ["zabbix-mcp"],
      "env": {
        "ZABBIX_URL": "https://your-zabbix-instance/api_jsonrpc.php",
        "ZABBIX_TOKEN": "your-read-only-api-token"
      }
    }
  }
}
Create a dedicated Zabbix user with read-only permissions (e.g. home-lab-mcp).
Generate an API token scoped to that user.
Install zabbix-mcp and point it at your instance using the config above.
Register the MCP server with Claude Code.
Example usage

Read query — list hosts and current problems:

Prompt: "Use Zabbix MCP to list all monitored hosts and show current problems/alerts."

Claude Code queried the Zabbix API through the MCP server and returned the list of monitored hosts along with currently active problems — no manual login to the Zabbix UI required.

Write attempt — permission boundary check:

Prompt: "Try to create a new test host called 'mcp-test-host' via Zabbix MCP."

Rejected — the Zabbix API returned a permissions error, since the home-lab-mcp user only has read-only rights. This was a deliberate test of the permission boundary described below, not an accident: the assistant can report on infrastructure state, but cannot alter it.

TODO — if you have the exact error message/response text, it's worth pasting verbatim here as proof.

Why read-only, and a dedicated user

Giving an AI assistant write access to production monitoring has a much larger blast radius than the current use case (querying state) needs. A scoped, read-only API token limits what could go wrong if the token ever leaked or the assistant issued an unexpected call — the worst case is a failed read, not a changed configuration.

This isn't just a theoretical guarantee — it's been tested directly (see Example usage above): a deliberate attempt to create a host through the MCP server was rejected by the Zabbix API's own permission check.

Roadmap
 Test creating/managing tags and host groups, and unifying host descriptions — aimed at making the Zabbix inventory more effectively searchable (note: this would require write access beyond the current read-only scope, so it implies a separate, more permissive user/token for that specific workflow)
