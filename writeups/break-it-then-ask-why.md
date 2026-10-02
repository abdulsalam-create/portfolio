# Break It, Then Ask Why: Exposing a Hidden Internal Dev Console With One Flipped Flag

I wasn't looking for anything clever. I'd been poking at this travel site for a while, the big booking kind, and I'd hit the point in a session where all the obvious stuff is gone and you're just sitting there staring. No injection, no broken access control on the endpoints I'd tried, nothing. So I did the thing I do when I run out of ideas: I stopped trying to find a bug and started trying to break the site on purpose, just to see how it would react.

That's honestly the whole trick. Most of my half-decent findings don't come off a payload list. They come from messing with something the app assumes will always be a certain way, and then paying attention when it flinches.

(The program is redacted here, along with the hostnames and the real service names. It's been fixed, so I'm keeping it anonymous.)

## The thing I kept noticing

Responses are full of little booleans. `enabled: false`. `debug: false`. `isInternal: false`. Feature flags, mostly. And every one of them is the server making a decision and then shipping that decision down to my browser to hold onto.

That always makes me a bit suspicious. If the server already decided, why is it telling me? And if it's telling me, what happens when I just change my mind for it?

## So I flipped all of them

I didn't do anything surgical here. I opened Burp, added a Match and Replace rule on the response body, and told it to swap `false` for `true`. Every `false`. The whole response.

```
Type:    Response body
Match:   false
Replace: true
```

Yeah, it's a sledgehammer. That's kind of the point. I wasn't after one specific flag, I just wanted to see what the page does when none of its "no" answers survive the trip to the browser. Turned the rule on, reloaded, watched.

It didn't throw an error. A little developer panel slid up from the bottom of the page instead.

That was not supposed to be there.

## Okay, so why did that work

This is the part I actually care about, because the "why" is the real bug. The panel showing up is just the symptom.

Turns out the console's visibility was wired to a few validation flags, and in production they were all set to `false`. Reasonable enough on paper. The problem is where that `false` was getting checked. The server sent the flags down to the browser, and the browser was the thing deciding whether to show the tool. So the entire security boundary was a word in a response body that I was holding.

When I flipped it to `true` before the page's JavaScript ever read it, I didn't bypass a control. I answered a question the server had handed to me, and I answered it the way that happened to suit me. The code to render the console was already sitting in every visitor's browser. The flag was the only lock on it, and it was a lock with the key taped to the front.

Same mistake shows up all over the place once you start seeing it. A hidden admin button. A greyed-out field the backend happily accepts anyway. An `isPremium` check that lives in JavaScript. If the browser gets to decide, it was never really decided.

## What was actually in there

The panel wasn't a toy. It had a row of tabs, and one of them, "Service Overrides," listed internal services by name, each with an editable host field and a port:

```
Service         Host           Port
ad-delivery     add override   ····/http
ad-selection    add override   ····/http
ab-testing      add override   ····/https
app-config      add override   ····/https
...and about a dozen more
```

(Generalised, and I've stripped the real ports. On the live site these were the actual internal names.)

So a public page was quietly handing any visitor a diagram of the backend. Service names, ports, protocols, and little "override" hooks for pointing a service somewhere else. There were other tabs too, observability, GraphQL, audits, the kind of thing that's obviously built for engineers and obviously never meant to leave the building.

## Why it's worth caring about

On its own it's information disclosure, and that's how it got triaged. A P4, accepted, small bounty. Nothing got popped, nothing got stolen.

But this is exactly the stuff you want during recon. Normally you're guessing at how the backend fits together. Here it was, labelled, with ports attached. That quietly upgrades everything else you might try afterwards, and those host-override fields in particular are the sort of thing that makes me want to spend a very long afternoon looking for SSRF.

The part that sticks with me isn't really technical, though. Somebody shipped a genuinely powerful internal tool to production and left it sitting behind a flag that anyone could flip. That only makes sense if you assume nobody's ever going to look. People look.

## What I took from it

I didn't find this because I was being smart. I found it because I got bored of the usual checklist, started breaking things just to see what fell out, and then, instead of shrugging and hitting undo when something weird happened, I actually chased the weird thing until I understood it.

That second half is the bit people skip. When the app does something you didn't expect, that's it telling you where its assumptions are thin. Usually you tidy it up and move on. Every so often you sit with it and it turns into a report.

If you're on the other side of this and fixing it: don't ship debug tooling to production, and if you really have to, gate it on the server where the user can't reach the switch. A flag you hand to the browser is a suggestion, not a control.
