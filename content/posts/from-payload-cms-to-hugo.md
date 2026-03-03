---
title: "From Payload CMS to Hugo: Writing Blog Posts as Code"
date: 2026-03-01
draft: false
ai: true
tags: ["Hugo", "Blogging", "AI", "Workflow"]
description: "Why I moved from Payload CMS to Hugo, and how writing blog posts in VS Code with AI assistance changed my workflow for the better."
image: "images/posts/from-payload-to-hugo.png"
---

For the past while, my personal site ran on [Payload CMS](https://payloadcms.com/) — a powerful, developer-friendly headless CMS built on Node.js. It served me well, but over time I started feeling friction in my workflow. So I switched to [Hugo](https://gohugo.io/), a static site generator where every post is just a Markdown file in a folder.

This post is about why I made that change, and how it unlocked something I didn't expect: a much tighter feedback loop with AI tools like Claude.

## What was wrong with Payload?

Nothing, really. Payload is an excellent CMS. It gives you a full admin panel, flexible content models, and everything lives in code. But for a personal blog, it was overkill:

- **Deployment overhead**: a Node server, a database, environment variables to manage. Every time I wanted to tweak something, I had to spin up the full stack.
- **Context switching**: writing a post meant opening a browser, navigating to the admin panel, and using a rich-text editor that — while decent — was never *my* editor.
- **AI-unfriendly**: the content lived in a database. I couldn't easily hand a blog post to an AI assistant and say "help me improve this paragraph" without copy-pasting back and forth.

The final straw was when I tried to integrate AI features into Payload's editor. I went down a rabbit hole of plugins, Node.js dependencies, and custom configurations — and the result was still a fraction of what I get for free by just writing Markdown in VS Code with Claude. That's when I realized the tool was working against me, not for me.

## Why Hugo?

Hugo is the opposite end of the spectrum: no database, no server, no admin panel. Your content is Markdown files. Your configuration is a single TOML file. You run one command and get a static site.

What made it click for me:

- **Everything is a file.** Posts, configuration, templates — it's all in a Git repository. I can see the full history of every change I've ever made.
- **The editor is VS Code.** I already spend most of my day here. Writing a blog post feels like writing code — because it *is* just writing in a text file.
- **Builds are instant.** Hugo compiles the entire site in milliseconds. No waiting, no build pipelines to debug.
- **Zero runtime dependencies.** The output is plain HTML/CSS/JS. I deploy to GitHub Pages with a simple workflow and forget about it.

## The real game-changer: AI in the editor

This is the part I didn't fully anticipate. Once my blog posts became plain Markdown files inside a project, they became first-class citizens for AI-assisted workflows.

With Claude integrated in VS Code, I can:

- **Draft and iterate in place.** I write a rough version of a post, highlight a section, and ask Claude to tighten it up — all without leaving my editor.
- **Get structural feedback.** "Does this post flow well? Is the argument clear?" Claude can read the whole file and give me feedback as if it were reviewing a pull request.
- **Generate front matter and metadata.** Tags, descriptions, summaries — the tedious parts that I used to skip or half-do.
- **Work with context.** Because the post lives alongside my code, Claude can reference my project structure, my other posts, even my site configuration. It understands the full picture.

With Payload, this kind of interaction required constant copy-pasting between the browser and the editor. With Hugo, it's seamless: the file is right there.

## The workflow now

My blogging workflow has become almost indistinguishable from my coding workflow:

1. Create a new `.md` file in `content/posts/`
2. Write a rough draft
3. Use Claude to help refine, restructure, or expand sections
4. Preview locally with `hugo server -D`
5. Commit, push, done

There's no admin panel to log into, no deploy to babysit. It's just files, Git, and my editor.

## Tradeoffs

To be fair, this setup isn't for everyone:

- **No visual editor.** If you prefer WYSIWYG editing, Hugo's Markdown-only approach might feel limiting.
- **No visual media management.** Hugo can resize, crop, and optimize images — but there's no UI for it. No drag-and-drop upload, no visual crop tool. You configure image processing in templates or shortcodes, which is powerful but requires upfront setup.
- **Theme customization requires Go templates.** Hugo's templating language has a learning curve if you need to go beyond what your theme provides.

For me, these tradeoffs are worth it. I'd rather have full control in my editor than convenience in a browser.

## Final thoughts

The shift from Payload to Hugo wasn't really about which tool is "better." It was about aligning my blogging workflow with the way I already work: in a code editor, with version control, and increasingly, with AI assistance.

If you're a developer who blogs (or wants to start), and you find yourself fighting your CMS more than writing, consider going static. The simplicity might surprise you — and the AI integration possibilities will only keep getting better.

And yes, there's a certain irony here: the vast majority of this very post was written by Claude. Which, if anything, proves the point — the workflow works. In the interest of transparency, every post on this site that was written with AI assistance is automatically flagged with a banner. I think that's the least you can do when the machine is doing most of the typing.
