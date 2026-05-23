---
tech: true
draft: false
slug: 'working-with-llms'
title: 'Working with LLMs'
publishedOn: '2026-05-05'
lastEditedOn: '2026-05-23'
---

As we usher in this new era of LLMs, it is interesting to see how different people are starting to work with them. And as a typical keyboard thudding monkey, I want to optimize my workflow too. Because a true master understands tools at it his/her disposal the best.

The way I currently work with them is straight forward way popularized by Claude Code, plan with it first in plan mode, then jump into implementation. I try to manually approve everything, but still I lose context in the "hit enter" hell. To over come it, I sometimes just let it make all the changes and then go back and start editing it. Now, ideally this should work, you plan meticulously and once the plan is solid, bang on, all code will be perfect. Right? Right?

I think that's a wrong model to think how software engineers work. Most of the times, we discover/realize things on the fly, and that could be as small as a super small limited scope change to a complete re-design. So it is more of an iterative loop rather than a one shot model. In that case, one would go in and out of plan mode refining the spec as they learn more.

## My questions

But before trying to refine our process, lets try to come up with points I want answers to:

1. How do I know I have explored all the possible ways to attack a problem, could there a simpler solution?

2. Another is, breaking down abstractions at correct boundaries, I think LLMs struggle with this right now. I see a lot of people dumping code in places where it shouldn't belong in the first place. Why is it dumped there, cause no one cared enough to think about boundaries. Well, this was a problem before LLMs too, but its much more worse right now.

3. Writing code by hand is a process which forces one to slow down and look at the surrounding code, think about frictions we face when coming up with code. Just being lazy and realizing a lot of things. LLMs don't have that (1). How to bring back this process of slowing down? And in what form? Hand-write everything again?

4. How do I trust the tests written by LLM? Amount of people who are not reading generated tests is baffling high. No one, literally no one I know is reading generated tests. They think if there are tests its enough. Amount of times I have found generated tests to not be helpful is actually very high. Like the saying of man goes: "To know a man, check his trash". "To know about an implementation, check its tests".

5. How to find subtle problems within the implementation, Antirez put it nicely: `"but still things that superficially work do not mean they are optimal."`(2)

## User workflows

Before we try to answer these questions, lets try to read Antirez's use of LLMs for array type support in redis (2) (3). Summarizing it the way I understood it:

- He wrote first design draft completely by himself
- Brought in LLM, started attacking draft from different angles, this would have likely required him asking correct questions to LLM
- He read whole code line by line with extreme care. I liked this a lot: `but still things that superficially work do not mean they are optimal.`
- He rewrote the whole implementation again in a mix of manual and LLM mode
- Extensive testing, a complete month dedicated to just that

In his own words towards the end:

```
For high quality system programming tasks you have to still be fully involved, but I ventured to a level of complexity that I would have otherwise skipped. AI provided the safety net for two things: certain massive tasks that are very tiring (like the 32 bit support that was added and tested later), and at the same time the virtual work force required to make sure there are no obvious bugs in complicated algorithms. To write the initial huge specification was the key to the successive work, as it was the key to review each single line of sparsearray.c and t_array.c and modifying everything was not a good fit.
```

As we are at it, these are some ways I have seen people around me use it:
- Clowns: Absolute direct vibe code, this is just dumb
- GreatPretenders: Give the problem to LLM, act like they understand it by saying: "we manually accepted edits", test it on basic cases and ship to production.
- Meticulously try to plan things with it, try to attack from different angles. From here on two more routes:
	- Strategist: Write code using a LLM assisted autocomplete
	- OldieGoldie: Write code completely by hand

We are not going to talk about Clowns and GreatPretenders at all except one statement to these people, PLEASE stop making my life difficult.

## Thoughts on my questions

Now that we have everyone's workflows in place, let's try to come back to our questions. (Answers in the same bullet point number as the question)

