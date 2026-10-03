# I Automated the Boring Part of Bug Bounty Hunting

For a while my bug bounty routine had this dumb little ritual in it. Every few days I'd open a dozen program pages across HackerOne, Bugcrowd, YesWeHack and Intigriti and just scan down them. Did anything new launch? Did a program I like quietly add a wildcard to scope? Did something come back after being paused? New scope is the best time to look at a target, before everyone else piles in, and the only way I had of catching it was remembering to look.

I'm not great at remembering to look. So I built a thing that does it for me, every six hours, for free, and emails me when something actually changes. It's called bounty-watch, and this is roughly how it came together and the handful of decisions that made it actually work.

(repo and live dashboard links are at the bottom.)

## Rule one, don't scrape

My first instinct was to scrape the four platforms. Bad idea. Logins, rate limits, terms of service, brittle HTML that breaks the week after you ship it. I didn't want a project whose entire job was to quietly rot.

Turns out I didn't have to. There's a public dataset, arkadiyt/bounty-targets-data, that already aggregates the public program and scope info for the major platforms and refreshes itself every half hour or so. So instead of hammering the platforms, bounty-watch just reads a few JSON files out of that repo. No logins, no scraping, nothing that puts my accounts at risk. Honestly that one decision is most of why the thing is still standing.

## The core is just a diff

Strip the features away and bounty-watch is a glorified diff. Every run it pulls the current program and scope data, flattens it into a simple shape (a key like `h1:shopify`, the platform, the in-scope and out-of-scope assets), and compares that against a snapshot it saved the previous run.

A key that wasn't there before is a new program. An asset that showed up in a program's scope is an addition. One that vanished is a removal. A program that disappears completely and later reappears is a pause and then a resume, which I track separately so a resumed program doesn't get announced as brand new and send me chasing nothing.

The very first run doesn't alert on anything, it just records the baseline. I learned that one the hard way, through a single glorious flood of notifications.

## Where do you run something that has to run forever, for free

This was the part I actually enjoyed working out. I didn't want a server. Servers cost money and need babysitting. So the whole thing runs on GitHub, and it quietly leans on three GitHub features for jobs they weren't really built for.

Actions is the scheduler. A cron fires the script every six hours.

Issues is the mailer. When something worth knowing happens, the workflow opens an issue, and since GitHub emails you about issues in your own repos, I get an email for free without ever touching SMTP or paying for an email API.

Pages is the dashboard. The script writes its data as JSON back into the repo, and one static page reads that and renders the feed, the scope changes, all of it, at a public URL.

So there's no backend anywhere. The repo is the database, Actions is the cron, Issues is the email gateway, Pages is the frontend. The daily data commits have a nice bonus too: GitHub switches off cron on repos that go quiet for a couple of months, and a repo that commits fresh data every day never looks quiet.

## Then it got opinionated

Watching every program for scope changes is handy but noisy. What I really care about is a short watchlist of programs I'm actually hunting, and for those bounty-watch does more.

It checks certificate transparency logs (crt.sh) for new subdomains under wildcard scopes. That's passive, it never sends anything to the target. When a new subdomain turns up it does a quick liveness check and flags possible subdomain takeovers.

It also fingerprints the frontend of each in-scope site with a single GET: the JS bundle filenames, the Server and X-Powered-By headers, the generator tag, the title. When that fingerprint shifts, something got deployed, and a fresh build is exactly when new bugs ship. It goes a step further and mines the changed JavaScript for new API endpoints, GraphQL operations and feature flags, so a deploy turns into a little to-do list of new things to go poke at. It watches for mobile app version bumps too.

All of that becomes an alert the moment it changes, instead of me stumbling onto it two weeks later.

## Figuring out what to look at

Once you're watching a lot of programs the next problem is just "okay, which one do I actually spend tonight on." So every program gets a hunt score out of 100. It's deliberately a plain additive model, not some mystery black box. Freshness weighs the most, because recent scope is the least picked over, then how much new surface has shown up lately, then the raw attack surface (a wildcard counts for more than a single host), then reward, then roughly how saturated the program probably is. I made it additive on purpose so one weak signal can't zero out an otherwise great target, which is what kept happening back when I tried multiplying everything together.

## What I'd tell you if you built the same thing

Keep it dependency-free if you can. The whole script is standard-library Python, no pip install, which means the Action has basically nothing to break and the thing will probably still be running in two years without me touching it.

Record a baseline on the first run and alert on nothing.

And lean on free infrastructure harder than feels reasonable. I never would have guessed Issues made a perfectly good email gateway until I needed one and didn't feel like paying for it.

That's really it. It isn't clever in any single place, it's just a diff wired into free infrastructure with a few opinions bolted on. But it has quietly made me faster at the one thing that actually matters in this game, which is getting to new scope before everyone else does.

Code: [github.com/abdulsalam-create/bounty-watch](https://github.com/abdulsalam-create/bounty-watch) · live dashboard: [abdulsalam-create.github.io/bounty-watch](https://abdulsalam-create.github.io/bounty-watch/)
