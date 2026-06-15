# Presage MCP Plugin

Claude skills for the Presage LIMS platform — lab data, work orders, samples, analyses, and compliance workflows.

## Available Skills

| Skill | What it does |
|-------|-------------|
| `presage-api-navigation` | General entry point for working with Presage LIMS via the MCP server — authentication, schema navigation, querying work orders, samples, products, and analyses |
| `presage-schema-cheatsheet` | Quick-reference for common Presage GraphQL types, filter structures, enum values, and field naming — read this before calling `search_schema` for frequent types |
| `presage-analysis-finder` | Finds which plants run a specific test or analysis across the network; also searches by brand, product SKU, or category |
| `presage-blank-thresholds` | Audits analyses with missing or unconfigured spec limits (thresholds); compares threshold coverage across plants |
| `presage-out-of-spec` | Finds, counts, and ranks out-of-spec, failed, warning, or at-risk test results; compares plants by quality performance |
| `presage-product-testing` | Retrieves test results for a specific product brand or SKU; finds which plants carry or test a given product |
| `presage-qc-workflow` | Surfaces open work orders, pending validations, unclosed samples, and incomplete calibrations or verifications |

## How to Use

**Install as a plugin (recommended):** Add this marketplace and enable the
`presage-mcp` plugin. Both paths work on the free plan — see the
[repository README](../README.md#install) for step-by-step instructions:

- **claude.ai (web or Desktop):** Customize → Plugins → **+** → Create plugin → Add
  marketplace → `https://github.com/ISoft-Data-Systems/isoft-mcp-plugins`, then enable
  `presage-mcp`.
- **Claude Code (CLI):** `/plugin marketplace add ISoft-Data-Systems/isoft-mcp-plugins` then
  `/plugin install presage-mcp@ISoft`.

**Upload a single skill:** In claude.ai, go to Customize → Skills and upload either a
skill's `SKILL.md` (from its folder under [skills/](skills/)) or a `.zip` of the skill
folder (use the zip when the skill has a `references/` folder).

**Attach to one chat:** Download a skill's `SKILL.md` and attach it as a file in your
conversation. This applies only to that chat and does not include the skill's
`references/` files.

## Questions

Contact the Presage product or support team.
