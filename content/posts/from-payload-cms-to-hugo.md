---
title: "The Blog That Almost Didn't Happen"
date: 2026-03-01
draft: false
ai: true
tags: ["Hugo", "Blogging", "AI", "Workflow"]
description: "I had Payload CMS, VS Code, and things to say. I still wasn't publishing. This is about what changed, and why it took an AI to get me out of my own way."
image: "images/posts/from-payload-to-hugo.png"
---

For about a year, my personal site ran on [Payload CMS](https://payloadcms.com/), a powerful, developer-friendly headless CMS built on Node.js. It served me well, but over time I started feeling friction in my workflow. So I switched to [Hugo](https://gohugo.io/), a static site generator where every post is just a Markdown file in a folder.

This post is about why I made that change, and how it unlocked something I didn't expect: a much tighter feedback loop with AI tools like [Claude](https://claude.ai).

## What was wrong with Payload?

Nothing, really. Payload is an excellent CMS. It gives you a full admin panel, flexible content models, and everything lives in code. For years I've been fascinated by CMS systems, and in the pre-AI era it felt natural to explore something popular, open-source, and technically interesting: Payload checked all those boxes. But for a personal blog, it was overkill:

- **Deployment overhead**: a Node server, a database, environment variables to manage. Honestly, Vercel handles most of this brilliantly, so this wasn't really the issue.
- **Context switching**: writing a post meant opening a browser, navigating to the admin panel, and using a rich-text editor. And actually, this was a *feature*: I could write from any device, without needing my computer. Not a complaint.

The real problem surfaced when I tried to integrate AI into my writing workflow. Adding AI assistance to Payload isn't trivial: it requires building custom plugins, wiring up API calls inside the editor, and navigating a fairly complex Node.js ecosystem. I went down that rabbit hole, and even [Claude](https://claude.ai) couldn't find a clean path forward. Not because Claude wasn't helpful (it was), but because the problem itself is genuinely hard, and I simply don't have the advanced technical chops to see it through. On top of that, any AI you embed into a CMS is inherently context-limited: it sees one field at a time, with no awareness of the rest of the post, the site's tone, or anything beyond the cursor.

That's when I realized the tool wasn't the problem: I was asking the wrong tool to do something it wasn't designed for. Payload is built for structured content at scale, not for a solo blogger who wants to think out loud with an AI co-pilot. The mismatch wasn't Payload's fault. **It was mine.**

## Why Hugo?

I won't spend much time on the generic part. Hugo has no database, no server, no admin panel: your content is Markdown files, your config is a TOML file, you run one command and get a static site. This is well documented and endlessly evangelized elsewhere.

What actually made it click for me was something more obvious in retrospect: I spend nine hours a day in Xcode or VS Code. Every problem I solve at work lives in a file somewhere. Every change I make is a commit. My entire mental model of "getting things done" runs on text files and Git. The moment writing a blog post became the same gesture as writing a Swift extension (open file, type, save, commit) it stopped feeling like a separate activity with its own overhead. It felt like work, which sounds like a strange thing to recommend, but for me it's actually the point.

The admin panel wasn't the problem. It was the context switch. The fifteen seconds of opening a browser, navigating somewhere, waiting for a rich-text editor to load: that's not a lot of time, but it was enough friction to make me choose not to do it. Removing that friction didn't change my intention to publish. **It changed whether I actually did.**

## The migration: from an empty folder to a working site

The actual migration was one of the most interesting parts of the whole experiment, because I did almost none of it manually.

I started with an empty folder and asked Claude to set up a Hugo site from scratch. I chose [Mana](https://github.com/Livour/hugo-mana-theme), a minimal, developer-focused theme that matched the aesthetic I had in mind. Claude handled the scaffolding, the configuration, and the theme setup.

Then I asked it to go to my old Payload site, fetch the two posts I had published there, and migrate them to Hugo: content, structure, and images included. It pulled everything, converted it to Markdown, placed the images in the right folders, and wired up the front matter. What would have taken me an afternoon took a few minutes. It also preserved the exact same URL slugs as the old Payload site, so existing links and search rankings didn't break.

The trickiest part was the public speaking section. I have a handful of talks recorded on YouTube, and I wanted to list them on the site. The obvious solution (embedding YouTube iframes) was off the table. YouTube embeds set third-party cookies, which under European regulations would require a cookie consent banner. I don't want a cookie banner. I want a clean, fast, private site.

So instead, I asked Claude to fetch each YouTube URL I provided, extract the video thumbnail, and display that as a static image linking to the video. No iframe, no tracking, no cookies. The user clicks the thumbnail and lands on YouTube directly. Their choice, their context. The site stays cookie-free.

It's a small decision, but it reflects something I care about: the web doesn't need to spy on you just because you watched a conference talk.

The only analytics on this site is [GoatCounter](https://www.goatcounter.com/), a free, open-source tool that collects completely anonymous page view counts. No fingerprinting, no cross-site tracking, no personal data. I can see that someone visited a page; I have no idea who.

## The real game-changer: AI wherever I write

This is the part I didn't fully anticipate, and I'm still not sure I can describe it well.

When I tried to embed AI into Payload, the best I could have achieved was a smarter text field. The AI would have seen whatever was in front of the cursor. It wouldn't know the title, the structure, the other posts, the tone I was going for, or the fact that I'd already made this same point three paragraphs earlier. **It would have been autocomplete with a larger vocabulary.**

With Hugo, the post is just a file in a project. Claude can open it the same way it opens any other file. It can read the whole thing, tell me where the argument falls apart, notice when I'm repeating myself, check whether the tone shifts awkwardly halfway through. It can cross-reference my other posts and tell me if I'm covering the same ground. It can look at my site configuration and understand that this is a personal blog with a specific kind of reader in mind, not a corporate knowledge base. The context isn't a single text field anymore: it's everything, the same way it would be for a collaborator who had actually read your work.

That shift, from "AI inside a tool" to "AI alongside everything", turned out to be the thing I was actually missing. Not a smarter editor. A collaborator that could see the whole picture.

And once you have that, the way you work changes in ways that are harder to articulate than a bullet point list. I'll try anyway in the next section.

## The workflow now

My blogging workflow has become almost indistinguishable from my coding workflow:

1. Give Claude the topic, some rough notes, or just a vague idea
2. Claude creates the file, writes the front matter, and drafts the whole post: structure, prose, everything
3. I review, adjust tone, correct anything that doesn't sound like me
4. Preview locally with `hugo server -D`
5. Commit, push, done

There's no admin panel to log into, no deploy to babysit. It's just files, Git, and my editor.

## Tradeoffs

To be fair, this setup isn't for everyone:

- **No visual editor.** If you prefer WYSIWYG editing, Hugo's Markdown-only approach might feel limiting.
- **No visual media management.** Hugo can resize, crop, and optimize images, but there's no UI for it. No drag-and-drop upload, no visual crop tool. You configure image processing in templates or shortcodes, which is powerful but requires upfront setup.
- **Theme customization requires Go templates.** Hugo's templating language has a learning curve if you need to go beyond what your theme provides.
- **AI-generated content risks sounding generic.** This is a real one. The more you delegate to an AI, the higher the risk that your writing loses its edge: the specificity, the opinions, the voice that makes it *yours*. AI prose tends to be competent but safe. If you're not actively pushing back, adjusting, and injecting your own point of view, the result can feel polished but hollow. The tool doesn't know what you actually think. That part is still on you.

For me, these tradeoffs are worth it. I'd rather have full control in my editor than convenience in a browser.

In my specific case, there's an extra dimension: I'm not a native English speaker, and Claude simply writes better English than I do. That's not a small thing. But I review every section carefully, not to fix grammar, but to make sure the ideas being communicated are actually mine. The words are Claude's; the opinions, the emphasis, the things I choose to keep or cut: **those are mine.**

And honestly? Without AI, I wouldn't publish anything at all. Not because I don't have things to say, but because time is the real bottleneck. AI doesn't just improve the writing: it makes the writing *happen*.


## Final thoughts

The shift from Payload to Hugo wasn't really about which tool is "better." It was about finding the setup where I can actually get things done, where the friction is low enough that publishing feels like a natural extension of how I already work, rather than a separate effort requiring its own mental overhead.

I'm not sure this workflow would work for everyone. But for a non-native English speaker who has things to say, limited time to say them, and an AI that writes better English than he does: it works remarkably well.

In the interest of transparency: this very post was written by Claude. I gave it the ideas, the opinions, and the editorial direction. It handled the words. I think that's worth being upfront about, which is why every AI-assisted post on this site is flagged with a banner at the top of the page, driven by a custom `ai: true` field in the front matter. I find it slightly uncomfortable, if I'm honest. There's something strange about reading your own thoughts back in prose that's cleaner than anything you'd write yourself. But the alternative (not publishing, not sharing, staying silent because the friction was too high) felt worse. So I made peace with it.
And the least I can do is be upfront about it.
