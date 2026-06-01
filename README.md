# Grist Widgets

Collection of **custom widgets** for [Grist](https://www.getgrist.com/), the open-source collaborative database platform.

This repo serves as a **widget catalog** hosted on GitHub Pages. Published widgets can be used directly in Grist.

---

## Published widgets

These widgets are stable and ready to use.

### TaskFlow

Suite of 3 project management widgets, usable together or separately.

| Widget | Description | Direct link |
|--------|-------------|-------------|
| **Kanban** | Task board with drag & drop, filters, modals | [Use](https://nic01asfr.github.io/Widgets-Grist/taskflow/kanban/) |
| **Gantt** | Interactive planning chart | [Use](https://nic01asfr.github.io/Widgets-Grist/taskflow/gantt/) |
| **Calendar** | Calendar view (month/week/day) | [Use](https://nic01asfr.github.io/Widgets-Grist/taskflow/calendar/) |

The 3 widgets automatically synchronize when they are in the same Grist document (shared selection).

---

## Usage

### Option 1: Direct URL

In Grist: **Add Widget** → **Custom** → **Enter Custom URL**

Paste the URL of the desired widget (see table above).

### Option 2: Widget catalog (Grist self-hosted)

Configure your Grist instance to use this repo as a widget source:

```bash
GRIST_WIDGET_LIST_URL=https://nic01asfr.github.io/Widgets-Grist/manifest.json
```

The widgets will automatically appear in the "Custom Widget" selector.

### Option 3: Fork and customization

1. Fork this repo
2. Edit the widgets in `projects/`
3. Publish to `published/`
4. Enable GitHub Pages on your fork

---

## Repo structure

```
published/         ← Production widgets (deployed on GitHub Pages)
projects/          ← Projects in development
skills/            ← Technical documentation and patterns
scripts/           ← Build and publishing tools
```

To contribute or understand the architecture: see [CLAUDE.md](CLAUDE.md)

---

## Contributing

- **Report a bug**: [Open an issue](../../issues/new)
- **Propose an improvement**: [Discussions](../../discussions)
- **Vote for a project**: Add a reaction on the project's issue

---

## Resources

- [Grist Documentation](https://support.getgrist.com/)
- [Grist Custom Widgets](https://support.getgrist.com/widget-custom/)
- [Grist Plugin API](https://support.getgrist.com/code/modules/grist_plugin_api/)

---

## License

MIT — Free to use, modify, and distribute.
