# Claude Code Instructions

## Development

- **Never run full builds** (`hugo --gc --minify` or `npm run build`)
- The user runs `npm run dev` (Hugo dev server) in the background
- Hugo hot-reloads changes automatically - just edit files
- The website source is in the `website/` directory

## Structure

- `website/config/_default/` - Site configuration (params.toml, menus, etc.)
- `website/content/english/` - Content pages
- `website/assets/images/` - Images
- `website/layouts/` - Custom layout overrides

## Theme

- Theme (hugoplate) is managed via **Hugo modules** (see `website/go.mod`)
- The `website/themes/` folder is gitignored - it's downloaded automatically by Hugo
- To override theme templates, create files in `website/layouts/` (don't modify the theme directly)

## CSS and TailwindCSS

- TailwindCSS v4 uses `hugo_stats.json` for CSS purging (determines which classes to keep)
- This file is auto-generated in CI before production builds
- Locally, it's auto-updated when running `npm run dev` (Hugo dev server)
- To manually regenerate: `npm run build:stats`

## Troubleshooting

If layout breaks (CSS not loading, navigation vertical):

1. **Regenerate hugo_stats.json** (most common fix):
```bash
cd website
npm run build:stats
```

2. **Re-download modules** (if modules are corrupted):
```bash
cd website
hugo mod get -u && hugo mod npm pack && npm install
```

Then restart the dev server.
