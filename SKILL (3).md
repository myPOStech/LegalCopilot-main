# SharePoint filing map (seed -- editable by lawyers)

This is the seed. The live copy lives at `_knowledge/sharepoint-map.md` in the shared SharePoint folder. Lawyers can edit the live copy directly to change folder names, add per-matter-type overrides, or adjust filename templates -- no developer needed.

The `sharepoint-filer` skill reads this file at the start of every filing operation. Changes take effect on the next file operation.

---

## Top-level folders

These are the top-level folders inside the team's shared SharePoint root. Each matter type maps to one folder.

| Matter type | Folder name |
|---|---|
| NDA | `NDAs` |
| Contract review | `Contract Reviews` |
| Regulatory question | `Regulatory Questions` |
| Corporate change | `Corporate Changes` |
| Project | `Projects` |
| KYC support | `KYC Support` |
| GTCs | `GTCs` |
| Materials review | `Materials Reviews` |
| Claims | `Claims` |
| Inspection support | `Inspection Support` |

To rename a folder: change the name in the right column above AND rename the folder on SharePoint to match. The Copilot picks up the change at the next filing operation.

---

## Subfolder naming

```
<sanitised ticket summary, truncated to 80 chars>
```

If two tickets ever end up with the same subfolder name, the second one's folder gets the ticket key appended: `<summary> (LEGAL-XXXX)`.

---

## Filename template

```
{TICKET-KEY}_{counterparty-or-tag}_{YYYY-MM-DD}_{role-marker}.{ext}
```

| Role | Marker | Extension |
|---|---|---|
| Original draft | `v{N}` | `.docx` |
| Devil's advocate review | `v{N}_review` | `.md` |
| Attachment from Jira | `attachment_{original-stem}` | original |
| Final email (sent) | `final` | `.eml` |
| Comments thread | `comments` | `.md` |

The `_review` files always sit in the same folder as the draft they reviewed -- never in a separate review folder.

---

## Per-matter-type overrides

By default, files for a ticket are filed under `{matter-type-folder}/{ticket-summary}/`. To change this for a specific matter type, add a section below.

### Example overrides (not active by default)

```yaml
# Uncomment and edit to activate

# nda:
#   subfolder_pattern: "{counterparty}/{ticket_summary}"
#   reason: "Group all NDAs with the same counterparty under one folder"

# inspection_support:
#   subfolder_pattern: "{regulator}/{ticket_summary}"
#   reason: "Group all matters with one regulator under one folder"
```

Variables available in `subfolder_pattern`:
- `{ticket_summary}` -- the sanitised ticket summary (default)
- `{ticket_key}` -- e.g., `LEGAL-4321`
- `{counterparty}` -- if extracted from the ticket
- `{regulator}` -- if extracted (only for regulatory-related matters)
- `{jurisdiction}` -- if extracted
- `{year}` -- e.g., `2026`

---

## Active overrides

*None at present. Add overrides to the section above if the team wants matter-specific filing.*
