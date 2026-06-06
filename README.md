# haogroot.github.io

Hugo source for the GitHub Pages site.

## Writing

Add posts under `content/posts/<slug>/index.md`. Put images for a post in the same folder, usually under `images/`, and reference them with relative paths.

## Local preview

```bash
hugo server
```

## Deployment

GitHub Actions builds the site and deploys the generated `public/` artifact to GitHub Pages.
