+++
categories = ["serverless", "security", "development"]
date = "2024-08-17T00:00:00+01:00"
description = "Learn how I built a modern, secure blog for my wife using Astro and TinaCMS. Discover the perfect middle ground between WordPress and static sites, with dynamic content, minimal maintenance, and zero hosting costs"
keywords = ["Astro blog setup", "TinaCMS", "WordPress alternatives", "static site generator", "dynamic content", "GitHub Pages blog", "web development", "web scraping", "Tailscale VPN", "Privoxy HTTP Proxy", "WYSIWYG editor", "custom blog themes", "tech tutorial", "minimal maintenance website", "blogging tools"]
title = "Keeping the Wife Happy: Building a Modern Website with Astro and TinaCMS"

+++

![](/images/building-a-modern-website-with-astro-and-tinacms/headline.webp)

When my wife, Ciara, told me she wanted to start a blog, my initial reaction was, “Oh no, here we go!” I knew setting up a website could quickly become complicated, expensive, and require ongoing maintenance—something I'd rather avoid. So, what's a guy to do?

### The Classic Options: WordPress or Static Sites?

The first thing that came to mind was WordPress. It’s user-friendly, which would make it easy for Ciara to manage content on her own. But, WordPress comes with its challenges. It requires a database, PHP, and regular security maintenance. With my laziness, combined with how frequently WordPress sites get targeted, getting hacked felt inevitable.

On top of that, the WordPress solution would cost money—either in the form of managed hosting, premium themes, or plugins to get everything just right. Sure, you can get started on the free tier, but to have full control over the blog and access to essential features, the costs quickly add up. This option could easily spiral into an ongoing investment, both in time and money.

On the other end of the spectrum, there’s the techie solution—a static site generator like Hugo which is what I use for my own site. Hosting it on GitHub Pages for free, writing content in Markdown, and deploying with a simple `git push`. But expecting Ciara to learn Markdown and Git just to write a blog post? That was never going to fly.

### Is There a Middle Ground?

I started to wonder if there was an in-between solution. A setup that combined the user-friendliness of WordPress with the simplicity, security, and zero hosting costs of a static site generator. That’s when I stumbled upon [Cassidy Williams’ blog post](https://cassidoo.co/post/blog-website-baby/) about using Astro for static blogs and combining it with TinaCMS for content editing. Could this be the magic bullet?

### The Proof of Concept: Astro and TinaCMS

I decided to throw together a proof of concept. Astro worked as a fantastic static site generator, and pairing it with TinaCMS allowed for a WYSIWYG editor that was simple and intuitive for Ciara to use. When changes were made in the editor, TinaCMS would automatically commit them to Git. From there, GitHub Actions would take over, building and publishing the site to GitHub Pages. In short, new content could be live on the site within minutes of her hitting "save."

![TinaCMS](/images/building-a-modern-website-with-astro-and-tinacms/tinacms.png)

This seemed perfect! Ciara could write, upload media, and self-publish, and I would have a minimal ongoing maintenance burden.

### Adding a Personal Touch: Custom Themes and Dynamic Content

![TinaCMS](/images/building-a-modern-website-with-astro-and-tinacms/best-sellers.png)

Once we had the basic blog up and running, it was time to add some personal touches. Ciara picked out a theme she loved, and I got to work customizing it. This is where I started having some real fun.

Ciara sells teaching resources on [Teachers Pay Teachers](https://www.teacherspayteachers.com/), so I thought it would be nice for her blog to feature her best-selling products. A quick bit of web scraping at build time allowed me to pull this content in dynamically. The result? A sleek, personalized blog with live product updates. You can check out the final result [here](https://ciarasclassroom.com/resources/).

But why stop there? I wanted to integrate even more dynamic content, like her latest Instagram posts, into the footer. This turned out to be more challenging than I expected. Instagram’s API was trickier to work with, and scraping was blocked when running from GitHub Actions due to IP restrictions.

### The Solution: Tailscale and Privoxy

![Proxy](/images/building-a-modern-website-with-astro-and-tinacms/proxy.png)

I already use [Tailscale VPN](https://tailscale.com/) for accessing home-hosted resources while away, and Tailscale offers a GitHub Action for establishing VPN connectivity. I set up a [Privoxy HTTP Proxy](https://www.privoxy.org/) on my home router, which allowed GitHub Actions traffic to route through my home internet connection.

![Github Action](/images/building-a-modern-website-with-astro-and-tinacms/github-action.png)

With everything in place, I updated GitHub Actions to run the script with traffic flowing over the HTTP Proxy and set up a scheduled GitHub Action to refresh the content daily. Now, the website updates automatically with new Instagram posts, giving it that dynamic touch without compromising the simplicity and security of a static site.

![Dynamic Content](/images/building-a-modern-website-with-astro-and-tinacms/dynamic-content.png)

### Final Thoughts: The Best of Both Worlds

This journey taught me that static and dynamic content can blend beautifully when you find the right tools. With Astro, TinaCMS, and GitHub Actions, we were able to build a website that feels modern, secure, and easy to use.

Ciara can now self publish content, while I (hopefully) never have to touch it again.
