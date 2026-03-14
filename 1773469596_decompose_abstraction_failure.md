---
title: Decomposing Abstraction Failure
author: Maximilian Dollinger
date: 2026-03-13
lastmod: 2026-03-13
draft: true
tags: [abstraction, leaky-abstractions, software-architecture, software-design, architecture-erosion, software-complexity]
description: Engineers argue endlessly about whether abstractions are good or bad, but these debates rarely go anywhere because the participants are diagnosing different problems. This post identifies four distinct ways abstractions fail — dependency, leakage, inversion, and drift — and offers a diagnostic for knowing which fix to reach for first.
---
# Decomposing Abstraction Failure

Endless arguments are fought over whether a given abstraction is good or bad. ORMs versus raw SQL. Microservices versus monoliths. Thin wrappers versus thick frameworks. The debates generate a lot of heat and not much resolution, in part because the participants are often talking past each other. One engineer says the ORM is a problem because nobody on the team understands the queries it generates. Another says it's a problem because it leaks database-specific behavior through what's supposed to be a portable interface. A third says it's a problem because it won't let them write the queries they actually need. These are three different diagnoses with three different fixes, and collapsing them into "the ORM was a bad choice" is why the conversation never goes anywhere.

The literature on abstraction — spanning foundational theory from the 1970s, architectural mismatch research from the 1990s, empirical studies of developer cognition, and practitioner writing — identifies four distinct ways abstractions fail. They look similar from the outside, overlap in practice, and need different responses. This post synthesizes them into a diagnostic: a framework for decomposing abstraction-related incidents into their constituent problems so you know which fix to reach for first.

-----

## What Abstraction Is and What It Costs

