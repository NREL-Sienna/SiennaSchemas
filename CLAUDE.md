# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is the single source of truth for JSON Schema definitions (draft-07) used across the Sienna ecosystem. It contains:
- Schema definitions organized by domain (Core, Operations, Investments, Dynamics)
- OpenAPI spec files that define which schemas belong to each generated package
- Schema validation tests

## Multi-Repository Architecture

The Sienna schema ecosystem spans three repositories:

| Repository | Purpose |
|---|---|
| **SiennaSchemas** (this repo) | Schema definitions, OpenAPI specs, schema validation tests |
| **PowerOpenAPIModels** | Code generation producing Julia and Python packages |
| **PowerSystemSchemas** | Hand-written PowerSystems.jl integration code (converters, utilities) |

## Generated Package Structure

Schemas are organized to generate four separate packages (both Julia and Python):

| Package | Contents | Dependencies |
|---|---|---|
| **PowerCoreOpenAPIModels** | Shared types: curves, enums, helpers (MinMax, UpDown, CostCurve, etc.) | None |
| **PowerOperationsOpenAPIModels** | Topology, Branch, StaticInjection, Service | Core |
| **PowerInvestmentsOpenAPIModels** | Technologies, Financials, Requirements, Attributes, Regions | Core |
| **PowerDynamicsOpenAPIModels** | DynamicGeneratorComponent, DynamicInverterComponent | Core |

Note: Investments depends only on Core (not Operations). The integer ID references between schemas are semantic, not formal type dependencies.

## Schema Directory Structure

```
Core/                    # Shared types (from common.json)
Operations/              # Topology/, Branch/, StaticInjection/, Service/
Investments/             # Technologies/, Financials/, Requirements/, Attributes/, Regions/
Dynamics/                # DynamicGeneratorComponent/, DynamicInverterComponent/
```

Each domain has a corresponding OpenAPI spec file (e.g., `openapi-core.json`, `openapi-operations.json`) that lists which schemas to include in that package's generation.

## Code Generation Approach

The codegen pipeline (in PowerOpenAPIModels) follows this pattern:

1. **Generate Core first** from `openapi-core.json`
2. **Generate other packages** from their respective OpenAPI specs
3. **Post-process** to avoid type duplication:
   - Parse Core's OpenAPI spec to get the list of Core type names
   - Delete matching model files from Operations/Investments/Dynamics output
   - Remove those entries from `modelincludes.jl` (Julia) or equivalent
   - Add import statement for the Core package

This ensures Core types are defined once and imported by dependent packages.

## Schema Conventions

- All schemas use JSON Schema draft-07 (`"$schema": "http://json-schema.org/draft-07/schema#"`)
- Cross-references use relative paths (e.g., `"$ref": "../../Core/common.json#/definitions/MinMax"`)
- Discriminated unions use `oneOf` with a `discriminator` block specifying `propertyName` and `mapping`
- Component schemas typically define `id` (integer), `name` (string), `available` (boolean) as base properties

## Generator Config Files

Each OpenAPI spec has a corresponding `openapi-config-*.json` with `inlineSchemaNameMappings` to work around [openapi-generator duplicate schema issues](https://github.com/OpenAPITools/openapi-generator/issues/18948). Usage:

```bash
openapi-generator generate -c openapi-config-core.json \
  -g julia-server \
  -o ./PowerCoreOpenAPIModels.jl
```

The config files are language-agnostic; specify `-g python` or `-g julia-server` on the command line.

## Creating a Release

```bash
git tag v1.0.0
git push origin v1.0.0
```

Pushing a version tag triggers the GitHub Actions release workflow, which packages all schemas into a tarball and publishes a GitHub Release. Downstream repos (PowerOpenAPIModels) poll for new releases to trigger regeneration.
