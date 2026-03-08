---
title: "The Zero-Cost Blogging Stack: Hugo, GitHub Pages, and AI from Your Phone"
date: 2026-03-08
draft: true
ai: true
tags: ["Hugo", "Blogging", "AI", "GitHub Pages", "Payload CMS"]
description: "A deeper look at the free tools behind this blog: from Payload CMS on Vercel + Neon, to Hugo on GitHub Pages — and how Claude on mobile changed the game."
---

In my [previous post](/posts/from-payload-cms-to-hugo/), I talked about why I moved from Payload CMS to Hugo. But I glossed over some important details: the infrastructure, the costs (or lack thereof), and a workflow change I didn't see coming — writing blog posts from my phone.

Let me expand on all of that.

## Payload CMS was free too — and that's worth saying

I don't want to give Payload a bad reputation. My previous setup was entirely free:

- **[Vercel](https://vercel.com/)** hosted the Payload application on its generous free tier. For a personal blog with moderate traffic, you'll never hit the limits.
- **[Neon](https://neon.tech/)** provided a serverless PostgreSQL database, also on a free tier. It spins down when idle and wakes up on demand — perfect for a low-traffic site.

The combination worked remarkably well. Payload gives you a full CMS with an admin panel, content modeling, authentication, media management — all backed by a proper database. And with Vercel + Neon, the whole stack costs exactly zero.

If you need a CMS with structured content, user roles, or dynamic features, Payload on Vercel + Neon is still an excellent choice. For my use case — a developer writing occasional blog posts — it was simply more than I needed.

## Hugo + GitHub Pages: a different kind of free

With Hugo, the entire hosting story becomes even simpler. There's no server, no database, no runtime. The site is just static HTML files, and GitHub Pages serves them for free.

Here's how the deploy works:

1. I push a commit to the `main` branch of my repository.
2. A [GitHub Actions workflow](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site) kicks in automatically.
3. The workflow installs Hugo, checks out the repo (including the theme submodule), and runs `hugo --gc --minify` to build the site.
4. The generated `public/` folder is uploaded as an artifact and deployed to GitHub Pages.

That's it. No build server to configure, no CDN to set up, no deployment platform to sign up for. GitHub handles everything — the CI/CD, the hosting, the SSL certificate, even the custom domain.

The workflow file is about 60 lines of YAML, and I haven't touched it since I set it up. Every push to `main` triggers a new deploy, and the site is live within a couple of minutes.

### What about costs?

GitHub Pages is free for public repositories. GitHub Actions gives you 2,000 minutes per month on the free tier. A Hugo build takes about 10 seconds. You'd need to deploy thousands of times a month to even get close to the limit.

So the total cost of hosting this blog is: **zero**.

## The unexpected upgrade: blogging from my phone

Here's something I didn't plan for but turned out to be the biggest workflow improvement.

With Payload, writing from my phone was technically possible — the admin panel is responsive — but it was clunky. Loading the CMS, navigating to the editor, dealing with a rich-text interface on a small screen... I never actually did it.

With Hugo, my blog posts are Markdown files in a GitHub repository. And that opens up a completely different set of tools.

### Claude on mobile

[Claude](https://claude.ai/) works on mobile — both the app and the web interface. And since my blog posts are just text files, I can:

- **Start a draft in Claude.** I describe what I want to write about, and Claude helps me outline and draft the post in conversational back-and-forth — all from my phone.
- **Iterate on the go.** During a commute, waiting in line, or lying on the couch — I can refine a post by chatting with Claude. No laptop needed.
- **Generate the full Markdown.** Claude produces the complete `.md` file with front matter, headings, and content. I just need to paste it into the repo.

The key insight is that the "editor" is no longer tied to a specific tool or device. When your content is plain text, any tool that can produce text becomes a viable editor — and AI assistants are very good at producing text.

### GitHub on mobile

Once I have a draft, getting it into the repository is straightforward:

- The **GitHub mobile app** lets me create and edit files directly in the repo.
- I can also use the **GitHub web interface** on my phone to commit a new file.
- For more involved changes, I can push from my phone using a Git client like [Working Copy](https://workingcopy.app/) (iOS).

A push to `main` triggers the deploy workflow, and the post is live. The entire process — from idea to published post — can happen on my phone without ever opening a laptop.

## Comparing the two stacks

| | Payload + Vercel + Neon | Hugo + GitHub Pages |
|---|---|---|
| **Cost** | Free | Free |
| **Admin panel** | Yes (full CMS) | No |
| **Database** | PostgreSQL (Neon) | None |
| **Deploy** | Vercel (auto) | GitHub Actions (auto) |
| **Mobile editing** | Possible but clunky | Natural (Markdown + AI) |
| **AI integration** | Limited | Seamless |
| **Best for** | Structured content, teams | Developer blogs, simplicity |

## The point isn't the tools

Both stacks are free. Both deploy automatically. Both work well for a personal blog.

The difference is in how they fit into your workflow. If you already live in a code editor and use AI tools daily, Hugo removes every layer of friction between thinking and publishing. And if you want to write from your phone, there's no simpler format than a Markdown file in a Git repo.

The best blogging stack is the one that gets out of your way. For me, right now, that's Hugo.
