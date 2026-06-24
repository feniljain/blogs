---
tech: true
draft: false
slug: 'new-website-2026'
title: 'My new small space in the vast web'
publishedOn: '2026-05-24'
lastEditedOn: '2026-05-24'
---

## Motivation

I have been pilled by "write more" propaganda, and its not something new, I have written [about it before](https://fknil.com/blog/writing-more-2024/) but never actually made progress on it. Something has changed this year, I utilized my new year to go through the book of `Accidental Genius`, it talks about freewriting, basically a concept about just typing out words till the editor in your brain takes a step back and you can get your original ideas out, a process of self discovery you can say. Well this is one benefit, mostly importantly I really liked writing as a means to gain clarity. This could work for anything life, work, problem solving, etc. To be frank, I hadn't written much after reading the book in January, but you know there are few things which grow over you? I think the idea has grown over me. So now, I just bang out lines, edit it once maybe and hit `push`. I even migrated to using markdown files for this, lower the friction to actually publish a blog post, the better.

This is one part of the equation, second is, I have grown to start liking [IndieWeb](https://indieweb.org/) concept. I first discovered the concept through [Susam Pal's Wander instances](https://codeberg.org/susam/wander). When I first visited it, I felt like my child like curiosity came back again, it was so wonderful to visit people's websites, read what they have been up to in life/tech, etc. I wanted a little space of my own too, not as fancy as some of those neocities pages, but just my own little simple space where I can host my own interests.

Thirdly, I have finally been able to get my RSS reader set up! I use [newsboat](https://github.com/newsboat/newsboat), its simple and it works! This time I made sure to very slowly add URLs to my feed, and also do regular scrutiny of which feed I want to keep. This has allowed to sustain a habit of actually opening RSS reader everyday and looking forward to new content 😁. Btw one of the key to achieving this was also having a rather regular cadence feed, for me that is [DailyWTF](https://thedailywtf.com/) and [xkcd](https://xkcd.com/). Its so fun to read them! With all this working out, of course I want a RSS feed of my own writing and we need a hosted space for that to happen, so why not build one!?

## Execution

Now lets talk about details, how I achieved it, what I liked about the way I took and what I didn't.

### Requirements

- Keep writing blog posts in markdown files
- Simple framework without any client-side javascript
- Still able to use `npm` packages, basically easy to leverage community work
- Keep draft blogs private if possible
- Easy to deploy

I dont want to increase friction of me writing, a simple markdown file, edit in obsidian and that should do it!

I want to have it load super fast!! No heavy frameworks like React, Vue, etc. I really don't want client side javascript. It should be build once, keep serving always after that! Basically a static site.

There are a bunch of neat things which you can find in `npm` packages like better markdown parsers, fancy wallpapers, dangerous malwares 💀 (:P). I definitely want to be able to use some of them if possible.

### Ditching hugo

For my old website, which my old self made, I used a simple Hugo template. It might have been easy to get up and going at that time, but I am not a fan of these templating engines myself. When I decided to revamp the website, I realized I needed a hugo binary on a specific version, cause it was not building with latest! Some people push their binaries itself to the repo, but what about changing machines? Working across different architectures, etc. I decided I did not want to deal with this crap. So what to do now? Well recently, I migrated my resume from all these templating engines to a custom HTML/CSS file. It is so beautiful now, I can make any changes, add as many points as I want, lay it out as I want, this is what I was looking for! LaTex? Hold my gun, HTML/CSS are always the original king 👑.

I wanted to replicate same success with my new setup too, so that's why I started writing my website in .... HTML and CSS. I could do all my pages with it, but what about markdown files, I would have to write scripts for rendering them? And then actually managing/using them in website is also a bit painful.

### Enter Astro

While I was figuring all this out, I attended a IndieWebClubBengaluru meetup and there, some of my old colleagues showed me their websites which were built using Astro. I initially wasn't interested as I thought it would be another heavy client side javascript framework. But then they stared showing me their lighthouse score, how fast their sites loads and how it did not have "client heavy javascript"! Bingo, that was all I wanted to hear. That's when I started exploring Astro, and went through full `guide` section in its [documentation starting from here](https://docs.astro.build/en/concepts/why-astro/).

And OH BOY!! I had hit jackpot, somehow these guys had exact thoughts like I had and had everything needed to fulfil my requirements. Markdown files to html? Content management from a third folder of all the markdown content? No client side javascript? RSS? EVERYTHING WAS PRESENT! And everything was supported first class!

There is just one thing amongst my requirements, which you would be confused about, how do you `keep draft blogs private if possible`, that's technically not something these frameworks can support. Well, to solve this I created a content collection, basically a folder which has all the blog posts. Astro would walk all the dirs/sub-dirs in this collection and make an array of blog posts. Main catch is, this folder is a [separate repo](https://github.com/feniljain/blogs) in my case. I added it as a submodule, and while it is public for now, I can convert it into a private one at any anytime. I can clone the submodule because I have access to it, but others can't. I can still keep the website code open though! Match made in heaven xD.

### Not so good parts

So till now, we discussed how Astro is ticking all the boxes, but nothing is perfect, there were a few caveats. First is resizing images in markdown. Images can be of varied size but resizing them dynamically is a very common usecase. Well it seems like Astro does not have good support for the same with markdown files. I had to convert my blog of [chinaga betta hike](https://fknil.com/blog/chinaga-betta-hike/) to a MDX file and inject Astro specific JSX to get it working, this was a huge bummer for me. Before moving on to MDX I tried a bunch of things:

- Use `img` tag in markdown file with local path
- Use `span` and `div` tag in markdown file with local path
- Try to use `githubusercontent` URL of the image with height and width mentioned in URL

None of these worked 😓. At the end, I had to shift to MDX which obsidian does not support out of the box, it does not even show it in UI!! Well this was one grievance.

Next was, `astro/@rss` by default does not support linking to images. So if I convert my markdown files to HTML for RSS consumption it does not separate out images in those with some kinda permalink. I am not sure if this an Astro problem, but the OutOfTheBox experience kinda broke here. For now, I am sending MDX files directly in RSS feed without even linking to the images :(

One last problem, was a surprising behaviour. When I first added frontmatter to all my blogs, registered it as content collection, Astro's hot refresh didn't pick them up at all. There were no errors in logs, browser console anywhere?? I had my `npm run dev` running from the morning since I have been building website incrementally. But I for some reason decided to kill and restart the server, and low and behold it didn't start. It failed with an error. Frontmatter of one of the markdown files was not proper (it was reading root README.md too). Well well, I spent so much time, tweaking `content.config.ts` and frontmatter for all the blogs, this was very frustrating.

## Further plans

But you know, as they say, all's well that ends well. Except it hasn't ended, I will take a pause on development for now, but will come back to try to make fixes for the image handling flow for HTML rendering and RSS readers.

I don't want this to be a passion project for now, it is more of a means to an end, to fulfil the core motivations I listed in the beginning of this blog. I want to take it slow, not try to go full speed once and then never come back to touch it. Core goal is writing more, so we will focus on that above all!

There's also one thing I wanna do before taking on work of image flow fixing, that is adding wander instance to my webpage, that would be fun to have, a corner of mine in the small/indie web community :)

For now, you can find [my website here](https://fknil.com/) and [RSS feed here](https://fknil.com/rss.xml). Till next time, chao! 😺
