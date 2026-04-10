---
title: Translaitor
date: 2025-06-09
lastmod: 2025-06-09
author: whale
tags:
  - blog
  - AI
  - book
  - french
  - IT
  - sveltejs
  - web
  - english
---
A couple of days ago I found that there are many books `pirates of the caribbean`. Unfortunately they are all in English and since I'm French and I didn't want to read a 400 pages book in its original language. So I decided to make a tool that would be able to translate an entire book using AI. Fortunately, the [Gemini API](https://gemini.google.com) has a pretty good free tier.

I began the work using [SvelteKit](https://svelte.dev/docs/kit/introduction), [SkeletonUI](https://www.skeleton.dev/), and [TailwindCSS](https://tailwindcss.com/). Here is the repo link: [https://github.com/TheWhale01/translaitor](https://github.com/TheWhale01/translaitor)

I've done something pretty usable but gemini's rate limiting is very annoying so I've decided to run an LLM locally ! Google has a very good translation model for this type of work and very lightweight ! Thanks to ollama, I can easily run it and query its API to asks for translations.

[TranslateGemma](https://ollama.com/library/translategemma)

The goal would be to:

- [ ] upload a book to the server
- [ ] extract the content of the epub
- [ ] for every paragraph / block of text
- [ ] send it to the local LLM
- [ ] Write the response to a new file
- [ ] Once every paragrah has been written, compress to epub and send it to the client

For now it's in its early stages but I can translate entire books without being restricted by a free-tier API