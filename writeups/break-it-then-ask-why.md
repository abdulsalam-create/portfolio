# Break It, Then Ask Why: Exposing a Hidden Internal Dev Console With One Flipped Flag

Most of my best findings did not start with a plan. They started with me breaking something, watching it react in a way it was not supposed to, and then refusing to move on until I understood *why*.

This is one of those. A single blunt change to a server response made a hidden internal developer console appear on a production website, the kind of tool that is supposed to live behind a VPN and a login, not one HTTP response away from any visitor. Here is how curiosity, not a checklist, got me there.

The target was a large travel-booking platform. I will keep it anonymous and redact the hostnames throughout.

---

## The mindset: break it, then ask why it broke

A lot of bug hunting advice is a list of payloads. Paste this into that field, try this header, run this scanner. That works, but it finds the bugs everyone else is also looking for.

The findings nobody else has usually come from a different habit: **deliberately breaking the application's assumptions and then investigating the wreckage.** Not "does this input cause an error," but "what does the app assume is always true, and what happens if I make it false? Or true?"

When something breaks, most people revert and move on. The bug is in the next step: *why* did it break? What did that reaction just tell me about how the system is built? That question is where the interesting stuff lives.

## The setup: poking at what the page trusts

I was looking at the site's responses, not hunting any specific vulnerability class. One thing I pay attention to is **boolean flags in response bodies.** Modern frontends ship a lot of their own configuration: feature toggles, `enabled: false`, `isInternal: false`, `debug: false`. Every one of those is a decision the server made and then *told the browser about*, which means the browser is now holding a decision it could be tricked into re-deciding.

So I asked the dumb, useful question: what if those flags are not just information, but the actual gate? What if the browser is the thing enforcing them?

## The break: flip every "false" to "true"

The fastest way to test that assumption is also the most blunt. In Burp Suite I set up a **Match and Replace** rule on the **response body**:

```
Type:    Response body
Match:   false
Replace: true
```

That is not surgical. It flips *every* `false` in every response to `true`. It is a wrecking ball, and that is the point. I was not trying to exploit one specific flag, I was trying to see what the application does when its own assumptions stop holding. Reload the page and watch what falls over.

What fell over was not an error. **A floating developer panel appeared at the bottom of the page.**

## Why it broke: the server was trusting the browser

This is the part worth slowing down on, because the "why" is the actual vulnerability.

The dev console was never meant to be visible in production. Its visibility was controlled by a set of validation flags, and in production those flags were set to `false`. So far, so reasonable.

The problem: **those flags were sent to the browser and then checked in the browser.** The server's logic was effectively "here is a flag that says you are not allowed to see this, please enforce that yourself." That is not access control. That is a polite request, and an attacker is under no obligation to honour it.

By flipping the flag to `true` in the response, before the page's JavaScript ever read it, I was not bypassing a security control. I was answering a question the server had outsourced to me, and I answered it in my favour. The code that renders the console was already shipped to every visitor. The only thing standing between a random user and an internal tool was a word in a response body that the user fully controls.

That is the root cause, and it is a pattern, not a one-off: **client-side enforcement of a server-side decision.** The same shape shows up as hidden admin buttons, `isPremium` feature gates, and "disabled" form fields that the server happily accepts anyway.

## What fell out: an internal service console

The panel was not cosmetic. It had tabs, and one of them was **Service Overrides**. It listed internal services by name, each with an editable **host override** field and a **port**:

```
Service           Host            Port
ad-delivery       add override    xxxx/http
ad-selection      add override    xxxx/http
ab-testing        add override    xxxx/https
app-config        add override    xxxx/https
...and a dozen more
```

(I have generalised the names and stripped the real ports. On the live target these were the platform's actual internal service names and port assignments.)

In other words, a production page was handing any visitor a map of the backend: internal service names, which ports they speak on, which protocol each uses, and hooks to override where those services point. There were other tabs too, for observability, GraphQL, consent, and audits, all internal tooling that assumed it was only ever shown to engineers.

## Why this matters

On its own this is an information-disclosure bug, and it was triaged as one (a P4, accepted, with a small bounty). No data was read, nothing was taken over. So why care?

Because **this is reconnaissance gold.** An attacker fingerprinting this platform usually has to guess at the backend. Here it is, labelled: the names of internal services, the ports they run on, the protocols, and the existence of override hooks. That turns a blind external assessment into an informed one. It is the kind of finding that does not pop a shell by itself but makes every other attack cheaper, especially the host-override fields, which hint at where server-side request forgery might be worth a very hard look.

The deeper issue is cultural. A debug tool this powerful was shipped to production and left one flag flip away from the public. That says the gate was built on the assumption that nobody would look, and "nobody will look" is the assumption bug hunters exist to break.

## The lesson

Two takeaways, one technical and one about how to think.

**Technical:** if the browser is checking a flag to decide what a user may see or do, it is not a security control. Access control has to be enforced on the server, where the user cannot reach it. "Hidden" is not "protected." Debug and developer tooling must be removed from production builds or guarded by real server-side authorization, not by a boolean the client can rewrite.

**Mindset:** I did not find this by looking for it. I found it by taking something the app assumed was stable, every `false` in its responses, and refusing to let it stay stable. The console appearing was a surprise. The bug was in chasing that surprise instead of reverting it. When the application does something unexpected, that is not noise to clean up, that is the application telling you where its assumptions are thin. Go there.

## How they should fix it

- **Strip dev and debug tooling from production builds.** The cleanest fix: if the code is not in the bundle, no flag can summon it.
- **Enforce access server-side.** If internal tooling must ship, gate it behind real authorization (auth role or IP allowlist) checked on the server, not a response-body flag.
- **Stop trusting client-reported state** for any security decision. Treat every flag, toggle, and "disabled" attribute you send to the browser as something the user can and will change.

## TL;DR

- I set a blunt Burp rule to replace every `false` with `true` in response bodies, not to exploit a flag but to see what the app does when its assumptions break.
- A hidden internal developer console appeared, with a Service Overrides tab exposing internal service names, ports, protocols, and host-override hooks.
- Root cause: the console's visibility was enforced in the browser via a flag the server sent to the user. Client-side enforcement of a server-side decision is not access control.
- Impact: information disclosure and strong reconnaissance aid for mapping the backend. Triaged as an accepted P4.
- The real lesson: break the app's assumptions on purpose, and when something unexpected happens, chase the "why" instead of reverting it.
