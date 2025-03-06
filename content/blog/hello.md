+++
title = 'Hello, blog!'
date = 2025-03-05T23:04:08Z
draft = false
summary = "My move to Hugo."
+++

I've wanted to blog on this website, but didn't want to write pages by hand. Not when I've already tried static site generators like [Hugo](https://gohugo.io).

## What's Hugo?

Hugo is a static site generator - it takes content and generates web pages. So this Markdown document:

```md
+++
title = 'Hello, blog!'
date = 2025-03-05T23:04:08Z
draft = false
summary = "My move to Hugo."
+++

I've wanted to blog on this website, but ...
```

Is easily turned into what you're viewing right now.

Since it's all static content, I can host it for free on GitHub Pages. This means it lacks the hosting overhead of something like
WordPress, but is more automated than typing every line yourself.

Granted, I could just copy & paste templates, but writing in Markdown is more my preference.

## Turning my website into a Hugo theme

If you look at [Hugo Themes](https://themes.gohugo.io/), you'll see that there are plenty to get started with. But moving over from handwritten HTML and CSS to something like this is a great opportunity to make my own!

I based mine on the base theme Hugo includes, by running:

```sh
hugo new theme theme-name
```

The folder hierarchy seemed daunting at first, but if you're not changing how the templates work then it's just
normal HTML/CSS/JS work - I really just tweaked my CSS for the theme.

## Final thoughts

It was fun to finally do this, and relatively easy too! Hopefully I can make it count with my posts.