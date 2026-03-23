---
title: The Cognitive Price of Abstraction
author: Maximilian Dollinger
date: 2026-03-23
lastmod: 2026-03-23
draft: true
tags: [abstraction, cognitive-load, software-design, software-complexity, debugging, maintainability]
description: Part 2 of a series on abstraction failure. Part 1 diagnosed how abstractions fail. This post asks the prior question: what does an abstraction actually cost? The answer is grounded in the cognitive constraints that make comprehension hard, the limits of your working memory, and the defect evidence that shows where overloaded working memory leads.
---

# The Cognitive Price of Abstraction

The case for abstraction is well-rehearsed. Dijkstra called it the only mental tool by which finite reasoning can address a multitude of cases (Dijkstra, 1972). Functions, modules, types, interfaces: these compress complexity into chunks a finite mind can manipulate. Every variable name is an abstraction. Every function call is an abstraction. The benefit is real, and it is enormous. It is also the only half of the trade-off that gets examined.

Every boundary is a named concept the developer must learn and hold in memory. Every layer between a symptom and a root cause is distance. When a boundary compresses more complexity than it introduces, the developer can reason about larger problems. When it does not, the developer's finite capacity is spent on the structure of the code rather than the problem the code is meant to solve. The benefit side has Dijkstra, Parnas, and fifty years of consensus behind it. The cost side has folklore. This post tries to ground the cost in evidence: what the cognitive science says about the mechanism, and what the defect data says about where it leads.

This is Part 2 of a series on abstraction failure. [Part 1](link-to-part-1) diagnosed *how* abstractions fail, identifying four distinct modes: dependency (the team doesn't understand the implementation beneath the boundary), leakage (the boundary's promises give way and implementation details bleed through), inversion (the abstraction hides capabilities the team actually needs, forcing workarounds), and drift (a boundary that was well-drawn at design time becomes inadequate as the implementation changes underneath). This post asks the prior question: what is the actual cost of a boundary, even when it does not fail in any of those ways? The scope is single-team codebases where developers share code and communicate directly. Multi-team dynamics are a different problem and are out of scope. Even within single-team codebases, the cost-benefit calculus shifts with scale: a 500-line script and a 200,000-line monorepo have very different abstraction economics, and this post does not attempt to draw the line.

The cognitive science I draw on (working memory limits, chunking, cognitive load theory) is well-established within experimental psychology, instructional design, and small-scale studies of code comprehension. What I do with it is extend it to architectural-scale decisions about abstraction in production systems. That extension follows the direction the evidence points, but goes beyond what any single study has directly tested. Where I am extending, I flag it.

-----

## The Cognitive Bottleneck

### Working Memory Is Small

In 1956, George Miller introduced the concept of the "chunk" as the unit of working memory: the largest meaningful unit a person recognises as a single thing (Miller, 1956). Miller observed that several cognitive tasks seemed to break down at around seven items, though he treated this number as a rough heuristic and noted the convergence across tasks was likely coincidental. Subsequent research revised the figure downward. Cowan (2001, 2010), in a wide-ranging review, argued that when rehearsal and grouping strategies are controlled for, the effective capacity is closer to three to five chunks. The exact number remains debated. It varies with the type of material, the task, and the individual. What does not vary is that the capacity is small. And crucially, chunk size depends on expertise: what a skilled reader recognises as a single unit, a novice must hold as several separate elements.

### Comprehension Is the Bottleneck

This constraint governs the central activity of software development. Xia et al. (2018), in a field study of approximately 80 professional developers across seven projects and over 3,000 working hours, found that comprehension and navigation together consume over 80% of developer time. Editing (the activity that actually changes the codebase) accounts for roughly 5%. That figure includes all comprehension: understanding domain logic, navigating code structure, reading documentation, undifferentiated. The paper does not quantify how much of the 80% is spent on the problem itself versus the way the code is organised.

But its root-cause analysis of long comprehension sessions points heavily toward structural factors: insufficient comments, meaningless names, large classes, inconsistent coding styles, and poor documentation. The abstraction-specific share is unquantified, but the overall budget is large enough that even a fraction of it is significant. The bottleneck is not writing code. It is understanding it. And the bottleneck of understanding is working memory.

