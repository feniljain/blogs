## Optimizing My Shell Startup Times

I was going through Thorsten's latest blog about faster startup times, and he talks about shell startup times, here is the direct link for curious: https://registerspill.thorstenball.com/p/how-fast-is-your-shell . This made me wonder how fast is my shell config, and work on my shell load times to at least get them in a bareable range.

So to start we find how to measure our load times, well Thorsten covers it nicely with this simple one liner:

```sh
time zsh -i -c exit
```

and one data point is not reliable, so let's run it 10 times:

```sh
for i in $(seq 1 10); do time $SHELL -i -c exit; done
```

I felt, looking at 10 results wasn't as good, so here's a script which gives you average for one of the columns:

```sh
#!/bin/zsh

for i in $(seq 1 10); do
  2>&1 time $SHELL -i -c exit
done | awk '{sum += $11} END {print "Average:", sum/NR}'
```

(Note this averages on total)

Cool, this looks good, well I hoped I could say the same for my shell startup times 😬. They were in order of ~600ms, that is very bad, super heavy leaded shoes in terms of original article :(
Thankfully it also links to some amazing articles, most notably: https://htr3n.github.io/2018/07/faster-zsh/. This is an amazing article which I referred through whole of my process, I did not end up implementing all the tricks because I wanted to keep all my config in .zshrc and also not much long.

First tip is about profiling zsh, and that is done using zprof, we can enable it by pasting following line in .zshrc:

```sh
zmodload zsh/zprof
```

or running `zmodload zprof` directly in shell (note the difference of `zsh/`). This gives us places where zsh spends most time in, and hence our data points to start optimizing. But before diving straight into checking profiling output, I decided to checkout few low hanging/no-brainer things I can sort myself out. This was easier to do considering I don't maintiain my .zshrc as avidly.

To start, I had to figure out what all config files does zsh even consider, thankfully htr3n's article covers sequences of config files loaded and hence also listing them. So I started checking those files and found out unused nix, orbstack, etc. exports. First low hanging fruits spotted! Next moved onto .zshrc and cleaned up unused/old/irrelevant exports.

Few of the time taking processes are, `eval` calculation and `subshell` spawning, if you are a mac user and have something like: `brew --prefix` in your config, that is an indication of subshell spawning. To optimize this, we can run given command manually and then paste the output in place of subshell spawning code. One thing to note is, this is a double-edge sword, as any changes in install locations from brew in future would cause breakages for us, so do think about it before applying. Well I applied them and this instantly causes my startup time to reduce by half i.e. we reach 300ms territory.

For our next target, article mentions version managers like rvm and nvm are super heavy and contribute a lot to startup times. That's easy, just stop using them? Right? Nope, we introduce some indirection. The godly trick which solves and breaks almost everything in computers. In this case, we define an additional function with same name as version manager, let's say `nvm` and move all the nvm required load code in there, and finally call `nvm` at last. In my case it ended up looking like this:

```sh
function nvm() {
    export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
    if [[ -e ~/.nvm/alias/default ]]; then
      PATH="${PATH}:${HOME}.nvm/versions/node/$(cat ~/.nvm/alias/default)/bin"
    fi
    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

    # invoke the real nvm function now
    nvm "$@"
}
```

What this results in is a neat trick, where whenever we call nvm for the first time, this function gets called and hence things gets loaded for the first time. So it does not block and increase load times on startup. This reduced my startup times to ~100ms!

Some amazing progress for small and straightforward changes.

Now for another small win, we disable auto-updates using:

```sh
DISABLE_AUTO_UPDATE="true"
```

This brought us to ~80ms. Next I removed `battery` and `git` plugins, this shaved off 10ms more, so ~70ms. This is also a good time to talk about those fancy shells, if you someone who has a fancy shell which shows git version, nvm version, battery level, neighbour's mom's number, then you will suffer from relatively higher load times always, cause of the many computations shell needs to do in background to fill up those fancy prompts. I personally moved off them much before and now use simple `robbyrussell` theme, which is super minimal and does not get in my way or try to occupy more space than necessary.

After this, according to zprof most of zsh's time is spent on autocompletions, and in turn compinit, etc. components, there are some really nice tricks to optimize these components further, for e.g. compinit tries to read a cache files everytime it is invoked, people have observered that's not necessary and can be reduced to once per day, this gave some more small wins. After a while there comes a plateau in optimizing, where you are in a land in which moving the needle to your favour becomes increasingly harder and harder, this is that territory. Well maybe that's not quite true, cause one of the big optimizations which can be done is removing oh-my-zsh, I tried that and it caused my load times to go down to ~20ms. The thing is I don't want to move off omz yet. I tried to find some alternatives but wasn't as happy, honestly didn't spend much time looking too, so if I do find a better solution will update this article! Just as a side note one of the resource I am planning to check out was this gist: https://gist.github.com/laggardkernel/4a4c4986ccdcaf47b91e8227f9868ded, it was linked in comments of same article (aint it a gold mine xD).

Another small thing which can help in speeding up very first shell load (i.e. after a fresh boot), is to remove ASL logs, these are apple logs which on some systems can grow to be huge. One can prune them easily using:

```sh
cd /private/var/log/asl/
ls *.asl
sudo rm !$
```

Read more about its benefits/losses in this article: https://osxdaily.com/2010/05/06/speed-up-a-slow-terminal-by-clearing-log-files/

I tried a few more things, like replacing hack mentioned in this article: https://coderwall.com/p/sladaq/faster-zsh-in-large-git-repository, to reduce git branch computation time, tho it wasn't that bad for me, so I skipped this one also especially because this required changing a omz lib file. Well at last this was the place where I stopped trying to do any more complex more code optimizations and rested my case for this time. (we will pick it up again for sure!)

If you would like to go through all of my changes together, this is the commit: https://github.com/feniljain/dotfiles/commit/202e22baee2164e4c38e18eb88db7a1b920f84c6#diff-d30bab601f4597c635d0bd4915f3475c4c22170a538d6781cd086bdfe100961fL5
