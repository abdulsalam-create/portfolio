# Break It, Then Ask Why: Exposing a Hidden Internal Dev Console With One Flipped Flag

Honestly, this one was kind of an accident.

I'd been on this travel site for a couple of hours (the big booking one) and had basically nothing to show for it. Ran through the usual stuff on the endpoints I could find and none of it went anywhere. When I get to that point I tend to stop looking for a specific bug and just start breaking things to see how the app reacts, and that's how I ended up here.

Quick note, I'm redacting the program, the hostnames and the real service names. It got fixed, so I'd rather keep it anonymous.

One thing I always keep half an eye on is the booleans in responses. They're everywhere once you look, `debug:false`, `isInternal:false`, feature flags, all that. And it kind of nags at me, because the server already made the decision, so why is it bothering to send it to me? And if it's in my browser then it's mine to mess with.

So I tested that in the laziest way possible. Opened Burp, set up a Match and Replace on the response body, swap `false` for `true`. Not one specific flag, just all of them at once.

```
Type:    Response body
Match:   false
Replace: true
```

It's a sledgehammer, I know. But I wasn't going for precision, I just wanted to see what the page would do if none of its "no" answers made it through to the browser. Turned the rule on, refreshed, and waited for something to fall over.

Nothing fell over. Instead a little developer panel slid up from the bottom of the page. Took me a second to even clock what I was looking at.

So then the real question was why that worked at all.

When I dug into it, the console's visibility was tied to a few validation flags that in production were set to `false`. Fine so far. The problem is where that check was happening. The server was sending the flags down and letting the frontend decide whether to render the tool. So the one thing keeping an internal console hidden was a `false` sitting in a response I was already holding in Burp. Flip it before the JavaScript reads it and the whole thing just renders. I didn't really bypass anything, the code was already shipped to everyone's browser, that flag was just the thing standing in front of it.

It's the same shape as a disabled button you can re-enable in devtools, or an `isPremium` check that only lives in JavaScript. If the browser gets to decide, it was never actually decided.

And the panel wasn't some empty stub either. It had a row of tabs, and one of them, "Service Overrides," listed internal services by name, each with an editable host field and a port next to it:

```
Service         Host           Port
ad-delivery     add override   ····/http
ad-selection    add override   ····/http
ab-testing      add override   ····/https
app-config      add override   ····/https
...and a dozen or so more
```

(Names generalised, ports stripped. On the live site these were the real ones.)

So a public homepage was basically handing out a map of the backend. Internal service names, ports, protocols, and little override hooks for repointing a service somewhere else. There were a few other tabs too, observability, GraphQL, audits, the kind of thing that's clearly only ever meant to be seen by engineers.

By itself it's information disclosure, and that's how it got triaged, P4, accepted, small payout. Nothing popped directly. But for recon this stuff is gold. Normally you're guessing at how the backend is wired together, and here it was just written out for me, ports and all. Those host-override fields especially had me wanting to go spend a long afternoon looking for SSRF.

What I actually take away from it is less about the bug and more about how it turned up. I wasn't being smart, I was just bored of the checklist and started breaking things, and then when something unexpected happened I didn't wave it off. That last bit is the part people skip. The app doing something weird is usually it telling you where its assumptions are thin. Most of the time you undo it and carry on. Every now and then you sit with it and it turns into a report.

And if you're on the other side of this and fixing it, the short version is don't ship debug tooling to production, and if you absolutely have to, enforce it on the server where the user can't get at the switch.