Therefore: anything that occupies working memory without aiding comprehension is a direct tax on the developer's primary activity. This is where abstraction becomes a double-edged tool. A good abstraction compresses complexity, freeing working memory for the problem at hand. A bad abstraction adds to the load without compressing anything.

### Load That Helps and Load That Doesn't

To reason more precisely about this, I want to borrow a distinction from educational psychology: the difference between cognitive load that serves comprehension and load that doesn't.

In 1988, John Sweller introduced Cognitive Load Theory (CLT) with work on cognitive load during problem solving (Sweller, 1988). In subsequent work, Sweller and colleagues developed the framework into a tripartite taxonomy distinguishing intrinsic load (the inherent difficulty of the material), extraneous load (effort caused by how information is presented rather than the information itself), and germane load (productive effort spent building mental models, or schemas) (Sweller, van Merriënboer, & Paas, 1998). The taxonomy has been contested internally: Kalyuga (2011) argued that the distinction between intrinsic and extraneous load may not be as clean as originally claimed, since what counts as "intrinsic" depends on the learner's prior knowledge. Sweller (2010) responded with a reformulation. This matters because the post relies on a version of the helpful/unhelpful distinction. But the core insight survives: working memory has limited capacity, and the way information is structured matters as much as its quantity.

CLT was developed for instructional design (presenting material to learners), and software maintenance is a different cognitive task: an expert performing recall and reasoning under time pressure, not a novice building schemas from scratch. The distinction between helpful and unhelpful load, though, does not depend on the learner's level. It depends on the structure of the material.

In software terms, this maps onto a distinction Fred Brooks (1987) drew: a developer reading code spends effort on the inherent difficulty of the problem (what Brooks called essential complexity), and effort on the way the code happens to be organised (what Brooks called accidental complexity). The essential complexity cannot be reduced without changing the problem. The accidental complexity is a design variable.

Peitek et al. (2021), using fMRI during program comprehension tasks, observed that code complexity was associated with increased activation in brain regions linked to working memory: the more distinct concepts a developer must track, the greater the measured neural response. Their study used short code snippets, not multi-layer architectures. The step from snippet-level complexity to architectural-level burden is an extension, but it follows the direction of the mechanism: more concepts to hold means more demand on a system with fixed capacity.

### What This Means for Abstraction

A good abstraction trades scattered implementation details for a single concept, a schema. The developer no longer holds the internals in working memory; they hold the interface. Net load goes down. Soloway and Ehrlich (1984) demonstrated a specific version of this benefit: expert programmers recognise conventional code-level patterns (loop idioms, sentinel variables) as units, performing dramatically better than novices when code follows standard conventions. That speedup *is* abstraction working as intended: a familiar pattern compresses what would otherwise be multiple separate elements into a single chunk. When the conventions were violated, the expert advantage collapsed to novice level, which is itself evidence that the benefit depends on the abstraction being learnable and conventional, not merely present.

A bad abstraction does the opposite. It adds load without reducing complexity. The developer now holds the abstraction itself (its name, its interface, its conventions), plus the details it fails to hide (because the boundary leaks or operates at the wrong level), plus the mapping between the two. The specific cost of holding this mapping has not been isolated in empirical studies of software comprehension. But it follows from the chunking mechanism: each side consumes chunks, and maintaining the correspondence is itself a cognitive task that competes for the same limited capacity.

One point deserves stating explicitly: the alternative to an abstraction is not nothing. It is the uncompressed implementation. Remove a boundary and the developer does not face fewer concepts; they face all the details the boundary was meant to hide. The argument here is not that abstraction is net negative by default. It is that a boundary must compress more than it costs, and some boundaries fail this trade-off.

### Familiarity Is Not Compression

The obvious counter-argument: someone who works with a particular module daily would navigate its layers faster. Miller's chunking mechanism explains why. Expertise builds schemas that compress a multi-step traversal into a single cognitive step. The Soloway and Ehrlich finding confirms this for code-level patterns (their study operated at the level of code idioms; the extension to architectural familiarity is plausible but untested). The benefit of familiarity depends on the code following patterns the developer already knows, not merely on the existence of abstraction. An unfamiliar abstraction does not compress; it adds load.

