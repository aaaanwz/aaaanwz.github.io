# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hugo-based static site blog hosted on GitHub Pages. The site is built using the hugo-PaperMod theme and contains technical blog posts in Japanese, primarily focusing on data engineering, cloud technologies, and software development.

## Development Commands

### Building the Site
```bash
# Build the site (outputs to ./public)
hugo --minify

# Build with specific base URL
hugo --minify --baseURL "https://aaaanwz.github.io/"

# Serve locally for development
hugo server
```

### Content Management
```bash
# Create a new post using the archetype
hugo new content/post/YYYY/post-name.md

# Create a new post for current year
hugo new content/post/$(date +%Y)/post-name.md
```

## Architecture

### Directory Structure
- `config.yml` - Hugo configuration file with site settings, theme configuration, and menu structure
- `content/` - All site content organized by type and year
  - `post/YYYY/` - Blog posts organized by year (2019-2025)
  - `archives.md` - Archive page
  - `portfolio.md` - Portfolio page
- `archetypes/` - Content templates for new posts
- `layouts/partials/` - Custom HTML partials (footer customization)
- `static/images/` - Static assets organized by post topic
- `themes/hugo-PaperMod/` - Hugo theme (likely a git submodule)

### Configuration Details
- **Theme**: hugo-PaperMod
- **Language**: Japanese (`ja`)
- **Base URL**: `http://aaaanwz.github.io`
- **Environment**: Production
- **Analytics**: Google Analytics enabled
- **Features**: Code copy buttons, table of contents, search functionality

### Content Structure
- Posts are organized by year in `content/post/YYYY/`
- Each post has frontmatter with title, date, draft status, and categories
- Categories include: BigQuery, data engineering topics, cloud technologies
- Images are stored in `static/images/` with subdirectories matching post topics

## Deployment

The site uses GitHub Actions for automatic deployment:
- **Workflow**: `.github/workflows/hugo.yml`
- **Hugo Version**: 0.123.3
- **Trigger**: Push to main branch or manual dispatch
- **Target**: GitHub Pages
- **Build**: Runs `hugo --minify` with production environment variables

## Working with Content

### Creating New Posts
1. Use `hugo new` command with appropriate path
2. Edit the generated file in `content/post/YYYY/`
3. Set `draft: false` when ready to publish
4. Add relevant categories in frontmatter
5. Include images in `static/images/[topic-name]/` if needed

### Post Frontmatter Format
```yaml
---
title: "Post Title"
date: YYYY-MM-DD
draft: false
categories:
- Category Name
---
```

### Theme Customization
- Footer customization is in `layouts/partials/footer.html`
- Site configuration includes social icons, navigation menu, and search settings
- Code highlighting uses monokai style with line numbers enabled