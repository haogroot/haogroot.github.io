# Hugo Source Pages Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the repository from committed static Hugo output into a Hugo source site that accepts Markdown posts and deploys through GitHub Actions.

**Architecture:** The repository stores Hugo configuration, Markdown content, page bundles, and the `hugo-coder` theme. GitHub Actions builds `public/` at deploy time and publishes it through GitHub Pages.

**Tech Stack:** Hugo, hugo-coder theme, GitHub Actions Pages deployment.

---

### Task 1: Source Tree

**Files:**
- Create: `config.toml`
- Create: `content/posts/union_find-leetcode/index.md`
- Create: `content/posts/漫談-linked-list-在-linux-kernel-中的不一樣/index.md`
- Create: `content/about.md`
- Create: `content/projects.md`
- Create: `content/contact.md`

- [x] **Step 1: Replace generated HTML with Hugo source files**

Move post Markdown files and images into `content/posts/` page bundles and define the site metadata in `config.toml`.

### Task 2: Theme

**Files:**
- Create: `.gitmodules`
- Create: `themes/hugo-coder`

- [x] **Step 1: Add the Coder theme as a git submodule**

Run: `git submodule add https://github.com/luizdepra/hugo-coder.git themes/hugo-coder`

### Task 3: Deployment

**Files:**
- Create: `.github/workflows/pages.yml`

- [x] **Step 1: Add GitHub Pages workflow**

Build with Hugo and deploy `public/` using `actions/deploy-pages`.

### Task 4: Verification

**Files:**
- Generated: `public/`

- [x] **Step 1: Run local Hugo build**

Run: `hugo --gc --minify`

- [x] **Step 2: Run local preview**

Run: `hugo server`

- [x] **Step 3: Check key pages**

Verify homepage, `/posts/`, `/posts/union_find-leetcode/`, and `/posts/漫談-linked-list-在-linux-kernel-中的不一樣/`.
