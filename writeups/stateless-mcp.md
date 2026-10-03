# MCP Deleted the Handshake, and That's Harder Than It Sounds

I've been building on the Model Context Protocol for a while, and the 2026-07-28 revision did something that looked tiny in the changelog and turned out to reshape how you write a server. It deleted the handshake.

No more `initialize`. No `notifications/initialized`. No `Mcp-Session-Id`. A server is now forbidden from assuming anything based on earlier requests on the same connection. Whatever it needs to answer a request has to arrive with that request.

On paper that reads like a simplification. In practice it quietly flips the whole thing inside out, and I wanted to actually understand how, so I did the slightly obsessive version of learning it. I implemented the spec from scratch, no SDK, zero runtime dependencies, and wrote a conformance suite to check I'd actually followed it. The repo is mcp-stateless-recon. Here's what I ran into.

## Why from scratch

I could have wrapped the official SDK and moved on. But you don't really learn a protocol by letting a library implement it for you, you learn it by getting the fiddly parts wrong and then right. So I implemented the wire protocol directly: the `_meta` key grammar, the `resultType` rules, the error-code bands, the routing headers, the Tasks state machine, the audience-bound tokens. My rule was that if the spec had a sentence with a MUST in it, I wanted a test named after that sentence.

That's roughly how it ended up at 708 tests, 140 of them conformance tests named after the requirement they assert. Sounds like overkill until you notice how many ways there are to be quietly non-compliant.

## Statelessness is easy to claim and hard to prove

Here's the trap that makes this revision interesting.

With no handshake, capabilities are per-request now, not per-connection. A client might say it supports the Tasks extension on one call and not mention it on the next, and the server has to answer each call on its own terms. "Does this client support tasks?" stops being something you can look up once and cache.

Anything that needs to outlive a single request now needs an explicit name that the client carries. Long-running work becomes a `taskId`. A call that's paused waiting for the user to answer something becomes a signed continuation token. Nothing sits in server memory tied to a socket.

The payoff is real. Any request can land on any instance, so you put the server behind a round-robin load balancer and the instances are interchangeable, a restart loses nothing, no sticky sessions needed.

But that's also exactly the trap, because the wrong version works perfectly on your laptop. If you park state in a single Map keyed by connection, every test you write on one machine with one client passes. Stateful leakage doesn't throw. It just quietly works in dev and then produces rare, unreproducible bugs in production once a second instance exists. That's the worst kind of bug, the kind that sails through CI.

So I decided statelessness was something to prove rather than assert. There's a test that boots two completely separate server instances, starts a multi-round-trip exchange on the first, answers it on the second, and finishes it back on the first. They share no memory at all, only the secret key used to sign continuation tokens. If any state had leaked into instance memory, that test falls apart. It's the one I trust most in the whole repo.

## The small things the spec is quietly strict about

The revision also added a pile of details that are easy to do carelessly and genuinely satisfying to do properly.

There's a formal grammar for `_meta` key names, and the reservation rule hinges on the second label of a prefix, so `com.mcp.tools/` is reserved for MCP but `com.example.mcp/` is not. Easy to skim past, fiddly to get exactly right.

Every result carries a `resultType`, and the legal set isn't fixed. It's the core values plus whatever the extensions this particular client declared bring along. So "is this a valid result type" turns into a per-request question too.

There are error-code bands with a couple of codes you're forbidden from using at all. There are routing headers (`Mcp-Method`, `Mcp-Name`) that let a gateway route a request without parsing the body, which only works because the spec now says disagreement between the header and the body is itself an error, code `-32020`. Implementing that mismatch rule is the whole reason the header can be trusted.

The Tasks extension has its own quiet rules: a terminal status must never change once it's set, and you must never hand a task back to a client that never said it could poll for one. And RFC 8707 resource indicators are the defence against a token minted for one MCP server being replayed against another, so tokens get bound to an audience and the server refuses to even start in that mode without a secret.

None of these are hard on their own. The interesting part is that together they punish the "eh, I'll just cache that" instinct at every single turn.

## The recon part exists to make it real

A protocol implementation with no real I/O is just a pile of opinions. So the server exposes a small network-reconnaissance toolserver: DNS resolution, TLS inspection, security-header grading, a bounded port scan, and a composite sweep. The point of those tools isn't the recon, it's that they exercise the hard parts of the spec with actual work. The port scan is long-running, so it drives the whole Tasks flow. The sweep pauses to ask the operator for authorisation and then for scope over two elicitation round trips, which is exactly the "a paused call has to survive on a different instance" path I wanted to stress.

Because it makes real network calls, the thing I tested hardest isn't any of the protocol code, it's the SSRF guard. Every target is checked against an allowlist and classified against the non-routable ranges before a single packet leaves, on the original request and on every redirect hop. The recon tools are the demo, but that guard is the part I actually lose sleep over, so it has more tests than anything else in the repo.

## What I took from it

The big one: when a system removes state, it doesn't remove the need for state, it just pushes it out into the open where you have to name it. Every bit of state I used to leave implicit under a connection became an explicit, signed, client-carried thing. More work up front, far less mystery later.

The other lesson is older than MCP. If a property matters and its failure is invisible, an ordinary test won't catch it, because the broken version passes too. You have to build the adversarial setup that can actually fail (here, two independent instances) and make that the thing you trust.

If you want the fiddly details with spec citations, they're in docs/PROTOCOL.md in the repo.

Code: [github.com/abdulsalam-create/mcp-stateless-recon](https://github.com/abdulsalam-create/mcp-stateless-recon)
