---
tech: false
draft: false
slug: fifth-elephant-2026
title: Fifth Elephant 2026
publishedOn: 2026-08-01
lastEditedOn: 2026-08-01
---


I recently attended `The Fifth Elephant` [conference](https://hasgeek.com/fifthelephant/2026/) on 31st July 2026, it was an interesting experience overall, I got to learn a lot of things, and also had a fanboy moment! Read on for the whole scoop `:)`.

## Speaker to attendee

Hasgeek had reached out to my `$WORK` for submitting a talk to the conference and I had gladly volunteered! Topic was already decided, it had to be about `Composable Query Engines`. Basically covering all the rave about [datafusion](https://github.com/apache/datafusion), [arrow](https://github.com/apache/arrow/), etc. Projects of this new era which are allowing everyone to build their own versions of specialised databases with shared infrastructure easily. Historically, it has been hard to introduce proper abstractions in databases. Abstractions which respect the performance and provide extensibility at the same time followed a super delicate balance. [Andrew Lamb](https://github.com/alamb) took on this challenge and has developed a remarkable project. By the way, this is just about the query engine. Similar credit goes to arrow and other awesome projects which are following the UNIX philosophy of doing one thing and designing it much better than any of the individual implementations in disparate databases could ever reach.

I was pretty delighted to get an opportunity to share a marvellous piece of engineering with a hall full of people who were interested in similar topics! I had prepared my slides, had an editorial feedback round and also did demo runs. All this and then just a week before the conference, they inform me they had to cut down my talk from 30 minutes to 15 minutes to "accommodate more talks". This was a huge bummer and it smelled like a big mismanagement from the team who's literal business is organizing these conferences. I had a talk internally at my `$WORK` and withdrew my talk respectfully.

Later, I received a reach out apologizing for the mishap and was offered a ticket to attend anyways. While I was pondering on whether to go or not I realized, a bunch of database folks were gonna attend this conference and above all [Sunny Bains](https://x.com/sunbains) was coming to give a talk! I have been following him on twitter for a very long time. His diligence, love for the craft, depth of knowledge, everything is super inspiring to me! With that settled I was ready to attend the conference `xD`.
## Talks

My cook was late that morning and consequently I didn't make it in time at the venue. I couldn't make it to first talk of the day which I was excited about : `Optimized AI inference with llm-d and a case-study on sovereign AI on heterogenous GPU cluster`

### Agents broke your architecture, not your data model`

It wasn't that sad though because immediately after that was the keynote with Sunny! To be frank, I am not sure what I was expecting from the talk, but it was disappointing? He put heavy emphasis on elasticity of the cluster i.e. workload management and working with huge number of ephemeral tables. But all the talk about LLMs seemed a bit forced, at times I even felt he was controlling himself from shitting on them 😛. Overall, a company marketing pitch which seemed like Sunny wasn't as excited to deliver.

### Full refresh to incremental: rebuilding denormalization for reporting at Razorpay scale

This is another talk I was super excited about. I know Piyush Goel has recently joined [Razorpay](https://razorpay.com/) to take the helm of their data platform division. I have known him since `Papers We Love` days, so its exciting to see him in a place which I think he totally deserves. Getting to the content, speaker talked about over 6-8 months of their recent experience in scaling Razorpay from single digit million to triple digit million transactions scale. They grew from relying on a `MySQL` deployment to a full fledged lakehouse deployment.

They had moved from `MySQL` to [TiDB](https://www.pingcap.com/) as their first move, but `TiDB` having it's own storage format was getting too expensive to maintain. That's when they decided to introduce lakehouse and hence S3 backed data storage in the mix. They shifted to a hybrid architecture of `TiDB` for hot path and [delta](https://delta.io/) tables for slow path. One of the main usecases of this was report generation, and not analytics, so SLAs were really small, that's when they ditched delta and started looking into streaming engines. They ended trying [Pinot](https://pinot.apache.org/) and streaming joins but their joins were so big, engines weren't able to handle the load.

That's when they decided to take a step back and solve the huge JOINs problem. To circumvent the joins, they pre-joined the table and created facts table so that they don't have to rely on heavy engine deployments which failed from time to time. For supporting these super wide fact tables, they moved to [Apache Iceberg](https://iceberg.apache.org/). They claimed to have observed better results with Iceberg due to its metadata structure, which helped them in pruning out a lot of data.

Their final form was, <5 minutes report generation from `TiDB` and slightly delayed report generation from lakehouse. While later catching up with Piyush I asked him how were they not having memory problems with such wide tables. They mentioned me that they now have only 11 tables (down from 24) in the fact table and the rest are computed with joins again. This seems to have been a good tradeoff landing them at <90% of the price of previous pipelines. Win-win situation!

Another interesting point I found in their talk was how they built secondary indexes to serve some instant query workloads. I asked if they ever considered [Apache Hudi](https://hudi.apache.org/), as that has supported indexes from a long time. The answer was herd mentality `:P`. Not even kidding, the decision was literally taken cause whole Industry is moving towards using Iceberg now `:)`.

### High concurrency & low latency serving on Apache Iceberg

This was sadly another marketing talk, even though the speaker kept trying to say that "What would you do outside of [Startree](https://startree.ai/)", everything was just about them.

### Your voice agent is (probably) doomed, OR how not to fall victim to outdated voice agent playbooks

This was the most whackall talk of this whole conference. It was about building a sports companion chat avatar which is named `Googly` and is a monkey talking about cricket??? Its seeming usecase is accompanying livestreamers. Tech side, talk was utterly straight forward industry knowledge. Instead, my big takeaway from this talk was, people are super delusional, they make such dumb products and actually talk about taking it to market.

Next few talks I skipped and worked on a personal project of mine, the ones I just sat through with earphones were:

- `Sponsored talk: The practical guide to reverse-engineering XXL codebases with agentic AI`
- `An Agent That Builds Agents: AI-Powered Recipe Generation for Local-First ETL`

### Architecting Observability Platform on S3 + Lambda

This was another talk I was super excited about because something similar has been cooking up at my `$WORK` too. So these guys are basically rebuilding observability on lakehouse powered by [Tantivy](https://github.com/quickwit-oss/tantivy), Datafusion, S3 and lambda. They even claimed to have beaten [Clickhouse](https://clickhouse.com/) in a benchmark they ran! First suspicion was when they gave some rough details about benchmarking setup but nothing about the actual queries used.

I really don't know if they are serious or not cause they may see some gains on small synthetic benchmark but they seriously can't win the game when fast caching, large aggergate heavy queries with spilling and distribution get involved. This is where my point about people being delusional was proven even more, how are people getting on podium and making such bold claims which have no footing. They don't deserve a skin in this game!

### Billboards near a coffee shop: making proximity search fast in ClickHouse

Finally, last talk was about geospatial query optimization on Clickhouse, unfortunately I had to leave the talk half way to get back home for an urgent errand `:(`.

## Conversations

I had some pretty nice conversations with different database developers!

When trying to approach Sunny on their `TiDB` booth, I ended up buzzing near a group enough and I got to talk with Chandra from [Cockroach Labs](https://www.cockroachlabs.com/) and Amit Gupta from Google! They both seem to be working on workload management, Vector DB and query optimization, pretty interesting stuff!

I also got to meet someone from [DataNimbus](https://datanimbus.com/), did you know this company serves Banks as their escrow partners and is a consulting arm of [Databricks](https://www.databricks.com/)!!?? These are like two completely separate companies, but surprisingly they do this under the same roof!

I also met someone working on `DocumentDB` in Microsoft, I tried to probe him if they got into any trouble with lawsuit between [FerretDB](https://github.com/FerretDB/FerretDB) and [MongoDB](https://www.mongodb.com/) about wire protocol, but it seems everything was good there. Guess Mongo won't mess around with Microsoft `xD`.

While talking with Sunny I also spoke with `Sayyaparaju Sunil`. I had a really lively conversation with him about his experience at [Aerospike](https://aerospike.com/), variant of LSM trees they used, DPDK, O_DIRECT, etc etc. I only realized after receiving a LinkedIn request that he was `VP Of Engineering` 😭. Its good to talk to people with genuine curiosity, that's the secret to networking `:)`.

Next I met with Sunny Bains finally!! And to my absolute surprise, HE ALREADY KNEW ME 😭💃. We have followed each other on twitter for a while, but I never knew he really liked my curated posting about systems engineering blogs and was asking why hadn't I posted in a while. Truth be told, I had stopped using twitter, but I might just come back after this conversation 🥹.
## End

And yeah, this was it, amazing experience all over and excited to attend TheFifthEl next year too `:)`.