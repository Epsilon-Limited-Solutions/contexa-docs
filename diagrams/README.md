# Architecture Diagrams

## How to View

### architecture.mmd (Mermaid Diagram)

This file contains the full system architecture in Mermaid format.

**Viewing Options**:

1. **GitHub** (Easiest)
   - Push this repository to GitHub
   - Navigate to `diagrams/architecture.mmd`
   - GitHub will automatically render the diagram

2. **VS Code**
   - Install extension: "Markdown Preview Mermaid Support"
   - Open `architecture.mmd`
   - Use preview pane (Cmd+Shift+V on Mac, Ctrl+Shift+V on Windows)

3. **Online Viewer**
   - Go to https://mermaid.live/
   - Copy contents of `architecture.mmd`
   - Paste into the editor
   - Diagram renders automatically

4. **Generate PNG/SVG**
   ```bash
   # Install mermaid CLI
   npm install -g @mermaid-js/mermaid-cli

   # Generate PNG
   mmdc -i architecture.mmd -o architecture.png

   # Generate SVG (scalable)
   mmdc -i architecture.mmd -o architecture.svg

   # Generate PDF
   mmdc -i architecture.mmd -o architecture.pdf
   ```

5. **JetBrains IDEs** (IntelliJ, PyCharm, WebStorm)
   - Install plugin: "Mermaid"
   - Open `architecture.mmd`
   - Preview will show automatically

---

### data-flow.md

This file contains ASCII/text-based data flow diagrams.

**Viewing**:
- Open in any text editor
- Best viewed in monospace font
- No special tools required

---

## Diagram Contents

### architecture.mmd
Shows the complete AWS architecture including:
- All AWS services and their relationships
- Data flow between components
- Network topology (VPC, subnets, security groups)
- User interactions
- Color-coded by component type:
  - Blue: User/Client components
  - Yellow: API/Gateway components
  - Purple: Compute (Lambda)
  - Green: Storage (S3, RDS)
  - Orange: ML components
  - Pink: Networking
  - Gray: Monitoring/Management

### data-flow.md
Contains 5 detailed flow diagrams:
1. Paper Ingestion Flow
2. ML Extraction Flow
3. Validation Flow
4. Knowledge Graph Generation Flow
5. Complete End-to-End Flow

Plus additional diagrams for:
- Error handling
- Monitoring points
- Key latencies table

---

## Editing Diagrams

### Mermaid Syntax Quick Reference

**Nodes**:
```mermaid
[Square Box]
(Rounded Box)
{Diamond}
[(Database)]
```

**Arrows**:
```mermaid
-->   Solid arrow
-.->  Dotted arrow
==>   Thick arrow
```

**Subgraphs** (group components):
```mermaid
subgraph "Group Name"
    A[Component A]
    B[Component B]
end
```

**Styling**:
```mermaid
classDef className fill:#color,stroke:#color
class NodeName className
```

Full documentation: https://mermaid.js.org/intro/

---

## Generating Updated Diagrams

If you make changes to the architecture:

1. Update `architecture.mmd` with new components/connections
2. Update `data-flow.md` with new flows if needed
3. Regenerate images:
   ```bash
   mmdc -i architecture.mmd -o architecture.png
   ```
4. Update documentation to reflect changes

---

## Troubleshooting

**Problem**: Mermaid diagram won't render
- **Solution**: Check syntax with https://mermaid.live/
- Common issues: Missing semicolons, unclosed quotes, invalid characters

**Problem**: Diagram is too large/complex
- **Solution**: Break into multiple diagrams (by layer or function)
- Use `%%` for comments to document complex sections

**Problem**: Need different output format
- **Solution**: Use mermaid-cli with different `-o` extension:
  - `.png` - Raster image
  - `.svg` - Vector image (scalable)
  - `.pdf` - PDF document

---

## Additional Resources

- Mermaid Live Editor: https://mermaid.live/
- Mermaid Documentation: https://mermaid.js.org/
- ASCII Diagram Tools: https://asciiflow.com/
- Draw.io (alternative): https://app.diagrams.net/
