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

**Upload to Claude chat:** Download the `SKILL.md` from the skill folder you want and attach it to your Claude conversation.

**Install as a plugin:** Install the Presage plugin in Claude Cowork or Claude Code through the plugin browser, or by uploading the `.plugin` file locally.

## Questions

Contact the Presage product or support team.
