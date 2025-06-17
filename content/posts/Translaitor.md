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

For now I need to understand how to send the file to the server. I already know that epub files are just zip files and they contain other files with markup language so it's very easy to parse and reproduce.

The goal would be to:
- [ ] upload a file
- [ ] send it to the server
- [ ] extract the content of the epub
- [ ] for every paragraph / block of text
- [ ] send it to the Gemini API
- [ ] Write the response to a new file
 once everything has been translated, zip the files to have the translated epub

A good thing would be to find the best prompt in order to have the best translation possible. Because since then I used the calibre extension to do that, but I saw that some names would be translated or not and some expression are not so good translated.