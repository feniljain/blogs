---
tech: false
draft: false
slug: india-foss-2026
title: IndiaFOSS 2026
publishedOn: 2026-09-28
lastEditedOn: 2026-09-28
---
<p align="center">
<img src="/rss-images/blogs/non-tech/2026/india-foss-2026/indiafoss.webp" alt="IndiaFOSS logo on a banner" style="width: 30%; height: 20%" />
</p>

[IndiaFOSS](https://fossunited.org/indiafoss/2026) is [FOSS United](https://fossunited.org/)'s annual flagship conference. It aims to bring together India's technical community under one roof to discuss anything and everything Open Source. Its 2026 edition happened on 26th and 27th September and what a blast it was. Amazing talks, engaging booths, inspiring personalities, loads of random catchup, fascination, motivation, confusion, all together in a neat two day package! Let's unfold the same `:)`

# First day

With heavy "first day, don't miss anything" motivation, I reached well before the `Welcome Note`. After fuelling ourselves with delicious breakfast and coffee, we headed straight towards Hall 3. This was the allotted room for `Hardware` and `Compilers, Programming Languages and Systems` tracks. I was super excited to find that we had Hall 3 booked this year, cause last year we were doing these devrooms in small classrooms. They were just enough to squeeze 30-40 people inside it, and I missed few entire devroom tracks for the same reason.

Seeing the big hall made me super happy. I took my seat and was ready for the showdown. These were some of the talks I loved in the hardware track:

- [Build your own open source keyboard with ZMK/QMK](https://fossunited.org/c/indiafoss/2026/cfp/32f9jjkob0) 
	- Even though I love keyboards, I did not know a lot about building one from scratch. I have been super happy clacking my [Keychron K2V2](https://www.keychron.com/products/keychron-k2-wireless-mechanical-keyboard) in this well.
	- Maybe just maybe it's time to build the [dactyl](https://www.diykeyboards.com/featured-products/product/76-dactyl-manuform) I've always dreamed about.
- [CNC4Everyone](https://fossunited.org/c/indiafoss/2026/cfp/1cbj5bhfhf)
	- I entered this talk expecting to see a CNC machine like [Jeff Gerling](https://www.youtube.com/watch?v=DJ6dWOVZh2k) had shown few months ago, but to my surprise, this was a machine which works with cardboards!
	- There was one quote which was repeatedly used by the speaker: "China can fill a stadium with CNC operators and we won't fill a room". Keeping aside the national sentiments, I did not realize there were so few CNC operators in India. My friend's brother owns a CNC machine and I know a startup targeting CNC machine operators as their customer base! It could be a metaphor but I do think CNCs are more common than they are realizing.
- [My zero-to-hero journey deploying a fiber-optic and wireless community mesh](https://fossunited.org/c/indiafoss/2026/cfp/9e9bajmm07)
	- This was easily one of my favorite talks. This guy has peak FAFO energy!! I have a personal respect for people who go out there and just figure stuff out. No overthinking about consequences or how hard it would be, none of that matters.
	- In this talk Kiran talked about his experience trying to connect a secluded farmland he owns with Wi-Fi. A bunch of his friends pooled in money and bought a farmland in the middle of nowhere and now for some reason they want WiFi there. While most of us would have given up, Kiran talks about how he got an uplink from BSNL there and connected sparse houses on the farmland with fiber optic cable. As the cables aren't that long, he had to FAFO and almost deceive a lineman into [Fusion splicing](https://en.wikipedia.org/wiki/Fusion_splicing) `xD`.
- [Building Open Source Music Hardware in India - Tarab Instruments](https://fossunited.org/c/indiafoss/2026/cfp/3nr9benq94)
	- Another guy with peak FAFO energy, but this time in music industry.
	- One of his slides said along the lines of: "Poverty is the mother of invention".
	- He talked about his journey of building musical instruments he absolutely adored but couldn't afford. So he decided to build his own! He described the complete process, his prototypes, tradeoffs in the current design and where he orders the parts from.
	- Another surprising point was the margins on these things! He completes the assembly at ~3k-ish and sells them from 10k! But he seems to hate making them and hence urged the audience to build it on their own from his github repo 🙃.

Next was the "compilers, programming languages and systems" devroom after lunch! Honestly I believe they should have organized this in the morning, with topics so deep, one needs a lot more focus. This was made worse by the extra rice I was served inspite of giving a small pinch hand sign. So after a few talks: insulin went 📈, eyelids went 📉. 😭

<p align="center">
<img src="/rss-images/blogs/non-tech/2026/india-foss-2026/compilers-devroom.webp" alt="Photo of Compilers Devroom" style="width: 30%; height: 20%" />
</p>

- [Semantic Quantization in KV Cache Eviction](https://fossunited.org/c/indiafoss/2026/cfp/381kk089so)
	- This sounded super interesting as I have recently found myself getting interested by the inner working of LLMs.
	- Except that it was full of math formulas, I didn't understand a bunch of it, but the simplified graphical representations of eviction policies were easy to grasp. And the results at the end showed impressive numbers, I am motivated to understand his findings in detail.
- [Bend, don't Break. Modern Techniques for Flexibility in Strongly Typed Functional Programming](https://fossunited.org/c/indiafoss/2026/cfp/d8b9uvdntg)
	- This was a talk from FP India's Organizer [Anupam](https://hachyderm.io/@haskman@functional.cafe). Another lively man who loves FP!
	- [FPIndia](https://hasgeek.com/fpindia) is one of my favorite meetups to attend in Bengaluru, and I always look forward to Anupam's talks, naturally this was an awesome one `:)`.
- [Benchmarking can be hard, here's why we're still creating a new one for OCaml](https://fossunited.org/c/indiafoss/2026/cfp/6aasju7noq)
	- Oh boi, benchmarking is super super hard. People don't realize just how hard it is to craft fair benchmarks. It is very easy to go wrong or game them. There's a reason database industry loves dissing on each other's benchmarks, most of them are incorrectly and unfairly crafted, and even the ones which try to do a fair comparison, fail often due to loads of minute details. I still remember the PlanetScale TPCDS benchmark fiasco.
	- Well, this talk didn't invent some magic pill to fix it and mostly discussed what a good benchmark means.
- [What LLVM Does Differently When The Target Is CPU VS GPU](https://fossunited.org/c/indiafoss/2026/cfp/33ogils4n5)
	- I am not an expert on LLVM internals, except knowing just enough from attending [LLVM Social](https://www.meetup.com/bangalore-compilers-meetup-group/) regularly. I think this is a blessing in disguise cause this way I am always excited by talks around these!
	- No doubt I enjoyed this one too. cuda-oxide's recent announcement has got me super excited to mess around with Rust and GPU on my linux system!
- [PythonBPF - now with loops, custom funcs, and more maps!](https://fossunited.org/c/indiafoss/2026/cfp/299lpkmuqh)
	- [eBPF](https://ebpf.io/) is too close to my heart, I have been doing local propaganda of it ever since it's inception, restricted but arbitrary code running inside kernel, I am in!
	- While I have not played with PythonBPF personally ( I am more of a [Rust](https://github.com/feniljain/ebpf_http_trace-rs) [guy](https://github.com/feniljain/ebpf_http_trace_loader-rs) ), it was inspiring to see two students maintaining such an important library independently.
	- Also their snake game demo with state in kernel was cool!

I had to escape the devroom after these talks cause I was embarrassed of nodding off. To refresh myself I headed straight to [Failures Mode Group](https://failuremodes.dev/) gathering. This is an informal meetup which happens at IndiaFOSS every year, where we mostly hang out and talk about our self-hosting setups. While this is not my area to shine (soon I will get a VPS to get started!), [Yash](https://yashgarg.dev/) just [bulldozered](https://yashgarg.dev/homelab/) over everyone's setup. Buddy was collecting aura points and definitely won over some fans on that day `xD`.

We went around some booths and called it a day with a plan to visit a nearby cafe with whole herd of random mismatched groups: FP India + Rust India + BangPypers + PyCon India organizers. It was fun catching up with everyone and promising to meet next year again 😭.

# Second day

<p align="center">
<img src="/rss-images/blogs/non-tech/2026/india-foss-2026/booth-2.webp" alt="Plotter machine" style="width: 30%; height: 20%" />
</p>

Second day we got a bit late to arrive at the venue. We were afraid we would miss the breakfast, but were luckily just in time. Though we did end up getting late for the talks cause of an amazing catch up with [Vishal](https://www.linkedin.com/in/fossdot/). I had last talked with him two years ago I believe and had no idea he was doing such amazing [philanthropy work](https://bodhya.net/) in Bihar. He mentioned how he has also worked for FREE for long periods of time just so that he can help government better implement the scheme effectively on the ground. Sadly, government is not interested to improve their systems, they are better sitting in their AC rooms doing nothing `:(`. He is currently focusing most of his time in educating kids in a rural village. All the best dude, I hope all your work for this good cause comes to fruition!

After this uplifting conversation, we headed straight to the security devroom!

- [Reverse Engineering the PAN QR Code](https://fossunited.org/c/indiafoss/2026/cfp/a45o8u74jv)
	- Another Indian government expose session, author basically disproved India government claims of PAN card being encrypted and certified.
	- Author disproved the fact by building an open source scanner of the rather closed source government approved module. Except for the incompetence part, I found the reverse engineering journey super interesting!
	- I also found it rather uplifting that author started his journey after attending Nemo's talk last year on building an Open Source UPI app! [Captain](https://captnemo.in/) keeps on inspiring the next generation 🫡
- [When NO_PROXY Lies: Finding an IPv4-Mapped IPv6 Patch Bypass in Axios](https://fossunited.org/c/indiafoss/2026/cfp/cmc3ntipr8)
	- I am a bit too surprised to find the root cause of this vulnerability, why tf are networking related libraries still parsing and comparing IP addresses using string comparisons!!???
	- Can't even call it javascript ecosystem slop cause the same vulnerability was found in [coturn](https://github.com/coturn/coturn) too 🫠.
 - [Extracting AES Keys from Embedded Devices Using Open Source Hardware](https://fossunited.org/c/indiafoss/2026/cfp/erphckugi7)
	 - Anything power analysis is a treat for my brain by default
 - [The Art of Userspace Sandboxing on Linux](https://fossunited.org/c/indiafoss/2026/cfp/9611q7a3nk)
	 - This was the only talk in the security devroom with no live demos. While others loathed the sudden shift in atmosphere, I felt even more at home!
	 - I am more used to the talks where author does not give a shit about making eye candy slides cause he was busy doing the real work! I personally try to always go for more eye candy and memes, etc but I am definitely not averse to such presentations `:)`.
	 - This was a good overview of the sandboxing landscape and even scraped some internal details at various points.
- [Keeping an Eye on your Eye: Disassembling and Reassembling Smart Camera Hardware to breach YOUR Security](https://fossunited.org/c/indiafoss/2026/cfp/edu2vnri2u)
	- This talk was an absolute chaos, both of the speakers were visibly very excited to deliver the talk and hence decided to do live desoldering on the stage 😭. Buddy was inhaling lead fumes without protection 💀. I even texted on an internal attendee group applauding their guts to do a live demo of this considering people have started showing recordings even of software demos cause demo gods aren't happy most of the times! Well, even for them, demo gods weren't in the mood on that day and crashed the laptop 😂.
	- All the chaos aside, they were blessed with some overtime and showed a glimpse of what they were going to present further on the firmware reverse engineering part. I found that really interesting and I still wished they could have just spent all their time in that section rather than trying the desoldering stunt `:(`.
- [Hacking India's Largest Exam System](https://fossunited.org/c/indiafoss/2026/cfp/2pgiaujhvm)
	- A BANGER talk from a 19 year old who just happened to know a few things about computers and put them to good use! A typical most wanted hacker origin story `xD`.
	- Even if you're not chronically online you would know about whole CBSC and CJP fiasco. This was the start of it all. Again, saddest part is government not admitting they have messed up big time and the absolute peak of skill issue by rebuilding the website multiple times and still getting hacked by the most basic exploits!

After the security devroom, we decided to attend this panel discussion around [future of FOSS, SWE and Technical Education](https://fossunited.org/c/indiafoss/2026/cfp/6pb727vn12). Of course it was all going to be about LLMs. LLMs this, LLMs that, it was SO depressing! I had forgotten why I had stopped opening threads discussing LLMs on hackernews and lobsters. I am happy in my own little bubble, please don't try to pressure burst it till actual economic collapse happens. All the doomers read [this](https://bcantrill.dtrace.org/2026/09/27/fools-expertise/).

In the second half, we went booth hopping, this time we had some amazing talk with Dev from Gooey.ai team, Anoop from Bruno, Raghav and Arun! After yapping all around, we headed for the [Communi-Con](https://fossunited.org/c/indiafoss/2026communi-con). This was mini conf with lightning talks which community voted on! We enjoyed talks discussing postmarketOS, smartphone self-hosting, elixir, modern javascript bloat hate, improvements in the same conference, etc.! All of this was made even more fun by an accompanying website which had live voting `:P` .

Another talk I really enjoyed was discussing WiFi setup of this conference, this was hands down the best WiFi setup I have seen in any conference I have ever attended. They had setup `1.X` auth with loads and loads of routers and switches to ensure whole venue (even toilets, I checked 😂) had amazing connection, hats off to the team!

As an after conference ritual, I again ended up hanging out with FP and Rust India folks accompanied by [Harsh](https://msfjarvis.dev/), [Yash](https://yashgarg.dev/) and [Sohom](https://signalshore.net/).

# Booths

<p align="center">
<img src="/rss-images/blogs/non-tech/2026/india-foss-2026/kiran-booth.webp" alt="Kiran's booth with interesting equipment" style="width: 30%; height: 20%" />
</p>

This was the first time I went around talking to people on booths, that's how I ended up talking to Anoop and Dev `xD`. But having done a setup 2 years ago for Rust India, I know it is really tiring! I didn't get to enjoy the conference and just hated the whole experience. This is part of the reason we don't organize Rust India booths anymore `:p`. To respect their time, I went around with Yash and they were all so lovely people! Booths have definitely evolved over the last year, now we have games, a LOT MORE merch, etc.
- `Data For India` had a nice guessing game with coins, stickers and highlighters to guess different domain statistics of India!
- There were students from school, who I instantly ran over to encourage! I had a good time interacting with them but also instantly felt like an uncle, I still remember how different adults used to visit student setup booths in science fair 😭.
- There was a booth for FDroid too! Harsh was too shy to grab some stickers from the booth, so I had sneak some out `xD`.
- There were booths of all the major linux distributions! Debian, Fedora, OpenSuse, Redhat, etc etc. I was surprised realizing all of them had an active community in Bengaluru!
- There was all the booth of [tangled](https://tangled.dev/). After watching their Communi-Con talk and exploring it a bit more, I am suddenly super bullish on them. They even support jujutsu out of the box!!
- I also loved the idea of Hardware Showcase room, there were so many interesting projects in there. Someone had built an arcade machine at home 😮.
- Kiran from the fiber talk above also had a booth, Yash spent A LOT of time and I really wished I had too, look at all the cool gadgets he had. Btw this was like 5% of the total things he head, buddy was packing SO MUCH MORE!
- Another club I didn't know about was `Creative club`, they meet regularly and collectively build and discuss their creative coding endeavours!
- There was an OSM booth, always a big supporter of theirs 🫡.

## Amazing side items I liked:

There were some details I liked want to list without discussing:

- Both days had different set of booths, this allows everyone to enjoy conference and give participants a chance to enjoy more diversity!
- Absolutely loved the idea of Communi-con, please please continue it!
- 8+ devroom tracks this time, this is up from 2-3 last year. Absolute gold decision.
- They decided to change the menu for second day's breakfast and lunch. Last year it was the same on both days.
- Kept stumbling randomly into IndieWebClub folk.
- Devrooms are handled by community, FOSS United team can focus on providing good framework, while actual domain experts do the amazing talk curation, recipe for amazing success!
- Good T Shirt design.
- Tiered ticketing system which allows earning individuals to help discount student tickets `:)`.

## Looking forward:

<p align="center">
<img src="/rss-images/blogs/non-tech/2026/india-foss-2026/booth-1.webp" alt="Laptop screen showing punch code loom" style="width: 30%; height: 20%" />
</p>

Few things I am looking forward to!

- In Shree's lightning talk he mentioned some points which got me super excited for the next edition!
	- FOSSDEM style conference: OH MY GOD, this is the holy gold standard for me
	- 20+ devrooms: YES YES, you get the idea
	- Moving out of Nimhans as a venue: We need a good university with lots of decently sized classrooms
	- Main track getting shrinked: Main hall was empty most of the time, convert the general track in a devroom and allot it a smaller space
	- Even cheaper tickets for students and diversity scholarships: Respect++
- Less LLM slop talks. Devroom organizers did an awesome job curating high quality talks, but some bad apples still slipped through the cracks `:(`.

## Things to improve personally:

I would like to do a few things better next year:

- I want to network with a LOT MORE strangers.
- Check schedule thoroughly, only after the conference I realized I missed a talk from a college junior of mine 😓.
- I want to interact with the speakers whose talks I liked. This time I missed interacting with Kiran and Raunak.
- I want to visit every single booth next year, I somehow missed `Diagram Chasing` and `Bengawalk` booths, thanks to [Adithya](https://adithyanair.com/blog/indiafoss-2026/) for pointing they even existed!

All of this has already got me super excited for IndiaFOSS 2027! And finally, Yash, me and Harsh at IndiaFOSS. ( Nice try [Tanvi](https://tanvibhakta.in/) . I am sure you will get the horns right in 2027 😂 )

<p align="center">
<img src="/rss-images/blogs/non-tech/2026/india-foss-2026/us-in-indiafoss.webp" alt="Us in IndiaFOSS" style="width: 30%; height: 20%" />
</p>

P.S. Thanks to Yash for almost all the photos, sorry no sorry 🙈.