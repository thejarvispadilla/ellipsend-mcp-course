# Ellipsend Integration Notes

*Version: pending rewrite*

This file contains the things your AI tool needs to know that aren't in the MCP schema:

- Webhook subscription endpoint and secret setup
- Standard workflow: inbound DM handling
- Standard workflow: comment-to-DM handling
- Meta platform quirks (24h messaging window, 7-day CTD window, self-echo, rate limits)
- Common gotchas on macOS (launchd TLS, dotenv override)

## Why this file exists

The MCP schema is self-describing. When your AI tool connects to `https://mcp.ellipsend.com/mcp/` it already sees every tool's name, description, input schema, and required fields. This file is not a mirror of that. It's the workflow context and platform quirks the schema can't communicate on its own.

## Status

Being rewritten based on debrief feedback from the Carl reference implementation (April 2026). Will be published here when ready.