1. I like Antirez's approach here, he took a month just to write the spec, and he didn't write first draft with the help of LLM, it was completely by himself. This is where I think Strategist and OldieGoldie's get defeated, I believe key point is: not reading the approach given by LLM first. Cause there are times, they just don't know, and they don't know what they don't know. They are not able to come up with few of the strategies you might come up with. You could call this the creative step or whatever. I have noticed, reading LLMs output first creates a bias in the mind, and also we might get hindsighted on asking the correct questions. That's why try to come up with a plan on your own and then work with LLM to try to attack it from different sides to solidify it.

2. On this part, I think there are two steps where this comes up, first is when planning i.e. 1st step and next is when actually writing code by hand and noticing a friction point. First part is addressable during first step itself, this is usually the easy part. But when it comes to the latter, I think it correlates with 3rd point of mine.

3. Now this is a tricky one, one needs to slow down, we slowed down once in the initial planning phase, but when next? In the iterative cycle I mentioned above, how do we slow down during the actual implementation section to notice these frictions? Well one way is converting into OldieGoldie, it is slow but definitely works! Though one could be lost completely in implementation details and want to complete it fast, which would lead us to the pre-LLM era problem of people writing absolute horrendous code without respecting any abstractions.

	 So, completely automated is bad, completely hand written is dicey, then Strategist wins? Well, I don't think so, again this is a point about slowing down, fancy autocompletes are not a great way to slow down and understand cross module dependencies. Well then what? I like Antirez's way here, seemingly he generated all code first as a PoC, realized few things during PoC to fix, re-generated it, assumed just reading everything in extreme detail would help but he didn't know the answer to: `but still things that superficially work do not mean they are optimal`. So he went back and rewrote whole implementation in a mix of manual and AI-assisted mode.

	 The difference between Strategist and this is using code as a throw-away signal, Antirez used the first version as a PoC, that's it. He then, rewrote the implementation in his own way completely, this makes the process so much faster and more context aware than one shotting the implementation and making abstractions etc on the fly.

4. For this point, Antirez said two things:
	- "Everything was working, and this type has massive testing, thanks, again to AI"
	- "When this stage was done, I started, during the third month, to stress test the implementation in many different ways."

	I don't think there's info on what he did here. As such testing is a very subjective topic and how to do it properly for a particular system is a monster of its own. For now, I try to follow the same procedure as before, try to come up with test cases myself and then involve LLMs to expand upon them on their own and combine to form a better list. This helps avoid a bunch of test cases which add 1K lines of abstractions on their own to test a simple thing.

5. For this one, I think 3rd point above goes in enough details about everything. Key to this point I believe is slowing down and reading code multiple times to try and think from different angles.

This kinda also lays out how I want to try using LLMs going forward.

## Unanswered Questions in Antirezs' article

Taking a small detour and going back to Antirez's article, I have a few things I would have liked to understand in more details:

- When he used in LLM in the planning phase, what part of it was him trying to probe questions out of LLM as an experienced user and what part of it was, LLM finding defects/improvements on its own?
- What part of codebases did he rewrite manually and what parts were rewritten using LLM? How did he decide which part to allocate to who?
- How did he approach testing in general, did he check LLMs generated tests in super detail? Did he rewrite them too? What did his one month of testing look like in detail? How did LLMs help outside unit tests?

## Conclusion

It was interesting to think about these things while writing this article down. I would have not imagined myself thinking about these things because I had previously been haunted by a college senior of mine being too strict on writing down huge number of pages of LLD, HLD, PRD, etc etc for club projects. Ofc we never finished the projects which he was supervising. I still don't know if all of this was coherent or just random rambling. Well, there's one thing for sure, I have something new to try and I will make sure I keep the rigor up in LLM age! [4]

Footnotes:

- (1) https://bcantrill.dtrace.org/2026/04/12/the-peril-of-laziness-lost/
- (2) https://antirez.com/news/164
- (3) https://github.com/redis/redis/pull/15162
- (4) https://oxide-and-friends.transistor.fm/episodes/engineering-rigor-in-the-llm-age
