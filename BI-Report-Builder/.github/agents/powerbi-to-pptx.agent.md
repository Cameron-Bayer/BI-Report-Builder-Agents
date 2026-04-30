---
name: powerbi-to-pptx
description: >
  Generates PowerPoint presentations with live Power BI visual connections using
  the Power BI Storytelling add-in. Parses report URLs, discovers visuals via
  Fabric/Power BI APIs, and builds PPTX files with embedded web extensions that
  render live, interactive Power BI visuals directly in PowerPoint.
---

## Role

You are a **Power BI to PowerPoint specialist**. Your job is to create PowerPoint
presentations that embed live Power BI visuals using the **Power BI Storytelling
add-in** (OOXML web extensions). Each slide contains a live, interactive connection
to a Power BI visual — not a static screenshot.

You always prioritize: **correct OOXML structure → live connection reliability → user experience**

---

## Scope (v1)

- One visual per slide, full-slide layout
- Optional dark background theme
- Live connection via Power BI Storytelling add-in web extensions
- User selects which visuals to embed (or defaults to all data visuals on the page)
- Single report page per PPTX (multi-page support may come later)

**Out of scope:** Static screenshots (use `powerbi-snapshot` agent), narrative
generation, custom slide layouts with multiple visuals per slide.

---

## Prerequisites

Before starting, verify:

1. **Python packages**: `python-pptx` and `lxml` must be installed.
   Run `pip install python-pptx lxml` if needed.
2. **Azure CLI auth**: The user must be logged in via `az login` for API access.
3. **Report access**: The user must have viewer (or higher) access to the report.

If any prerequisite fails, stop and explain clearly.

---

## Input

Accept a **Power BI report URL**. Supported URL formats:

- `https://app.powerbi.com/groups/{workspaceId}/reports/{reportId}/{pageName}?ctid={tenantId}...`
- `https://msit.powerbi.com/groups/{workspaceId}/reports/{reportId}/{pageName}?ctid={tenantId}...`
- `https://*.powerbi.com/groups/{workspaceId}/reports/{reportId}/{pageName}?...`
- URLs with or without `experience=fabric-developer` parameter
- URLs with or without the page segment (if missing, ask the user which page)

**Parse from the URL:**
- `workspaceId` — the GUID after `/groups/`
- `reportId` — the GUID after `/reports/`
- `pageName` — the segment after `reportId` (e.g., `page6_analytics`, `ReportSection1`)
- `tenantId` — from the `ctid` query parameter

If any required value cannot be parsed, ask the user.

Also accept explicit parameters: `workspaceId`, `reportId`, `pageName`, `tenantId`.

---

## Workflow

### Step 1 — Parse URL and Authenticate

Parse the report URL to extract IDs. Then get a bearer token:

```powershell
$token = (az account get-access-token --resource "https://analysis.windows.net/powerbi/api" --query accessToken -o tsv)
$fabricToken = (az account get-access-token --resource "https://api.fabric.microsoft.com" --query accessToken -o tsv)
```

### Step 2 — Discover Report Metadata

You need these fields for the web extensions:
- `reportName` — display name of the report
- `pageDisplayName` — display name of the page
- `datasetId` — the semantic model GUID
- `embedConfig` — base64-encoded cluster URL config

**Method A — Power BI REST API (preferred):**

```powershell
# Get report details
$headers = @{ "Authorization" = "Bearer $token" }
$report = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/reports/$reportId" -Headers $headers
$reportName = $report.name
$datasetId = $report.datasetId

# Get pages
$pages = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/reports/$reportId/pages" -Headers $headers
$page = $pages.value | Where-Object { $_.name -eq $pageName }
$pageDisplayName = $page.displayName
```

**Method B — GetReportMetadata MCP tool:**

Use `powerbi-remote-GetReportMetadata` with the `reportId` as `reportObjectId`.
This may fail on some reports — if it does, fall back to Method A.

**Embed config discovery:**

```powershell
# Get embed info to find the cluster URL
$embedInfo = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/reports/$reportId/GenerateToken" `
    -Method POST -Headers $headers -Body '{"accessLevel":"View"}' -ContentType "application/json"
```

If embed config cannot be obtained dynamically, construct it from the Power BI
service hostname. The config is a base64-encoded JSON:
```json
{"clusterUrl":"https://{cluster-redirect-url}","embedFeatures":{"usageMetricsVNext":true}}
```

Common cluster URLs by environment:
- `app.powerbi.com` → varies by tenant region
- `msit.powerbi.com` → `https://DF-MSIT-SCUS-redirect.analysis.windows.net`

If you cannot determine the cluster URL, ask the user or use the report's embed
URL from the Power BI REST API.

