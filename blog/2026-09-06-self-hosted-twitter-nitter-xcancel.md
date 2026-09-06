---
layout: post.njk
title: "Self-Hosted Twitter: Running Your Own Nitter Instance After the Takedowns"
date: 2026-09-06
description: "Every public Nitter instance keeps getting killed by X Corp. Here's how to run your own read-only Twitter frontend on your homelab — the Docker Compose stack, the guest-account gotchas, and the honest legal caveat."
tags: ["nitter", "xcancel", "twitter", "x", "privacy", "self-hosted", "homelab", "docker", "rss", "de-google", "read-only", "frontend"]
author: "Bryan Moon"
canonical: "https://devhandbook.io/blog/2026-09-06-self-hosted-twitter-nitter-xcancel"
affiliate: true
cta: true
---

On August 24, 2026, X Corp. sent cease-and-desist letters demanding the permanent takedown of Nitter — the open-source, privacy-first Twitter frontend — along with its public instances. The repository is now archived. The public instances are dropping one by one.

And yet, in the weeks since, the number of *working* Nitter instances has actually gone *up*. A HN thread titled "Nitter has more working instances than before the takedowns" hit 654 points. The takedown didn't kill Nitter — it scattered it.

That's the whole story in miniature: **you can't kill a self-hostable tool by taking down the public instances.** You can only push people to run their own. This post is about doing exactly that — running your own read-only Twitter frontend on your homelab, so it can't be taken down from under you.

I'll cover Nitter vs XCancel (they're not the same thing), the Docker Compose stack, the guest-account and session gotchas that actually determine whether it works, and the legal caveat nobody wants to state plainly.

## Nitter vs XCancel: They're Not the Same Thing

Before the config, clear up a confusion that's everywhere right now.

**Nitter** is *software you run*. It's a free, open-source (AGPLv3) alternative Twitter frontend written in Nim. It proxies requests to Twitter's unofficial API, strips the JavaScript and ads, and serves a clean, fast, read-only view of tweets, profiles, and search. It also generates RSS feeds for any account — which is quietly the killer feature for homelabbers. You point it at your own domain, and *you* are the instance.

**XCancel** is a *public instance* — a hosted service at xcancel.com that runs Nitter (or a fork of it) for you. It's not software you can self-host; it's a website someone else operates. It's useful as a fallback, but it has the exact same single point of failure as every other public instance: it can be taken down, rate-limited, or C&D'd at any moment.

The distinction matters because the whole point of self-hosting is removing that single point of failure. XCancel is a bookmark. Nitter is a thing you own.

There's also **Nitterium**, a newer Android app (Kotlin/Jetpack Compose) that wraps Nitter instances in a native UI with subscriptions and feed groups. It's a nice companion if you want a phone-native experience, but it still depends on *some* Nitter instance being alive — ideally yours.

## The Honest Legal Caveat (Read This First)

I'm not a lawyer, and this isn't legal advice. But you should understand what you're doing before you do it.

Nitter works by scraping Twitter's unofficial API. That API is not public, not documented, and not licensed for third-party use. X Corp. has demonstrated — repeatedly, and now with formal C&D letters — that it considers Nitter a violation of its terms. Running your own instance doesn't make it legal; it makes it *harder to find and shut down*.

The practical reality: a single-user instance on your own domain, serving your own RSS feeds, is a very different risk profile from a public instance serving thousands of strangers. But it's not zero. If you're uncomfortable with that, the honest answer is to not run Nitter — use XCancel or just read Twitter in a logged-out browser.

For the rest of this post, I'll assume you've made your peace with the gray area and want the technical how-to.

## The Stack: Nitter + Redis in Docker

Nitter needs two containers: the Nitter app itself and a Redis instance for caching. Here's the working Compose stack, adapted from the (now-archived) upstream `compose.yml`.

```yaml
# docker-compose.yml
services:
  nitter:
    image: zedeus/nitter:latest
    container_name: nitter
    ports:
      - "127.0.0.1:8080:8080"   # bind to localhost; front it with a reverse proxy
    volumes:
      - ./nitter.conf:/src/nitter.conf:ro
      - ./sessions.jsonl:/src/sessions.jsonl:ro   # optional, see below
    depends_on:
      - nitter-redis
    restart: unless-stopped
    healthcheck:
      test: wget -nv --tries=1 --spider http://127.0.0.1:8080/Jack/status/20 || exit 1
      interval: 30s
      timeout: 5s
      retries: 2
    user: "998:998"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  nitter-redis:
    image: redis:6-alpine
    container_name: nitter-redis
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - nitter-redis:/data
    restart: unless-stopped
    healthcheck:
      test: redis-cli ping
      interval: 30s
      timeout: 5s
      retries: 2
    user: "999:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

volumes:
  nitter-redis:
```

Two things worth calling out:

1. **Bind to `127.0.0.1`, not `0.0.0.0`.** You do *not* want your Nitter instance exposed directly to the internet. Front it with your reverse proxy (Traefik, Caddy, Nginx) and put it behind your existing auth or at least a firewall. A public Nitter instance is a liability; a private one is a tool.

2. **`read_only: true` + `cap_drop: ALL`** are the upstream defaults and worth keeping. Nitter is a scraper that talks to an untrusted third-party API; you want it as locked down as possible.

## The Config File

Nitter reads a single `nitter.conf`. Here's the minimal version that works, with the two fields you must change:

```ini
[Server]
hostname = "nitter.yourdomain.com"   # used for generating links
title = "nitter"
address = "0.0.0.0"
port = 8080
https = false
httpMaxConnections = 100
staticDir = "./public"

[Cache]
listMinutes = 240
rssMinutes = 10
redisHost = "nitter-redis"          # the container name from compose
redisPort = 6379
redisPassword = ""
redisConnections = 20
redisMaxConnections = 30

[Config]
hmacKey = "CHANGE_ME"               # openssl rand -hex 32
base64Media = false
enableRSS = true
enableRSSUserTweets = true
enableRSSUserReplies = true
enableRSSUserMedia = true
enableRSSUserArticles = true
enableRSSSearch = true
enableRSSList = true
enableDebug = false
proxy = ""
proxyAuth = ""
apiProxy = ""
disableTid = false
maxConcurrentReqs = 2
maxRetries = 1
retryDelayMs = 150

[Preferences]
theme = "Nitter"
replaceTwitter = "nitter.net"
replaceYouTube = "piped.video"
replaceReddit = "teddit.net"
proxyVideos = true
hlsPlayback = false
infiniteScroll = false
```

The two fields that matter:

- **`hmacKey`** — generate a unique value with `openssl rand -hex 32`. It signs media URLs. Leaving the default `secretkey` is a real (if minor) security hole.
- **`redisHost`** — must be `nitter-redis` (the container name), not `localhost`. This is the #1 "it's not working" cause: Nitter starts, but every request 500s because it can't reach Redis.

## The Part That Actually Determines Whether It Works: Sessions

Here's the thing every "just run Nitter" tutorial glosses over: **Nitter needs Twitter credentials to function reliably.**

Nitter can operate in two modes:

1. **Guest mode** — no credentials, using Twitter's guest tokens. This works for basic browsing but is heavily rate-limited and increasingly unreliable as X tightens its anti-bot defenses.

2. **Session mode** — you provide real Twitter account cookies (an `auth_token` + `ct0`), and Nitter uses them to make authenticated requests. This is dramatically more reliable, but it means you need a Twitter account you're willing to burn, and that account can get banned for automated access.

The upstream repo ships a script to generate sessions: `tools/create_session_browser.py`. It drives a real headless browser (via `zendriver`) through the login flow to capture the cookies, because — and this is the key detail — **X now requires a "castle token" generated by client-side JavaScript during login, which blocks the old pure-API approach.** The older `create_session_curl.py` is explicitly marked deprecated and will fail.

The practical workflow:

```bash
# One-time: generate a session for a throwaway account
pip install -r tools/requirements.txt
python3 tools/create_session_browser.py myusername mypassword TOTP_SEED --append sessions.jsonl
```

The output is a JSON line with `auth_token` and `ct0`, which Nitter reads from `sessions.jsonl`. Mount that file into the container (as in the Compose above) and Nitter will use it.

**The honest tradeoff:** session mode is more reliable but risks the account. Guest mode is safer but flakier. For a personal instance serving your own RSS feeds, guest mode is often "good enough" — you're not hammering it with thousands of requests. Start with guest mode, and only bother with sessions if you hit rate limits.

## The RSS Angle (Why Homelabbers Actually Want This)

The feature that makes Nitter worth self-hosting isn't the pretty frontend — it's **RSS**.

Nitter generates RSS feeds for any account, search query, or list:

- `https://nitter.yourdomain.com/@username/rss` — a user's tweets
- `https://nitter.yourdomain.com/@username/with_replies/rss` — tweets + replies
- `https://nitter.yourdomain.com/search/rss?q=keyword` — search results

This means you can follow Twitter accounts in your existing RSS reader (Miniflux, FreshRSS, TT-RSS — all of which I've covered on this site) without ever opening Twitter, without an account, and without the algorithm. For a homelabber who already runs an RSS reader, Nitter slots in as "the Twitter bridge."

This is also why the takedowns matter so much to this audience: public Nitter instances' RSS endpoints die with the instance. Your own instance's RSS endpoints die only when *you* stop running it.

## The Gotchas (What Actually Breaks)

Three things that will bite you, in order of how often they bite:

1. **Redis hostname.** As above — `redisHost = "nitter-redis"` in the config, not `localhost`. The container can't see `localhost`; that's the host's loopback, not the Redis container's.

2. **Rate limiting without sessions.** Guest mode works until it doesn't. If you see empty timelines or 429s, that's Twitter throttling guest tokens. The fix is sessions (with the risk that entails) or just accepting that a personal instance gets throttled less than a public one.

3. **The `hostname` field.** If you leave it as `nitter.net`, every link Nitter generates points at the (dead) public instance. Set it to your own domain before you go live, or you'll click a link and land on someone else's broken instance.

## The Bottom Line

The Nitter takedowns were supposed to be the end of the story. Instead they're the clearest possible advertisement for the self-hosting thesis this site has been making all year: **anything you can run yourself, you can't be cut off from.**

Nitter is a read-only Twitter frontend that gives you clean browsing and — more importantly — RSS feeds for any account, on your own domain, under your own control. It's a gray area legally, it needs a throwaway account to be fully reliable, and it's not a drop-in replacement for actually using Twitter. But for the homelabber who wants to follow a handful of accounts without the app, the tracking, or the algorithm, it's the right tool.

If you're already running an RSS reader, this pairs naturally with [self-hosted RSS (Miniflux/FreshRSS/TT-RSS)](/blog/2026-08-28-self-hosted-rss-reader-miniflux-freshrss-ttrss/). If you're on the broader privacy kick, it slots in next to [de-Googling your Android](/blog/2026-09-02-degoogling-android-grapheneos-self-hosted-stack/) and [self-hosting Firefox Sync](/blog/2026-09-02-self-hosting-firefox-sync/). Own your stack, one service at a time.

*Sources: the [Nitter repository](https://github.com/zedeus/nitter) (archived Aug 24, 2026 following X Corp. cease-and-desist), the [Nitterium](https://github.com/kaleedtc/Nitterium) Android wrapper, and the HN discussion "Nitter has more working instances than before the takedowns."*