In his 1972 Turing Award lecture, Dijkstra described abstraction as the only mental tool by which finite reasoning can address a multitude of cases. Without it, every piece of software would require holding the entire implementation in your head simultaneously. That same year, Parnas published his [argument](https://doi.org/10.1145/361598.361623) for information hiding: the value of a module boundary comes from isolating design decisions that could change independently. A good boundary hides *the right complexity* — the decisions most likely to shift — so the rest of the system is insulated when they do.

Between them, the theory was essentially complete by 1972. What follows is about the gap between what's known and what's practiced.

-----

## Dependency: A Learning Problem

The moment you adopt someone else's abstraction, you inherit their implementation decisions — including the ones that will eventually conflict with what you need. While the abstraction holds, this dependency is invisible. The cost surfaces when something breaks in a way the abstraction didn't anticipate.

Some strategies reduce dependency: learn the implementation yourself, hire someone who knows it, replace it with something you understand better, narrow your usage surface to the well-understood subset, contribute upstream to make behavior more transparent. All of these leave the team genuinely better equipped to work with the layer beneath.

Containment is different. Circuit breakers, fallbacks, and graceful degradation accept the dependency and limit the blast radius. You still don't understand what's underneath; you've decided not to crash when it fails. Containment is sometimes the only option — a critical-path dependency on a third-party payment API with opaque internals isn't going to become learnable no matter how much you invest. In that case, containment isn't a consolation prize. It's the engineering discipline of resilience applied correctly. But confusing containment with resolution — treating blast-radius management as if it were understanding — is how teams end up with systems that fail gracefully but are impossible to improve. Know which one you're choosing.

Why is dependency so persistent? Because the knowledge needed to resolve it is expensive to acquire and fragile to maintain. LaToza, Venolia, and DeLine studied developer work habits at a large software company and found that developers invest enormous effort building mental models of code — models that exist only in their heads, are shared through face-to-face conversation, and go stale when the system changes or the person leaves. Written documentation was genuinely inadequate for the questions developers actually needed answered. The implicit knowledge problem isn't a failure of discipline. It's a structural property of how knowledge lives in software teams. Given finite learning budgets, the question isn't "why don't engineers learn everything underneath?" It's "which layers should this team invest in understanding?" — and the diagnostic at the end of this post is one way to answer that.

-----

## Leakage: A Boundary Problem

You write a SQL query. It's logically correct. It runs in 200 milliseconds on your development database and 45 seconds in production. You rewrite it — same logic, different structure — and now it's fast. The query planner, which SQL was supposed to abstract away, has bled through.

In 2002, Spolsky gave this phenomenon a name in ["The Law of Leaky Abstractions"](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/): all non-trivial abstractions leak. The underlying complexity doesn't disappear; it waits. He's right that leakage is inevitable. What matters for practitioners is that leakage is also a design variable. A well-drawn boundary doesn't eliminate leakage — it concentrates it, making the bleed-through predictable and manageable. A badly drawn boundary leaks everywhere, unpredictably.

Why are so many boundaries badly drawn? Because the assumptions that define them are implicit. Garlan, Allen, and Ockerbloom documented this in their architectural mismatch research: components make assumptions about each other across at least four dimensions — the nature of components (infrastructure, control model, data model), the nature of connectors (protocols, data model), global architectural structure, and the construction process — and these assumptions are overwhelmingly undocumented. They aren't written down. They aren't exposed by interfaces. The assumptions that need to be stable are the ones nobody thought to state. This isn't a problem that individual diligence solves. It's a property of how software components are designed and distributed.

The SQL example illustrates how leakage and dependency interact. The leakage is a property of the boundary: SQL's declarative interface doesn't fully insulate users from planner behavior, and that's a permanent design fact. But when a team is *surprised* by the leakage, that surprise is a dependency problem — they didn't learn the layer below. Since the boundary isn't going to change (SQL is what it is), the productive response is to resolve the dependency: learn how the planner works, map where it leaks, and absorb that into your working knowledge of the tool.

-----

## Inversion: A Level Problem

Sometimes the abstraction hides capabilities the team actually needs, forcing them to re-implement lower-level functions using the higher-level interface. This is abstraction inversion — traditionally discussed as a design-time problem (first widely noted in the context of Ada's rendezvous construct, which forced programmers to build simpler synchronization primitives from a more complex one), but the same dynamic shows up in production: the team discovers mid-incident that the abstraction won't let them do what they need.

The team may understand the implementation perfectly. The boundary may be well-drawn. The problem is that the abstraction operates at the wrong level: it took away access the team needs and gave back machinery the team doesn't want.

The third ORM complaint from the opening — the team can't write the queries they need — is often this. They don't merely need to understand the generated SQL (dependency) or cope with the ORM leaking planner behavior (leakage). They need to write SQL the ORM won't let them express — a window function, a recursive CTE, a query hint. The abstraction hasn't leaked; it's *blocked access*. The fix isn't studying the layer below or redrawing the boundary. It's operating at a different level of abstraction entirely: choose a less opinionated tool, or restructure the interface to expose what the team actually needs.

-----

## Drift: When Good Boundaries Go Bad

A boundary drawn well at design time can become inadequate as the implementation changes underneath. Consider a database connection pool configured for a specific driver version. The pool's timeout and recycling settings are tuned for that driver's connection lifecycle: how it signals staleness, when to evict. The driver ships a minor update. Keepalive semantics shift. The pool configuration hasn't changed, but it was right for the old behavior. Under sustained load, the pool hands out connections the driver considers stale. Requests fail intermittently. Every metric looks healthy.

This is drift. The boundary encoded specific implementation behavior rather than the abstract promise (managed connections that are valid when handed out). When the implementation moved, the boundary leaked in a new place.

Perry and Wolf identified the broader phenomenon in their 1992 foundational work on software architecture, distinguishing *drift* — where a system's implementation diverges from its intended architecture through insensitivity — from *erosion*, where design decisions actively violate architectural principles. Li et al.'s systematic mapping study of 73 papers on architecture erosion found that both technical and non-technical factors drive the divergence, with consequences spanning performance degradation, maintenance difficulty, and system brittleness.

Sandi Metz described the practitioner experience of a related form of decay in "The Wrong Abstraction": an abstraction that was appropriate when introduced accumulates conditional paths as requirements diverge, until the shared code serves nobody well. Whether the implementation moves underneath (drift) or the use cases diverge above (Metz's pattern), the result is the same: a boundary that once worked no longer does, and the team has to recognize the decay before they can address it.

The advice is Parnas's, applied over time: draw boundaries around *what* the implementation promises, not *how* it currently delivers. Boundaries that encode version-specific behavior break when the implementation moves. Boundaries drawn against the abstraction's contract survive. The difficulty, as Garlan et al. documented, is that implementations rarely make their promises explicit.

-----

## The Diagnostic

When something breaks involving an abstraction, the first move is not to start debugging. It is to ask: *what kind of failure am I looking at?*

**Is the team confused about what the implementation is doing?** That's dependency. Reduce it: invest in understanding, replace with something better known, narrow the usage surface. If the dependency isn't learnable, contain it — but know you're choosing containment.

**Is the team confused about where the abstraction's promises give way?** That's leakage. Redraw the boundary to concentrate it, or — if the boundary is outside your control — map the leakage and absorb it into your working knowledge.

**Is the team working around the abstraction rather than through it?** That's inversion. Change the level of abstraction: choose a different tool, restructure the interface, or back out the abstraction entirely.

**Has a boundary that used to work stopped working?** That's drift. Re-evaluate the boundary against what the system has become, not what it was when the boundary was drawn.

Most real incidents involve more than one mode. The SQL query that surprises the team is leakage compounded by dependency. The ORM that keeps requiring escape hatches is inversion compounded by leakage. Decomposing the incident tells you which responses to apply and in what order. Studying the implementation when the boundary is the problem, redesigning the boundary when nobody understands what's underneath, or doing either when the level itself is wrong — all deepen the confusion instead of resolving it.

Eisenstadt's 1997 study of debugging war stories found that 53% of debugging difficulty came from just two sources: large gaps between root cause and symptom, and bugs that rendered debugging tools inapplicable. He was studying debugging in general, not abstraction failures specifically — but the connection is hard to ignore. A leaky boundary puts the symptom in one layer and the cause in another. A dependency gap means the team can't traverse the distance. An inverted abstraction means the team's tools operate at the wrong level to see the problem. Drift means the map the team is using no longer matches the territory.

The diagnostic doesn't make any of this easy. It makes it tractable. Invest in understanding the layers where your incidents cluster. Map the leakage of the boundaries you can't change. Replace the abstractions whose level doesn't match your work. Revisit the boundaries you stopped examining. The framework has been available for fifty years. The question is whether the engineers using the tools bring the fundamentals to apply it — and whether their organizations give them the time and support to do so.

-----

## Sources

Dijkstra, E. W. (1972). The humble programmer. *Communications of the ACM*, *15*(10), 859–866. <https://doi.org/10.1145/355604.361591>

Eisenstadt, M. (1997). My hairiest bug war stories. *Communications of the ACM*, *40*(4), 30–37.

Garlan, D., Allen, R., & Ockerbloom, J. (1995). Architectural mismatch: Why reuse is so hard. *IEEE Software*, *12*(6), 17–26.

Garlan, D., Allen, R., & Ockerbloom, J. (2009). Architectural mismatch: Why reuse is still so hard. *IEEE Software*, *26*(4), 66–69.

LaToza, T. D., Venolia, G., & DeLine, R. (2006). Maintaining mental models: A study of developer work habits. In *Proceedings of the 28th International Conference on Software Engineering* (pp. 492–501). ACM.

Li, R., Liang, P., Soliman, M., & Avgeriou, P. (2022). Understanding software architecture erosion: A systematic mapping study. *Journal of Software: Evolution and Process*, *34*(3), e2423.

Metz, S. (2016, January 20). The wrong abstraction. sandimetz.com. <https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction>

Parnas, D. L. (1972). On the criteria to be used in decomposing systems into modules. *Communications of the ACM*, *15*(12), 1053–1058. <https://doi.org/10.1145/361598.361623>

Perry, D. E., & Wolf, A. L. (1992). Foundations for the study of software architecture. *ACM SIGSOFT Software Engineering Notes*, *17*(4), 40–52.

Spolsky, J. (2002, November 11). The law of leaky abstractions. *Joel on Software*. <https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/>