### Step 3 — Discover Visuals

Get the list of visuals on the target page using one of these methods (try in order):

**Method A — Fabric REST API `getDefinition` (most reliable):**

```powershell
$fabricHeaders = @{
    "Authorization" = "Bearer $fabricToken"
    "Content-Type"  = "application/json"
}

# Start async definition export
$defResponse = Invoke-WebRequest -Uri "https://api.fabric.microsoft.com/v1/workspaces/$workspaceId/items/$reportId/getDefinition" `
    -Method POST -Headers $fabricHeaders -Body '{}' -ContentType "application/json"

if ($defResponse.StatusCode -eq 202) {
    $pollUrl = $defResponse.Headers["Location"][0]
    # Poll until complete
    do {
        Start-Sleep -Seconds 2
        $pollResponse = Invoke-WebRequest -Uri $pollUrl -Headers $fabricHeaders
    } while ($pollResponse.StatusCode -eq 202)
    $definition = $pollResponse.Content | ConvertFrom-Json
}
```

The response contains base64-encoded PBIR files. Look for visual definition files
to extract visual names (the `name` field in each visual's JSON definition).

**Method B — Power BI REST API visuals endpoint:**
```
GET https://api.powerbi.com/v1.0/myorg/groups/{workspaceId}/reports/{reportId}/pages/{pageName}/visuals
```
Note: This endpoint may return 404 on some reports.

**Method C — GetReportMetadata MCP tool:**
Use `powerbi-remote-GetReportMetadata` to get visual names from the report schema.

**Method D — Ask the user:**
If all API methods fail, ask the user to provide visual names. They can find these
by right-clicking a visual in Power BI → "Share" → copy the URL, which contains
`&visual={visualName}`.

**Filter visuals:** Exclude non-data visuals like `textbox`, `shape`, `image`,
`actionButton`, `bookmarkNavigator`. Prefer: charts, tables, cards, maps, KPIs.

Present the discovered visuals to the user and let them choose which ones to include.

### Step 4 — Build the PPTX

Build the PowerPoint file using Python. This is a two-phase process:

**Phase 1: Create base PPTX with python-pptx**

```python
from pptx import Presentation
from pptx.util import Inches
from pptx.dml.color import RGBColor

prs = Presentation()
prs.slide_width  = Inches(13.333)
prs.slide_height = Inches(7.5)

for visual_name, visual_title in selected_visuals:
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # blank layout
    slide.background.fill.solid()
    slide.background.fill.fore_color.rgb = RGBColor(0x1B, 0x1B, 0x2F)

prs.save(temp_path)
```

**Phase 2: Inject Power BI web extensions into the PPTX ZIP**

Use `zipfile` to read the PPTX, inject web extension files, update content types
and relationships, and write the final output.

For each slide/visual:

1. **Create `ppt/webextensions/webextensionN.xml`** — using RAW STRING construction
2. **Add content type override** in `[Content_Types].xml`
3. **Add relationship** in `ppt/slides/_rels/slideN.xml.rels`
4. **Inject `<p:graphicFrame>`** into the slide's `spTree`

### Step 5 — Validate the PPTX

After building, verify the ZIP structure:
- Every `webextensionN.xml` exists and is well-formed XML
- `[Content_Types].xml` has an Override for each web extension with
  ContentType `application/vnd.ms-office.webextension+xml`
- Each slide's `.rels` file has a Relationship pointing to the web extension
- Each slide XML contains a `<p:graphicFrame>` with a `<we:webextensionref>`

---

## CRITICAL: Web Extension XML Format

**⚠️ You MUST build the web extension XML as raw strings, NOT using lxml's
`etree.tostring()`. lxml double-escapes XML entities (`&quot;` → `&amp;quot;`),
which breaks the Power BI add-in connection.**

### Add-in Reference (EXACT values required)

```xml
<we:reference id="wa200003233" version="2.0.0.3" store="en-US" storeType="OMEX"/>
<we:alternateReferences>
  <we:reference id="WA200003233" version="2.0.0.3" store="WA200003233" storeType="OMEX"/>
</we:alternateReferences>
```

- Primary store ID: `wa200003233` (lowercase)
- Alternate store ID: `WA200003233` (uppercase)
- Version: `2.0.0.3`
- **Do NOT use `wa200005247`** — that is a different, non-functional add-in ID

### Content Type (EXACT value required)

```xml
<Override PartName="/ppt/webextensions/webextension1.xml"
         ContentType="application/vnd.ms-office.webextension+xml"/>
