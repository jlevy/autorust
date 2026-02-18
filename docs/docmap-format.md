# docmap.yml Format Specification

The `docmap.yml` file lives at the root of a playbook repository and serves as both a
human-readable table of contents and the machine-readable source of truth for
`autorust guide` CLI subcommands.

## Format Version

```yaml
format: 1
```

autorust checks the `format` field for compatibility.
If the format version is newer than what autorust supports, it prints a warning
suggesting the user upgrade autorust.

## Structure

```yaml
format: 1

sections:
  - name: <Section Name>           # Required. Displayed as a group header.
    description: <optional text>    # Optional. Shown below the section name.
    topics:
      - topic: <topic-id>          # Required. Becomes `autorust guide <topic-id>`.
                                   # Must be a valid CLI identifier: lowercase,
                                   # hyphens allowed, no spaces.
        description: <text>        # Required. One-line description shown in listing.
        files:                     # Required. One or more file paths relative to
          - <path/to/file.md>      # the playbook repo root.
          - <path/to/another.md>   # Multiple files are printed in order with
                                   # separators.
```

## Constraints

- **`topic` must be unique** across all sections.
  autorust validates this on load.
- **`files` paths must exist** in the playbook repo.
  autorust warns (but doesn’t fail) if a file is missing — this handles playbook
  versions where a doc hasn’t been added yet.
- **Section order is display order.** Topics within a section are displayed in the order
  listed.
- **Topic IDs are CLI identifiers.** They must match `[a-z][a-z0-9-]*` (lowercase
  alphanumeric with hyphens).
  This ensures they work as `autorust guide <topic>` subcommands without quoting.

## Example

See `repos/rust-porting-playbook/docmap.yml` for a complete example.

## How autorust Uses This File

1. `autorust guide` loads `docmap.yml` from the discovered playbook path.
2. `autorust guide` (no args) iterates sections and topics, printing a formatted listing
   with descriptions.
3. `autorust guide <topic>` looks up the topic, reads its file(s), and prints their
   content to stdout.
4. If a topic has multiple files, each is printed with a header line:
   ```
   --- reference/python-to-rust-test-coverage-playbook.md ---
   [file content]

   --- guidelines/test-coverage-for-porting.md ---
   [file content]
   ```

## Versioning

The `format` field allows future evolution:
- **format: 1** — Current.
  Flat topic list with sections, files, descriptions.
- Future versions might add: topic aliases, tags, prerequisite chains, etc.

autorust supports `format: 1` and will warn on unknown format versions.
