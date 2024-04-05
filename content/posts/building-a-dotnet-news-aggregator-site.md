---
title: "Announcing dotnetramblings.com or how to build a .NET news aggregator site."
date: 2024-04-05T17:15:12+02:00
description: "Let's check how I build my new site www.dotnetramblings.com. This site is a .NET news aggregator that updates its content every three hours. The main technologies employed to build it include Hugo, Python and Github Actions." 
tags: ["dotnet", "github", "cloud", "microsoft", "azure", "python", "devops"]
draft: false
---

I frequently visit numerous news aggregator pages, which are fantastic because they allow me to find the latest .NET articles from various sites on a single page.

However, how challenging could it be to construct one of these sites? It turns out it's pretty easy. I managed to build one of them in a couple of free afternoons.

You can visit it by going to:
- www.dotnetramblings.com

The site is fully automated; I don't need to lift a finger to update the content on it. Every few hours, it fetches the latest news from a series of RSS feeds and updates the site autonomously.

In this post, I will show you how I built it.


# **What I have built?**

The site utilizes the following technologies:
- The site is built using [Hugo](https://gohugo.io/). Hugo is one of the most popular open-source static site generators.
- A Python script that fetches the latest news from a series of RSS feeds and programmatically updates the site content.
- A GitHub Action that runs the Python script.

Below is a diagram illustrating the entire process.

![dotnetramblings-diagram](/img/dotnetramblings-diagram.png)

As you can see, the process is incredibly simple, but there are a couple more components worth mentioning:

- Another Python script removes news older than one week from the site, preventing uncontrolled growth of the site. This script is called by the same GitHub Action.
- A second GitHub Action is responsible for publishing the Hugo site. The first GitHub Action (as shown in the diagram above) fetches the latest news and updates the site content, but after updating it, it needs to be deployed somewhere. In this case, we're deploying it to another GitHub repository with GitHub Pages enabled, utilizing this secondary GitHub Action.


# **Do you want to contribute?**

If anyone wishes to contribute by adding an RSS feed from their own site or a site they favor. Here's how to do it.

## **How to add your site**

### Prerequisites

- A functioning RSS feed that can be accessed via the Internet.

### Enrollment Process

1 - Visit my Github repository: https://github.com/karlospn/building-a-dotnet-news-aggregator-site

2 - The ``/data`` folder contain the current feeds utilized by this site.

3 - Create a new ``yml`` file with the following attributes:
 - ``Feed (required)``: The URL of the RSS feed.
 - ``Title (required)``: The name of your site.
 - ``Website (required)``: The URL of your site.
 - ``Description (optional)``:  A brief description of your site's content.
 - ``Author (optional)``: The name of the site's author.
 - ``Language (optional)``: The language of the site.

Here's an example:

```yml
Feed: https://www.mytechramblings.com/index.xml
Title: My technical ramblings
Website: https://www.mytechramblings.com
Description: Technical ramblings from a software engineer
Author: Carlos Pons
Language: English

```

4 - Open a Pull Request and await my approval.

5 - Your posts will then begin to appear on our site.

