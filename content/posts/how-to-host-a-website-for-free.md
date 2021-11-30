---
title: "How to host a static website for free"
date: 2021-10-18T09:34:33Z
draft: false
---

If you have a static website that should be served, there is no need for a hoster that costs money. You can use [Google Clouds Object Storage](https://cloud.google.com/storage) or [Github Pages](https://pages.github.com/) like i did.

I use [Hugo](https://gohugo.io/) a static website generator to transform the raw text written in [MarkDown](https://www.markdownguide.org/) into a static website. The transformation is handled automatically by [Github Actions](https://github.com/features/actions) when i commit my local workspace to the [Github repository](https://github.com/pcbaecker/www-pcbaecker-com).

Github pages then displays a specific branch of the repository as a website. You don't need a tool like hugo. If you want, you can directly commit html files and publish them with Github pages for example on the master branch. In my case there is a branch called 'gh-pages' which contains the html files and will be served under pcbaecker.github.io.

If you want to serve the website with a custom domain, you can just enter the name and github will serve the website if a request for the domain is made.

![Settings for Github pages](/static/images/github-pages-settings.png)

In order to make requests with the custom domain to github pages you must configure the domain with CNAME entries that point to the original github pages subdomain.

![DNS settings on Cloudflare](/static/images/cloudflare-dns-for-github-pages.png)

After that being done, every request to your custom domain will be proxied by cloudflare and served by github pages. You may want to configure cloudflare page rules to redirect for example https://pcbaecker.com to https://pcbaecker.com.

This website is an example for this procedure and can the repository can be found [here](https://github.com/pcbaecker/www-pcbaecker-com).