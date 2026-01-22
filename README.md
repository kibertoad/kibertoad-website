# kibertoad-website

Personal website for Igor Savin (kibertoad).

## Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) (v0.136+)
- [Go](https://go.dev/dl/) (for Hugo modules)
- [Node.js](https://nodejs.org/) (for npm dependencies)

## Getting Started

```bash
cd website
npm install
npm run dev
```

The site will be available at http://localhost:1313/

## Theme

The theme (hugoplate) is managed via **Hugo modules** - the `website/themes/` folder is gitignored and downloaded automatically by Hugo on first run.

To update the theme:

```bash
cd website
hugo mod get -u
hugo mod npm pack
npm install
```

## Troubleshooting

If layout breaks (CSS not loading, navigation vertical), re-download modules:

```bash
cd website
hugo mod get -u && hugo mod npm pack && npm install
```

Then restart the dev server.

## Building for Production

```bash
cd website
hugo --minify
```

The built site will be in `website/public/`.

## Content Structure

Content files are in `website/content/english/`:

- `_index.md` - Homepage
- `blog/` - Blog posts
- `poetry/` - Poetry collection
- `open-source/` - Open source projects
- `games/` - Games
- `professional/` - Professional experience
- `about/` - About page
- `contact/` - Contact page

## Configuration

- `website/hugo.toml` - Main Hugo configuration
- `website/config/_default/params.toml` - Theme parameters
- `website/config/_default/menus.en.toml` - Navigation menus
- `website/data/social.json` - Social media links

## Layout Overrides

Custom layout overrides are in `website/layouts/`. These take precedence over the theme's layouts.

## Deploying to GitHub Pages

This repository includes a GitHub Actions workflow for automatic deployment.

### Setup (one-time)

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Pages**
3. Under "Build and deployment", set **Source** to "GitHub Actions"

### How it works

- Every push to `main` triggers a build and deploy
- The workflow builds the Hugo site and publishes to GitHub Pages
- Your site will be available at `https://<username>.github.io/<repo-name>/`

### Custom domain (optional)

1. Add a `CNAME` file in `website/static/` with your domain
2. Configure DNS with your domain provider
3. Update `baseURL` in `website/hugo.toml` to match your domain

## Theme Documentation

For theme customization, see the [Hugoplate documentation](https://github.com/zeon-studio/hugoplate).