And in any sufficiently large codebase, most encounters with a module are by people who are not experts in that specific module. LaToza, Venolia, and DeLine (2006) found that developer mental models exist only in their holders' heads, are shared through face-to-face conversation, and go stale when the person leaves or the system changes. The knowledge that makes a module navigable is fragile, personal, and non-transferable.

If the cost of an abstraction depends on what the team already knows, and what the team knows is fragile and non-transferable, then the cost cannot be captured by any metric that looks only at the code. Hao et al. (2023) tested this directly. They measured cognitive load via EEG while programmers read code, and compared what the programmers actually experienced as difficult against what standard static metrics predicted. Metrics such as McCabe's cyclomatic complexity and SonarQube's Cognitive Complexity deviated considerably from measured cognitive difficulty. Some code that scored high on complexity metrics was processed easily by developers who had the right schemas; some code that scored low overwhelmed developers who lacked them.

The mismatch was large enough to undermine the premise that static analysis can substitute for human judgment about cognitive cost. If tooling cannot reliably measure the cost of an abstraction, then the decision of when to abstract requires judgment about what creates comprehension difficulty in a specific context, for a specific team. This is an uncomfortable conclusion for an industry that prefers automatable answers, but the evidence points there.

-----

## From Cognitive Load to Defects

The previous section established that code complexity taxes working memory. This section argues that the tax leads to defects, but the relationship deserves more care than a simple causal arrow.

Complex code produces more bugs. That is the most replicated finding in defect prediction research, and it operates through several channels: complex code is harder to test thoroughly, harder to review effectively, and has more combinatorial surface area for things to go wrong. This post focuses on the cognitive channel, the claim that working memory overload is a direct source of programming errors, because it is the channel most affected by abstraction decisions. But the argument does not require the cognitive pathway to be the only one. The channels are complementary, and they compound.

Before continuing, a note on what this section does. It chains together evidence from several distinct research traditions: controlled studies of programming errors (Anderson & Jeffries, Crichton et al.), biometric field studies of professional developers (Müller & Fritz), defect prediction at the component level (Nagappan et al., Zimmermann & Nagappan), and debugging studies (Schröter et al., Eisenstadt, Parnin & Orso). Each link is individually supported. But the overall claim, that working memory overload produces defects which then survive longer in layered code, requires all of them to hold simultaneously. No single study has tested the full chain.

### The Cognitive Channel: From Overload to Errors

The evidence that working memory overload directly produces programming errors is older and more direct than is often recognised. Anderson and Jeffries (1985) studied errors that students made while working with LISP functions across four experiments. They found that errors occur when there is a loss of information in the working memory representation of the problem, and that error frequency increases with the complexity of irrelevant aspects of the task, that is, extraneous load in CLT terms. The errors were slips, not misconceptions: random in distribution, consistent with a capacity mechanism rather than a knowledge gap. (The study used students, not professional developers. But the mechanism depends on schema availability for the specific material, not on career stage. A senior developer encountering unfamiliar code lacks the relevant schemas just as a student does, and most encounters with a given module are by people who are not experts in that module.)

Crichton, Agrawala, and Hanrahan (2021) updated this finding for modern program comprehension. In a controlled study of program tracing (mentally simulating a program on concrete inputs), they confirmed that working memory limits from cognitive psychology transfer to the programming domain. Participants failed at predictable points when the number of variables they had to track exceeded their working memory capacity, and they made characteristic WM errors, accidentally swapping associations between variables. Participants who adopted a strategy requiring more simultaneous tracking (following data dependencies upward rather than reading linearly) made more WM errors when tracing straight-line code. The mechanism, that exceeding WM capacity produces specific observable errors during code comprehension, is the same mechanism that operates when a developer traces execution across abstraction layers.

The step from controlled studies to production code is bridged, partially, by Müller and Fritz (2016). In a field study with ten professional developers over two weeks, they measured cognitive load via biometric sensors (heart rate variability, electrodermal activity, skin temperature) while the developers worked on real change tasks. Code elements where developers experienced higher measured cognitive load subsequently had more quality concerns identified in peer code reviews. The biometric classifier detected 50% of the bugs found in reviews, outperforming classifiers based on traditional code metrics. A replication with five developers in a different country and company produced similar results. The sample is small and the study correlational, but it is the closest available evidence linking physiologically measured cognitive difficulty to actual defects in professional software development.

