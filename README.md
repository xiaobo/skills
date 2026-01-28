# DMLC MCP Server - Agent Skills

This directory contains **Agent Skills** for Claude Code and other Claude-based AI assistants.

## What are Agent Skills?

Agent Skills are Claude-specific markdown files that provide domain expertise and procedural knowledge to AI assistants. They use **progressive disclosure** to save tokens:

1. **Metadata** (always loaded) - Brief description and when to use
2. **Full SKILL.md** (loaded on demand) - Core concepts and workflow
3. **Reference files** (loaded as needed) - Detailed syntax and examples

## Directory Structure

```
skills/
├── README.md                           # This file
└── dml-expert/                         # DML Expert Skill
    ├── SKILL.md                        # Core: DML concepts, workflow, overview
    ├── syntax/
    │   ├── column-macros.md            # Detailed column-macro syntax
    │   └── measures.md                 # Detailed measure syntax
    ├── examples/
    │   ├── common-patterns.md          # Real-world query patterns
    │   └── business-logic-rules.md     # Business domain rules
    └── troubleshooting/
        └── compilation-errors.md       # Common errors and fixes
```

## DML Expert Skill

**Purpose**: Guide AI assistants through data analysis workflows using DML/Model Query SQL.

**When to use**: 
- Working with CSV/Excel data files
- Creating or modifying DML models
- Writing Model Query SQL
- Debugging DML/Model Query SQL compilation errors

**Key capabilities**:
- Understanding the 5-step data analysis workflow
- Generating and editing DML model files
- Writing Model Query SQL with proper syntax
- Applying business logic rules for query interpretation
- Troubleshooting compilation errors

## How It Works

### For Claude Code Users

Claude Code automatically discovers skills in the `skills/` directory:

1. **Skill Discovery**: Claude scans for `SKILL.md` files at startup
2. **Metadata Loading**: Reads the purpose and "when to use" section
3. **Progressive Disclosure**: Loads full content only when relevant to the task
4. **Reference Loading**: Loads detailed syntax files only when needed

### For Other MCP Clients

If your MCP client doesn't support Agent Skills:

- Use **MCP Prompts** instead (already implemented in `DmlcMcpServer.java`)
- MCP Prompts provide similar guided workflows via `prompts/list` and `prompts/get`
- See the main README for MCP Prompts usage

## Hybrid Approach

This project uses **both** MCP Prompts and Agent Skills:

| Feature | MCP Prompts | Agent Skills (SKILL.md) |
|---------|-------------|------------------------|
| **Scope** | Protocol-level, client-agnostic | Claude-specific |
| **Trigger** | User-initiated (explicit selection) | Agent-initiated (automatic) |
| **Token Efficiency** | Loaded on-demand | Progressive disclosure |
| **Use Case** | Workflow templates | Domain expertise |
| **Portability** | Works in any MCP client | Claude Code, Claude.ai only |

**Best of both worlds**:
- **MCP Prompts** = User-facing workflow templates (for all clients)
- **Agent Skills** = Agent-level domain knowledge (for Claude)

## Content Sources

The DML Expert Skill incorporates knowledge from:

- **nl2sql project** (`../nl2sql/`)
  - Real-world DML examples from `data/models/`
  - Query patterns from `data/examples.yaml`
  - Business logic rules from `data/models/preference.md`
  - Prompt engineering patterns from `src/nl2sql/prompt.py`

- **DML documentation**
  - DML syntax and semantics
  - Model Query SQL syntax
  - Compilation error messages

- **Production experience**
  - Common pitfalls and solutions
  - Best practices for model design
  - Query optimization patterns

## Maintenance

### Updating Skills

When updating the DML Expert Skill:

1. **Edit SKILL.md** for core workflow changes
2. **Edit syntax/*.md** for syntax reference updates
3. **Edit examples/*.md** for new patterns or business rules
4. **Edit troubleshooting/*.md** for new error cases

### Adding New Skills

To add a new skill:

1. Create a new directory under `skills/`
2. Add a `SKILL.md` file with:
   - Purpose statement
   - "When to use" section
   - Core concepts
   - Links to reference files
3. Add supporting reference files in subdirectories
4. Update this README

## Testing

To test if skills are working:

1. **With Claude Code**: Open the project and ask Claude to help with a DML task
2. **Check logs**: Claude should mention loading the DML Expert Skill
3. **Verify behavior**: Claude should follow the workflow and use correct syntax

## References

- [Anthropic Agent Skills Blog Post](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [MCP Specification - Prompts](https://modelcontextprotocol.io/specification/2025-06-18/server/prompts)
- [DML Documentation](../README.md)

## License

Same as the parent project.

