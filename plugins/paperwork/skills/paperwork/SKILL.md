---
name: paperwork
version: "1.0.0"
description: >
  Use this skill when the user wants to create, author, or manage Paperwork document
  templates (reports, invoices, certificates, letters, or any PDF output), OR when
  they want to run or generate a document from an existing template with new data or
  field values, OR save or route a generated PDF to a local folder, SharePoint, or
  any file service.

  Trigger on phrases like: "create a template", "set up an invoice template",
  "generate my monthly report", "run the certificate template with this data",
  "create my report and save to Finance folder", "send the report to SharePoint",
  "generate a letter for...", "produce a PDF from...".

  Skip for: conceptual questions about Paperwork or Scryber internals, help with
  HTML or CSS unrelated to document generation, or questions about CLI flags where
  the user is not actively trying to author or generate a document.
---

# Paperwork Template Skill

The `paperwork` CLI generates PDF documents from reusable template configs. This skill guides two workflows:

1. **Author** — create a reusable template (HTML layout + TemplateConfigV1 JSON config)
2. **Generate** — run an existing template with data/field overrides and route the output

Detect which workflow applies from the user's intent. If the user is creating something new, use the authoring workflow. If they're producing a document from an existing template, use the generation workflow. Both can happen in a single session.

---

## Authoring a Template

### Step 1 — Understand the document

Identify from context (ask if unclear):
- **Document type**: report, invoice, certificate, letter, statement, etc.
- **Template name**: a short kebab-case identifier (e.g. `monthly-report`, `invoice`, `certificate`)
- **Fields**: scalar parameters the caller supplies at runtime (dates, names, titles, reference numbers)
- **Data sources**: JSON objects or arrays the template iterates over (line items, records, tables)

### Step 2 — Choose storage location

Ask the user where to store the template:

| Option | Path | When to use |
|---|---|---|
| Local | `./templates/<name>/` | Project-specific — travels with the repo |
| Global | `~/.paperwork/templates/<name>/` | Shared — accessible from any directory |

Create the directory before writing files.

### Step 3 — Scaffold the files

Create three files in the template folder:

**`<name>.html`** — HTML layout using handlebars expressions (see binding reference below)

**`<name>.json`** — TemplateConfigV1 config (see schema below). Use `"Source"` type with relative file paths for layouts, styles, and data files so they stay readable and editable:

```json
{
  "mainLayout": "main",
  "layouts": [
    { "name": "main", "type": "Source", "source": "./templates/<name>/<name>.html" }
  ],
  "styles": [
    { "type": "Source", "source": "./templates/<name>/<name>.css" }
  ],
  "data": [
    { "name": "<dataName>", "type": "Source", "source": "./data/<dataName>.json" }
  ],
  "fields": [
    { "id": "date", "name": "Document Date", "value": "", "type": "string", "required": true }
  ]
}
```

**`<name>.css`** — base styles (only create if the document needs it)

### Step 4 — Validate

Always validate after scaffolding:

```bash
paperwork validate ./templates/<name>/<name>.json
```

On success: `Valid. Layouts: 1, Styles: 1, Data: 1, Fields: 1`

On failure: errors are printed to stderr with the field path that failed.

### Step 5 — Test generate (optional)

Offer to do a test run with sample/placeholder data:

```bash
paperwork generate --config ./templates/<name>/<name>.json \
  --field date=2026-03-30 \
  --output /tmp/<name>-test.pdf
```

---

## Running a Template

### Step 1 — Find the template

Search in order:
1. `./templates/<name>/` (project-local)
2. `~/.paperwork/templates/<name>/` (global)

If not found, ask the user for the path.

### Step 2 — Extract overrides from context

From the user's request, identify:
- **`--data name=file`** overrides — new JSON files to substitute for named data sources
- **`--field key=value`** overrides — runtime scalar values (date, recipient name, period, etc.)

### Step 3 — Determine output destination

Detect from context:

| User says | Action |
|---|---|
| "save to ~/Documents/..." | `--output ~/Documents/Finance/report.pdf` |
| "save to [folder]" | `--output <resolved-path>/filename.pdf` |
| "send to SharePoint" / "upload to..." | generate to stdout, pipe to upload command |
| "save locally AND send to..." | write to `--output`, then separately upload the file |

For pipe/upload destinations, ask the user for the upload command if not inferable from context. Do not hard-code upload tools — use whatever the user specifies (`az`, `curl`, a custom script, etc.).

### Step 4 — Build and run the command

**Local file:**
```bash
paperwork generate --config ./templates/invoice/invoice.json \
  --data customer=./data/acme-corp.json \
  --field date=2026-03-30 \
  --field title="March Invoice" \
  --output ~/Documents/Finance/acme-march-2026.pdf
```

