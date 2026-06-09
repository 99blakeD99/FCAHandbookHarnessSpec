# Deployment & Integration Patterns

Once you have implemented the Harness (following HowToImplement.md), choose the integration method below that fits your architecture.

## Choice of Integration Methods

### Named Tool for an LLM

**What it is**: Register the harness as a named tool that an LLM (Claude, ChatGPT, etc.) can invoke.

**When to use**: You're building an LLM application and want the model to call compliance analysis as needed.

**How**:
- Implement the harness in Python (or your runtime)
- Expose it as an HTTP endpoint or function
- Register in your LLM SDK tool definitions using the Tool Specification below
- The LLM invokes the tool when it detects a compliance question

**Tool Specification**:

```
Name: Compliance Enquiry

Description: Use this tool to answer questions about regulatory compliance. 
The tool retrieves relevant Handbook entries via semantic search, reasons 
over them, and provides citations with exact text. Invoke when users ask: 
"Which requirements apply to [product/service]?" or "Is [situation] compliant?". 
Do NOT invoke for out-of-scope questions or general advice.
```

---

### Model Context Protocol (MCP)

**What it is**: Expose the harness as an MCP server so Claude Desktop and other MCP clients can use it.

**When to use**: You want Claude (or other Claude users in your organization) to access the harness without building a custom wrapper.

**How**:
- Implement the harness in Python
- Wrap it as an MCP server (using the claude-python-sdk MCP utilities)
- Configure in `claude_desktop_config.json` with server path and environment variables
- Claude Desktop and other MCP clients can now invoke it

**Reference**: See [Anthropic MCP Documentation](https://modelcontextprotocol.io)

---

### Standalone Agent

**What it is**: Run the harness as a persistent agent that accepts queries and returns compliance analysis.

**When to use**: You want a dedicated compliance agent that manages its own state, handles complex multi-turn reasoning, or integrates deeply with your compliance workflow.

**How**:
- Implement the harness with agent orchestration (e.g., ReAct pattern)
- Add state management for conversation context
- Run as a service or scheduled task
- Accept queries (API, CLI, message queue) and return structured compliance analysis

---

### Plugin/Extension

**What it is**: Integrate the harness as a plugin into an existing platform (Slack, Teams, Jira, your internal tools, etc.).

**When to use**: Your team already uses a platform and wants compliance queries available there.

**How**:
- Implement the harness core in Python
- Build a plugin/bot for your platform (Slack Bot SDK, Teams connector, Jira app, etc.)
- Accept user queries in the platform interface
- Return formatted compliance analysis in the platform