### Why Structural Complexity Concentrates the Risk

These studies establish that cognitive overload produces errors in programming tasks. The defect prediction literature shows where those errors accumulate in production systems.

Nagappan, Ball, and Zeller (2006) studied the post-release defect history of five Microsoft software systems and found that failure-prone components are statistically correlated with code complexity measures. What matters for the abstraction question is which *kind* of complexity drives the risk. Zimmermann and Nagappan (2008) found that structural complexity (the coupling and dependencies *between* components) predicts defects more strongly than internal complexity within any single function. The way components relate to each other is where the defect risk concentrates.

These studies measured existing coupling in shipped systems, not the marginal effect of adding or removing a boundary. But the inference follows: if inter-component coupling is where defect risk concentrates, and every boundary creates a coupling surface between components, then unjustified boundaries create coupling surfaces that serve no compensating purpose. A justified boundary removes coupling by hiding an independent decision. An unjustified boundary adds a coupling surface without removing any. The claim is not "fewer boundaries, fewer defects." It is that unjustified boundaries add coupling without compensating benefit. The hard question is how you know in advance which boundaries are justified.

### Why Abstraction Layers Make Bugs Survive Longer

Even when a defect exists, the time it takes to find and fix it varies enormously. And the dominant factor in that variance is distance between cause and symptom.

Schröter, Bettenburg, and Premraj (2010), in a mining study of Eclipse project bug reports, found that approximately 40% of bugs were fixed in the top frame of the stack trace (the location closest to the visible symptom) and close to 88% were fixed within the top ten frames. Bug reports that included stack traces were fixed significantly faster than those without. The pattern is clear: the further the root cause sits from the symptom, the harder the bug is to find and fix. Every function boundary is an abstraction boundary: it hides an implementation behind an interface. A ten-frame stack trace is ten such boundaries, and each one displaces upstream context from working memory. The developer traversing frame eight is no longer holding the context from frames one through three, not because those frames were in a different domain, but because working memory is full.

Eisenstadt (1997), whose qualitative study of debugging war stories was discussed in Part 1, provided a framework for understanding *why* distance matters: he identified large spatial or temporal chasms between root cause and symptom as one of the two dominant sources of debugging difficulty. The Schröter et al. data gives that framework quantitative support: bugs close to the symptom get fixed; bugs far from the symptom survive.

Parnin and Orso (2011), in a study that won the ISSTA 2021 Impact Paper Award, found converging evidence from the tool side. They investigated whether automated fault localization actually helps programmers and found that on easier tasks, the tool helped experienced developers find faults faster. On harder tasks, where cause and effect were more distant, the tool provided no benefit. Even with a pointer to the faulty line, the developer still needed to understand *why* that line was wrong, and that understanding required cross-layer context the tool could not provide. The pattern is consistent with what the working memory mechanism predicts: when the context needed to understand a fault spans more layers than working memory can hold, pointing to the location without providing the context cannot close the gap.

-----

## What This Means

If abstraction has a cognitive cost that compounds into defects, through both increased bug production and increased debugging distance, then the decision of when to introduce a boundary is not merely aesthetic. The cost side of the equation is empirically grounded: working memory is small, comprehension dominates developer time, overload produces characteristic errors, and layered code makes those errors harder to find and fix.

But the cost is only one side. A good abstraction compresses complexity, freeing working memory for the actual problem, and the Soloway and Ehrlich finding shows how large the benefit can be when the abstraction matches the developer's schemas. The difficulty is that both cost and benefit depend on what the team already knows, and that knowledge is perishable and context-bound. The cost of an abstraction is empirically grounded. When that cost is worth paying is a judgment call that no study has systematized.

Part 1 showed that abstractions fail in distinct ways. This post shows that the act of introducing an abstraction has a cost even when it does not fail in any of those ways. Every boundary occupies working memory. Every coupling surface is a place where defects can hide. Every layer between symptom and root cause is distance a developer must cross. The question is not whether abstraction can go wrong. It is whether, for a given boundary in a given context, the compression it provides justifies the load it imposes. The evidence can inform that judgment. It cannot replace it.

-----

## Sources

Anderson, J. R., & Jeffries, R. (1985). Novice LISP errors: Undetected losses of information from working memory. *Human-Computer Interaction*, *1*(2), 107–131.