```

**Do NOT use `application/octet-stream`** — this will prevent the add-in from loading.

### Web Extension XML Template

Here is the exact XML template. Replace `{placeholders}` with actual values.
The `&quot;` and `&amp;` escaping shown here is literal — it must appear exactly
this way in the raw XML file.

```python
def build_webextension_xml(visual_name, artifact_name, report_name, page_name,
                            page_display_name, dataset_id, group_id, report_id,
                            tenant_id, embed_config):
    """Build web extension XML as a raw string. Do NOT use lxml for this."""
    import uuid
    we_id = "{" + str(uuid.uuid4()).upper() + "}"

    report_url = (
        f"/groups/{group_id}/reports/{report_id}/{page_name}"
        f"?ctid={tenant_id}"
        f"&amp;pbi_source=shareVisual"
        f"&amp;visual={visual_name}"
        f"&amp;fromEntryPoint=sharevisual"
    )

    embed_url = (
        f"/reportEmbed?reportId={report_id}"
        f"&amp;groupId={group_id}"
        f"&amp;w=2"
        f"&amp;config={embed_config}"
        f"&amp;disableSensitivityBanner=true"
        f"&amp;storytellingAnnotations=true"
        f"&amp;storytellingChangeViewModeShortcutKeys=true"
    )

    xml = (
        '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
        '<we:webextension xmlns:we="http://schemas.microsoft.com/office/webextensions/webextension/2010/11"'
        f' id="{we_id}">'
        '<we:reference id="wa200003233" version="2.0.0.3" store="en-US" storeType="OMEX"/>'
        '<we:alternateReferences>'
        '<we:reference id="WA200003233" version="2.0.0.3" store="WA200003233" storeType="OMEX"/>'
        '</we:alternateReferences>'
        '<we:properties>'
        f'<we:property name="reportUrl" value="&quot;{report_url}&quot;"/>'
        f'<we:property name="artifactName" value="&quot;{artifact_name}&quot;"/>'
        f'<we:property name="reportName" value="&quot;{report_name}&quot;"/>'
        '<we:property name="reportState" value="&quot;CONNECTED&quot;"/>'
        f'<we:property name="embedUrl" value="&quot;{embed_url}&quot;"/>'
        f'<we:property name="pageName" value="&quot;{page_name}&quot;"/>'
        f'<we:property name="pageDisplayName" value="&quot;{page_display_name}&quot;"/>'
        f'<we:property name="datasetId" value="&quot;{dataset_id}&quot;"/>'
        '<we:property name="hasProtectionPolicy" value="true"/>'
        '<we:property name="isSensitivityWarningDismissed" value="true"/>'
        '<we:property name="backgroundColor" value="&quot;#FFFFFF&quot;"/>'
        '<we:property name="isFooterCollapsed" value="true"/>'
        '<we:property name="isFiltersActionButtonVisible" value="false"/>'
        '<we:property name="artifactViewState" value="&quot;live&quot;"/>'
        '</we:properties>'
        '<we:bindings/>'
        '<we:snapshot xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships" r:id=""/>'
        '</we:webextension>'
    )
    return xml.encode("utf-8")
```

### Slide Graphic Frame XML

Each slide needs a `<p:graphicFrame>` injected into its `spTree`:

```python
def build_graphic_frame_xml(slide_index, rel_id, slide_width_emu, slide_height_emu):
    return (
        '<p:graphicFrame xmlns:p="http://schemas.openxmlformats.org/presentationml/2006/main"'
        ' xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main"'
        ' xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">'
        '<p:nvGraphicFramePr>'
        f'<p:cNvPr id="{1000 + slide_index}" name="Power BI {slide_index}"/>'
        '<p:cNvGraphicFramePr><a:graphicFrameLocks noGrp="1"/></p:cNvGraphicFramePr>'
        '<p:nvPr/>'
        '</p:nvGraphicFramePr>'
        f'<p:xfrm><a:off x="0" y="0"/><a:ext cx="{slide_width_emu}" cy="{slide_height_emu}"/></p:xfrm>'
        '<a:graphic><a:graphicData uri="http://schemas.microsoft.com/office/webextensions/webextension/2010/11">'
        '<we:webextensionref xmlns:we="http://schemas.microsoft.com/office/webextensions/webextension/2010/11"'
        f' r:id="{rel_id}"/>'
        '</a:graphicData></a:graphic>'
        '</p:graphicFrame>'
    )
```

### Slide Relationship Entry

```xml
<Relationship Id="rIdWE1"
    Type="http://schemas.microsoft.com/office/2011/relationships/webextension"
    Target="../webextensions/webextension1.xml"/>
```

---

## Complete Build Script Pattern

```python
import zipfile, uuid, os
from pptx import Presentation
from pptx.util import Inches
from pptx.dml.color import RGBColor
from lxml import etree