**Pipe to upload:**
```bash
paperwork generate --config ./templates/monthly-report/monthly-report.json \
  --data records=./data/march-records.json \
  --field month=March \
  --field year=2026 \
  | <upload-command> "Finance/Reports/March 2026.pdf"
```

**Save locally then upload:**
```bash
paperwork generate --config ./templates/monthly-report/monthly-report.json \
  --field month=March \
  --output /tmp/march-report.pdf && \
  <upload-command> /tmp/march-report.pdf "Finance/Reports/March 2026.pdf"
```

### Step 5 — Confirm output

- Local file: success is printed to stderr — `PDF written to /path/to/file.pdf (12,345 bytes)`
- Piped: check the upload command's exit code and output
- Report the result to the user

---

## CLI Quick Reference

### `generate` flags

| Flag | Description |
|---|---|
| `--config <file>` | TemplateConfigV1 JSON config (mutually exclusive with `--layout`) |
| `--layout <file>` | HTML layout file (inline mode, no config) |
| `--style <file>` | CSS file — `--layout` mode only, repeatable |
| `--data <name>=<file>` | JSON data file, repeatable. In `--config` mode, overrides matching data source by name |
| `--field <key>=<value>` | Scalar field override, repeatable |
| `--output <file>` | Output PDF path. Omit to write raw bytes to stdout |
| `--async` | Use async render path |

Exit codes: `0` = success, `1` = error (message on stderr).

### `validate` usage

```bash
paperwork validate <config>
```

Checks: valid JSON, at least one layout, every item has name + content/source.

### `info`

```bash
paperwork info
```

Prints CLI version, Scryber version, .NET runtime, and OS.

---

## TemplateConfigV1 Schema

```json
{
  "mainLayout": "main",
  "layouts":    [ ...content items... ],
  "styles":     [ ...content items... ],
  "data":       [ ...content items... ],
  "fields":     [ ...field definitions... ]
}
```

### Content item shape (layouts, styles, data)

```json
{
  "name":    "main",
  "type":    "Content",
  "content": "...",
  "source":  "https://...",
  "auth":    "",
  "format":  ""
}
```

- `type`: `"Content"` = inline value in `content`; `"Source"` = fetch from `source` URL or path
- `auth`: auth handler for remote sources (e.g. `"NoCors"`, `"Atlassian"`)
- `name` is required for layouts and data; optional for styles

### Field/parameter definition

```json
{
  "id":       "date",
  "name":     "Document Date",
  "value":    "2026-01-01",
  "type":     "string",
  "desc":     "The date printed on the document",
  "required": true
}
```

- Accessed in templates as `{{fields["date"]}}`
- Overridden at runtime with `--field date=2026-03-30`
- `required` is informational only — not enforced by the CLI

---

## Handlebars Binding Reference

All bindings go in the `data-content` attribute (not as text nodes).

```html
<!-- Field (scalar parameter) -->
<p data-content='{{fields["date"]}}'></p>

<!-- Data object property -->
<p data-content='{{customer.name}}'></p>

<!-- Nested property -->
<p data-content='{{order.address.city}}'></p>

<!-- Iterate array -->
{{#each items}}
<tr>
  <td data-content='{{this.description}}'></td>
  <td data-content='{{this.amount}}'></td>
</tr>
{{/each}}

<!-- Conditional block -->
{{#if customer.isPremium}}
<p>Premium customer discount applied.</p>
{{/if}}

<!-- Filtered iteration -->
<template data-bind="{{selectWhere(items, .type == 'taxable')}}">
  <tr>
    <td data-content='{{.description}}'></td>
    <td data-content='{{.amount}}'></td>
  </tr>
</template>

<!-- String concatenation -->
<h1 data-content='{{concat("Invoice — ", fields["month"])}}'></h1>
```

Data objects are referenced by their `name` from the `data` array. Use `fields["id"]` for scalar parameters.

---

## Agent Behaviour Notes

- **Infer intent from context** — if the user says "create my monthly report", first check if a `monthly-report` template already exists (local then global). If it does, that's a generation request, not authoring.
- **Ask before creating files** — confirm the template name and storage location before scaffolding.
- **Minimal scaffolding** — only create a CSS file if the document actually needs styling. Don't add boilerplate fields or data sources that weren't mentioned.
- **Don't hard-code destinations** — ask the user for the upload command or script path for SharePoint and file service destinations.
- **Validate before handing back** — always run `paperwork validate` after authoring. Don't skip it.
- **Future: remote templates** — the Paperwork service supports storing templates in a Firebase-backed repository. This is not yet available via the CLI but is planned. If the user mentions "Paperwork templates" as a remote source, note that remote template storage is a planned feature.
