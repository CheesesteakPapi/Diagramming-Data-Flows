# Diagrams Directory

This directory contains Mermaid.js diagrams for visualizing data flows and system architectures.

## Structure

- **examples/**: Sample diagrams demonstrating various Mermaid.js features
- **data-flows/**: Custom data flow diagrams for your projects

## Quick Start

### Creating a New Diagram

1. Create a new `.md` file in the appropriate directory
2. Add your Mermaid diagram using the following syntax:

````markdown
```mermaid
graph LR
    A[Start] --> B[End]
```
````

### Viewing Diagrams

Mermaid diagrams can be viewed in several ways:

1. **GitHub**: Diagrams render automatically when viewing `.md` files on GitHub
2. **Mermaid Live Editor**: Copy/paste your diagram code to [mermaid.live](https://mermaid.live/)
3. **VS Code**: Install the "Markdown Preview Mermaid Support" extension
4. **Command Line**: Use the Mermaid CLI tool (requires Node.js)

## Diagram Types

Mermaid supports various diagram types:

- **Flowchart**: `graph` - For data flows and process diagrams
- **Sequence Diagram**: `sequenceDiagram` - For interaction flows
- **Class Diagram**: `classDiagram` - For object relationships
- **State Diagram**: `stateDiagram` - For state machines
- **Entity Relationship**: `erDiagram` - For database schemas
- **Gantt Chart**: `gantt` - For project timelines
- **Pie Chart**: `pie` - For data distribution

## Resources

- [Mermaid Documentation](https://mermaid.js.org/)
- [Mermaid Syntax Guide](https://mermaid.js.org/intro/syntax-reference.html)
- [Mermaid Live Editor](https://mermaid.live/)
- [GitHub Mermaid Support](https://github.blog/2022-02-14-include-diagrams-markdown-files-mermaid/)
