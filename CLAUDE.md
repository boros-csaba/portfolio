# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal portfolio website built with 11ty (Eleventy) static site generator. The live site is hosted at http://www.boroscsaba.com and deployed to AWS S3 with CloudFront distribution.

## Architecture

- **Static Site Generator**: 11ty (Eleventy) v2.0.0
- **Templating**: Nunjucks (.njk files)
- **Styling**: Plain CSS 
- **JavaScript**: jQuery 3.6.0 for client-side interactions
- **Images**: Optimized using @11ty/eleventy-img with WebP/PNG formats
- **Build Output**: HTML minification enabled
- **Hosting**: AWS S3 + CloudFront + Route53 + ACM SSL

## Development Commands

```bash
npm start          # Start local development server at http://localhost:8080/
npm run build      # Build static site to ./dist directory
npm ci             # Install dependencies (used in CI/CD)
```

## Project Structure

```
src/
├── _includes/           # Nunjucks templates and partials
│   ├── base.njk        # Main layout template with meta tags, analytics
│   ├── post.njk        # Blog post layout
│   └── partials/       # Reusable template components
├── assets/
│   ├── css/            # Stylesheets
│   ├── img/            # Images and media files
│   └── js/             # JavaScript files
├── posts/              # Blog post content (.njk files)
└── index.njk           # Homepage template

dist/                   # Build output directory (gitignored)
.eleventy.js           # 11ty configuration
```

## Key Configurations

**Eleventy Config (.eleventy.js)**:
- Input directory: `src/`
- Output directory: `dist/`
- Custom image shortcode for responsive images
- HTML minification transform
- Asset passthrough copying

**Image Processing**: 
- Generates responsive images with multiple widths (400, 800, 1280px)
- Outputs WebP and PNG formats
- Lazy loading and async decoding enabled
- Images saved to `./dist/assets/img/`

## Content Management

- **Blog Posts**: Written as .njk files in `src/posts/`
- **Templates**: Nunjucks templating with layouts in `src/_includes/`
- **Assets**: Static files copied from `src/assets/` to `dist/assets/`
- **Images**: Use the custom `image` shortcode for responsive images

## Deployment

Automatic deployment via GitHub Actions on pushes to master branch:
1. Install dependencies with `npm ci`
2. Build site with `npm run build`
3. Deploy `./dist` to S3 bucket `www.boroscsaba.com`

## Analytics & Tracking

The site includes:
- Google Analytics (G-Y1654MX7N3)
- Hotjar tracking (hjid: 3416579)
- Calendly widget integration
- Cookie consent management