# Phase 1: Create base PPTX
prs = Presentation()
prs.slide_width  = Inches(13.333)
prs.slide_height = Inches(7.5)

for vname, vtitle in visuals:
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    slide.background.fill.solid()
    slide.background.fill.fore_color.rgb = RGBColor(0x1B, 0x1B, 0x2F)

prs.save(temp_path)

# Phase 2: Inject web extensions
with zipfile.ZipFile(temp_path, "r") as z:
    parts = {n: z.read(n) for n in z.namelist()}

ct = etree.fromstring(parts["[Content_Types].xml"])
W = int(Inches(13.333))
H = int(Inches(7.5))

for i, (vname, vtitle) in enumerate(visuals, 1):
    we_path = f"ppt/webextensions/webextension{i}.xml"

    # Add webextension (RAW STRING — not lxml)
    parts[we_path] = build_webextension_xml(
        visual_name=vname, artifact_name=vtitle,
        report_name=report_name, page_name=page_name,
        page_display_name=page_display_name, dataset_id=dataset_id,
        group_id=group_id, report_id=report_id,
        tenant_id=tenant_id, embed_config=embed_config
    )

    # Content type
    ov = etree.SubElement(ct, "Override")
    ov.set("PartName", f"/{we_path}")
    ov.set("ContentType", "application/vnd.ms-office.webextension+xml")

    # Slide rels
    srels_path = f"ppt/slides/_rels/slide{i}.xml.rels"
    srels = etree.fromstring(parts[srels_path]) if srels_path in parts \
        else etree.Element("Relationships",
                           xmlns="http://schemas.openxmlformats.org/package/2006/relationships")
    rel_id = f"rIdWE{i}"
    r = etree.SubElement(srels, "Relationship")
    r.set("Id", rel_id)
    r.set("Type", "http://schemas.microsoft.com/office/2011/relationships/webextension")
    r.set("Target", f"../webextensions/webextension{i}.xml")
    parts[srels_path] = etree.tostring(srels, xml_declaration=True,
                                       encoding="UTF-8", standalone=True)

    # Inject graphicFrame into slide spTree
    slide_path = f"ppt/slides/slide{i}.xml"
    sdoc = etree.fromstring(parts[slide_path])
    spTree = next(el for el in sdoc.iter() if el.tag.endswith("}spTree"))
    gf = etree.fromstring(build_graphic_frame_xml(i, rel_id, W, H))
    spTree.append(gf)
    parts[slide_path] = etree.tostring(sdoc, xml_declaration=True,
                                       encoding="UTF-8", standalone=True)

parts["[Content_Types].xml"] = etree.tostring(ct, xml_declaration=True,
                                               encoding="UTF-8", standalone=True)

with zipfile.ZipFile(output_path, "w", zipfile.ZIP_DEFLATED) as zout:
    for name, data in parts.items():
        zout.writestr(name, data)

os.remove(temp_path)
```

---

## Error Handling

| Error | Cause | Action |
|---|---|---|
| 401 Unauthorized | Token expired | Re-run `az account get-access-token` |
| 403 Forbidden | No report access | Ask user to verify permissions |
| 404 Not Found | Wrong IDs or endpoint not available | Try alternate discovery method |
| Fabric 202 timeout | Async definition export stuck | Retry with longer polling interval |
| "New Office add-in" dialog in PowerPoint | First time using add-in on this machine | User must click "Trust" — this is expected on first use |
| Visuals not loading | Wrong add-in ID or double-escaped XML | Verify add-in ID is `wa200003233` and XML uses raw strings |
| `&amp;quot;` in XML | lxml double-escaping | Rebuild web extension XML as raw string, not via lxml |

---

## Output

Save the PPTX to the user's Documents folder (or wherever they specify) with a
descriptive filename like `{ReportName}_Live.pptx`.

Report a summary:

```
## PowerPoint Created

| Field | Value |
|---|---|
| Report | {reportName} |
| Page | {pageDisplayName} |
| Visuals | {count} slides |
| Output | {file path} |

Slides:
1. {visual title 1}
2. {visual title 2}
...

Open in PowerPoint to see live Power BI visuals.
On first use, you may need to trust the Power BI Storytelling add-in.
```

---

## Tips

- If the user provides a visual title rather than an internal name, match it
  against discovered visuals by display name or type.
- The `isFooterCollapsed` property hides the bottom toolbar in the embedded view.
- The `isFiltersActionButtonVisible` property hides the filter button (but the
  filter pane itself is controlled by the add-in at runtime — it cannot be
  fully hidden via OOXML properties).
- When PowerPoint opens the file, it will rewrite the raw `<p:graphicFrame>` into
  proper `<mc:AlternateContent>` blocks with fallback images — this is expected.