Brooks, F. P. (1987). No silver bullet: Essence and accidents of software engineering. *Computer*, *20*(4), 10–19.

Cowan, N. (2001). The magical number 4 in short-term memory: A reconsideration of mental storage capacity. *Behavioral and Brain Sciences*, *24*(1), 87–114.

Cowan, N. (2010). The magical mystery four: How is working memory capacity limited, and why? *Current Directions in Psychological Science*, *19*(1), 51–57.

Crichton, W., Agrawala, M., & Hanrahan, P. (2021). The role of working memory in program tracing. In *Proceedings of the 2021 CHI Conference on Human Factors in Computing Systems* (Article 56, pp. 1–13). ACM. <https://doi.org/10.1145/3411764.3445257>

Dijkstra, E. W. (1972). The humble programmer. *Communications of the ACM*, *15*(10), 859–866. <https://doi.org/10.1145/355604.361591>

Eisenstadt, M. (1997). My hairiest bug war stories. *Communications of the ACM*, *40*(4), 30–37.

Hao, G., Hijazi, H., Durães, J., Medeiros, J., Couceiro, R., Lam, C. T., Teixeira, C., Castelhano, J., Castelo Branco, M., Carvalho, P., & Madeira, H. (2023). On the accuracy of code complexity metrics: A neuroscience-based guideline for improvement. *Frontiers in Neuroscience*, *16*, 1065366.

Kalyuga, S. (2011). Cognitive load theory: How many types of load does it really need? *Educational Psychology Review*, *23*(1), 1–19.

LaToza, T. D., Venolia, G., & DeLine, R. (2006). Maintaining mental models: A study of developer work habits. In *Proceedings of the 28th International Conference on Software Engineering* (pp. 492–501). ACM.

Miller, G. A. (1956). The magical number seven, plus or minus two: Some limits on our capacity for processing information. *Psychological Review*, *63*(2), 81–97.

Müller, S. C., & Fritz, T. (2016). Using (bio)metrics to predict code quality online. In *Proceedings of the 38th International Conference on Software Engineering* (pp. 452–463). ACM/IEEE. <https://doi.org/10.1145/2884781.2884803>

Nagappan, N., Ball, T., & Zeller, A. (2006). Mining metrics to predict component failures. In *Proceedings of the 28th International Conference on Software Engineering* (pp. 452–461). ACM.

Parnin, C., & Orso, A. (2011). Are automated debugging techniques actually helping programmers? In *Proceedings of the 2011 International Symposium on Software Testing and Analysis* (pp. 199–209). ACM.

Peitek, N., Apel, S., Parnin, C., Brechmann, A., & Siegmund, J. (2021). Program comprehension and code complexity metrics: An fMRI study. In *Proceedings of the 43rd International Conference on Software Engineering* (pp. 524–536). IEEE/ACM.

Schröter, A., Bettenburg, N., & Premraj, R. (2010). Do stack traces help developers fix bugs? In *Proceedings of the 7th IEEE Working Conference on Mining Software Repositories* (pp. 118–121). IEEE. <https://doi.org/10.1109/MSR.2010.5463280>

Soloway, E., & Ehrlich, K. (1984). Empirical studies of programming knowledge. *IEEE Transactions on Software Engineering*, *SE-10*(5), 595–609.

Sweller, J. (1988). Cognitive load during problem solving: Effects on learning. *Cognitive Science*, *12*(2), 257–285.

Sweller, J. (2010). Element interactivity and intrinsic, extraneous, and germane cognitive load. *Educational Psychology Review*, *22*(2), 123–138.

Sweller, J., van Merriënboer, J. J. G., & Paas, F. (1998). Cognitive architecture and instructional design. *Educational Psychology Review*, *10*(3), 251–296.

Xia, X., Bao, L., Lo, D., Xing, Z., Hassan, A. E., & Li, S. (2018). Measuring program comprehension: A large-scale field study with professionals. *IEEE Transactions on Software Engineering*, *44*(10), 951–976.

Zimmermann, T., & Nagappan, N. (2008). Predicting defects using network analysis on dependency graphs. In *Proceedings of the 30th International Conference on Software Engineering* (pp. 531–540). ACM.
