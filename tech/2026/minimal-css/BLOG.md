---
tech: true
draft: false
slug: minimal-css-for-everything
title: Minimal CSS for everything
publishedOn: 2026-06-10
lastEditedOn: 2026-06-10
---
Today I am going to talk about minimal CSS to make responsive websites. This is the best time to write this down cause I have built [my website](fknil.com) recently, and I fought through a jungle of different loosely connected pieces of ideas I had in my brain to make this website responsive and with no horizontal scroll. Please please remove horizontal scrolls from your websites, it is not that hard. Personally if I come across one, it drives me nuts. And this is not just noobs writing CSS who introduce them, I just saw it yesterday on Github compare page!! A multi-million dollar company, with all the resources in the world, and they can introduce such bugs. Its still present as of writing this blog i.e. 10-06-2026-06-09 in DD-MM-YYYY, check it out [here](https://github.com/apache/datafusion/compare/main...feniljain:datafusion:feat-offset-pushdown).

I was doing a lot of web-dev during my college days. Being part of a super active club, we used to conduct a lot of events! And each one required at least a website, with android/iOS app being optional. We were always in build mode, one event goes, another one is knocking on the door. You would think with all this pressure, why not use something like Wordpress? Well there are two reasons:

- We had some of the best designers, they have wild imaginations and pushed us to our limits always
- We were students and we wanted to learn!

This was pre-LLM era, so all we did was [Fuck Around Find Out](https://www.youtube.com/watch?v=AOPZuXh-f5M). Ahh the golden days. But due to all this grinding, we had become super efficient at base responsiveness of any page. For each project, we would bang out responsive base layouts pretty fast, hard part was things like rendering a 3D globe (well this was a library so we escaped), or making CSS animations with canvas, etc. I was involved in almost all the projects either as a frontend guy or a backend guy, sometimes both :P

Around when I took over most of the development, something flipped in development practices of our team. I had pushed a rule of not using any CSS frameworks. I had recently learnt about flexbox, and it felt magical! I wondered to myself: What have I been doing till this day. Trying to use loads of media queries, z-index, float, display and what not to make the website just barely responsive. Responsiveness was just a weird screen away from breaking. As soon as I learnt about it, and built enough intuition I realized: all these frameworks were helping with was hiding skill issue. There were zero reasons, for a complete learning project to use any frameworks. So they were banned, but this meant literally hand-rolling everything in the whole website, now this could be a CTF website or recruitment portal, anything which the next event and design curiosity led to.

The reason I am talking about this today is I recently read an interesting article from [Matklad](https://matklad.github.io/2026/06/04/css-unavoidable-bad-parts.html). While I follow his blog posts for systems engineering/low level/PL design, etc, this one stuck to me. No-one wants to learn oddities of CSS, this is especially true for people who want to control everything about their website but not be bothered with too much framework complexity, I am still that guy! So following his lead, what are my rules which I follow and want to remember to make this happen?

With this motivation, I am going to give you a recipe which has worked for me over the years! Lets get started:

## Box Sizing

First set:
```css
* { box-sizing: border-box; }
```

Make all the elements use `border-box`. By default CSS does not include padding and border as part of the height and width calculation of the element, it just feels unintuitive to me, as well as to Matklad too xD

[MDN link](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/box-sizing)
## Flexbox

```css
display: flex;
```

Anywhere where you want to arrange multiple elements in a way where they can wrap, you want them to be strictly stacked, strictly next to each other, strict space between or any such scenarios, use flexbox. I am not doing a proper justice to how important and life changing flexbox is. So, for this one thing, do sit and read full [MDN reference](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/CSS_layout/Flexbox). It is really good! And you would definitely forget it, so next time you will have to come and see this again for few things xD

Next is:

```css
justify-content: center;
align-items: center;
```

How to center a child div? SOLVED!

I am not even kidding, this is it. And its not just this, `justify-content` has other values like `start`, `end`, `space-between`, `space-around` and `space-evenly`. Again [MDN page](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/justify-content) gives a good interactive example for these properties.

and same for `align-items`: `stretch`/`center`/`start`/`end`.

There is one question to ask though, why two properties? why not just one? Well for that we will have to understand flexbox a bit and this explanation will also double down as our introduction to next two flexbox properties. Again, I am only going to touch just enough to give an idea, for detailed documentation do refer to its official [MDN page](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Flexible_box_layout/Basic_concepts).

Flexbox has two directions in which it tries to reason about its child element. One is called the `main axis` and other is called the `cross axis`. By default, `main axis` is horizontal and `cross axis` is vertical. So, if you say:

```css
justify-content: start;
align-items: start;
```

You can expect all children to be in the left hand side upper corner of your parent element. Or, if you set:

```css
justify-content: end;
align-items: end;
```

they will be in the right hand side lower corner. There are loads of different combinations. You should personally play with these combinations to get the best feel! Once you understand these, it will start to feel like superpower :)

There is one catch though, values of main and cross axis can change according to `flex-direction`, which can be `row`, `row-reverse`, `column` or `column-reverse`. Main axis in each case is:

- `row`: left-to-right
- `row-reverse`: right-to-left
- `column`: top-to-bottom
- `column-reverse`: bottom-to-top

One can easily infer about cross-axis. And finally there's `flex-grow`. This is used to tell an element to take a stretch factor in free space. A simple example to understand it would be:

Imagine there's a website with body set as 100% of screen space. At the top there's a header and at the bottom there's a footer, in the middle you have dynamically sized content. Header is of height 10% and footer, 5%. Now we can set `flex-grow` on middle element to `1`, which would make it automatically occupy all the remaining space between them.

In this example one could compute the height of middle element as 85%, but think of multiple elements of varying size, some fixed, some dynamic, all of them interacting with each other and they are dynamically inserted in any order, having properties like `flex-grow`, helps in that case!

There are other properties on flexbox, but I don't remember now if I used any regularly in college, at least I didn't need any in my current website.

## Single media query

Just use:

```css
@media (max-width: 480px) { /* all needed CSS */ }
```

for any mobile phone specific stuff. Important to note point will be: it is only to be used for things like: image size is 30% on desktops, but on phone it needs to be 50%. DO NOT use it to get responsiveness. Flexbox should have everything you need to achieve that!

## Margin on body

For some reason Firefox (not sure about other browsers), had a default margin of 8?? This caused so much confusion to me :(

Setting it zero removed horizontal scroll and overflow I was debugging xD

By default just remove it on `html` and `body` tag:

```css
html, body {
    /* by default it has 8 margin 🤦 */
    margin: 0;
}
```

From Matklads blogpost I also learnt about other such non-intuitive things browsers set and hence every website should perform a so called "CSS Reset", basically a small set of sane properties to keep at the the top of project and start building from that base. One linked by Matklad was [this](https://www.joshwcomeau.com/css/custom-css-reset/).

## Percent over pixels for margin and padding

Percentage scales better for different screen sizes by default, so stop hard-coding pixels! Btw these pixels do not map to actual pixels of your screen. Its just a logical construct which has a mapping decided by browser to the actual physical pixels!

There could be a case where margin and padding looks off on different screen resolutions. I usually keep two sets, one for mobile and one for any other screens.

## Rem over pixels for font size

Again don't hard code using pixels, just use rem. While its intuitive, a [good read](https://matklad.github.io/2022/11/05/accessibility-px-or-rem.html) on the same was linked in Matklads post.

## CSS variables for light and dark mode

Use global CSS variables for setting light and dark mode colors and use them with system preferences like this:

```css
:root {
    --bg: #fafafa;
    --fg: #212121;
    --muted: #5a5a5a;
    --logo-backdrop: #1565c0;
    --alt-bg: #e0e0e0;
    --border: #d0d0d0;
    --text-color: #212121;
}

@media (prefers-color-scheme: dark) {
    :root {
        --bg: #212121;
        --fg: #dadada;
        --muted: #a0a0a0;
        --logo-backdrop: #42a5f5;
        --alt-bg: #424242;
        --border: #3a3a3a;
        --text-color: #dadada;
    }
}
```

This would change theme of the website according to system preferences by default. You can also force it using a button on your website :)

## Don't forget LVHA

The `Love/Hate` relationship of CSS. Its basically a rule covering how CSS loads properties on anchor tag in a particular order:

- `:hover` must come after `:link` and `:visited`
- `:active` must come after `:hover`

If you does not follow these, you might see properties being overridden randomly. So always remember these!

## Semantic tags

Try to use HTML semantic tags like `<em>` for `italic`, `<strong>` for `bold` , etc. These are for text decoration, but there are tags for loads of other things like `lists`: `<ul>`, `<li>`, navigation lists etc. There are a ton of semantic tags baked into HTML, try to make use of them as much as possible and leave the defaults as they are!

##  Divs

And finally, use divs liberally when trying to make sense of some responsive layout, I usually find giving borders of different colors to different divs a pretty good way to debug CSS issues. Now, I know we have a very powerful `Inspector`, but you know to debug two very different elements together in a complex hierarchical top-down dependent system (flexbox all the way down), this trick is useful xD

## Conclusion

And that's it, a good example of all these tips in use is my website's [codebase](https://github.com/feniljain/fknil). Not saying, these are the golden rules which are absolute perfect, there could be things wrong with this. Accessibility comes to mind as the major footgun when discussing webdev. So, would love to hear about things which can go wrong or are incorrect! Thanks for reading and until next time, ciao!
