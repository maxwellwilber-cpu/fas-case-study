> **Where the system lives:** FAS is built on Vellum, a visual pipeline platform, so the artifact is the architecture rather than a repository of source files. This case study documents that design: 27 nodes, 4 LLM stages, 73 validation checks across 14 governed tables, and fail-closed behavior when validation fails.
>
> **The code that tests it is public.** [EVS](https://github.com/maxwellwilber-cpu/evs) is a 43-test Python framework that validates this pipeline's output ,  it caught 12 of 12 planted errors plus 2 real issues found independently.

# I Built an AI Financial Analyst for Small Businesses. Here's Why 73 Validation Checks Mattered More Than the AI.

*How a non-engineer designed a multi-stage AI pipeline that produces CFO-grade analysis for business owners who can't afford a CFO.*

---

> **System:** 27-node AI pipeline on Vellum · 73 validation checks across 14 governed tables · 96.2/100 quality score · Tested on real client data ($1.41M revenue)
>
> **Related:** [External Validation Script (EVS)](https://github.com/maxwellwilber-cpu/evs) ,  43 pytest tests covering the validation framework itself | [AI Implementations Portfolio](https://github.com/maxwellwilber-cpu/ai-implementations-portfolio) ,  19 documented implementations

---

Every time I work on an AI system, I think about my mom.

She's the most talented staging designer in Silicon Valley. Her eye for design built a successful business. But her lack of business education made everything harder than it needed to be. Pricing decisions, cash flow forecasting, understanding whether a quarter was actually good or just felt good. She paid consultants who didn't really help. She figured it out through grit, but it cost her years of stress she didn't need.

There are millions of business owners like her. Service businesses doing $1M to $5M in revenue. They have QuickBooks and maybe a bookkeeper. They can see their numbers, but they can't interpret them. *"Is a 56% labor cost ratio bad? Compared to what? What should I do about it?"* Answering those questions today means hiring a fractional CFO at $3K to $10K per month, or flying blind. Most choose blind.

I built the Financial Analysis System (FAS) to change that. It takes a standard QuickBooks export and produces what a CFO would deliver: trend analysis, diagnostic claims backed by evidence, actionable recommendations, scenario projections, and a data quality score that tells the owner how much to trust the results. The pipeline runs in under 10 minutes and costs less than a dollar per run.

But the system that matters isn't the AI. It's everything I built around the AI to make sure it tells the truth.

---

## The Problem with AI Financial Analysis

LLMs are confident liars. Early in development, I ran the pipeline end-to-end and got output that looked professional. Clean formatting, specific dollar amounts, plausible recommendations. But when I traced the numbers back, some claims cited metrics that didn't exist in the data. The LLM had hallucinated values that seemed reasonable but were fabricated.

A business owner reading *"your labor costs have increased 15% year-over-year"* will make decisions based on that number. If it's wrong, the system has done harm. And the business owner wouldn't know the difference.

This is the core design problem: how do you build an AI system for people who can't tell when the AI is wrong?

---

## The Architecture: Decompose, Validate, Trace

The answer was decomposition. Eighty percent of financial analysis is deterministic math ,  calculating ratios, detecting trends, comparing periods. Only about twenty percent requires judgment: classifying accounts, diagnosing root causes, writing recommendations. I split the work accordingly.

The FAS is a 27-node pipeline built on Vellum. 23 nodes run deterministic Python. 4 nodes call Claude for judgment tasks. Each component can be tested, debugged, and improved independently. Data flows through seven stages: intake and classification, normalization, metric computation (32 financial metrics), diagnostics, recommendations with scenario projections, validation and output assembly, and parallel delivery to email, Google Sheets, and a React dashboard.

Two design decisions shaped everything:

**Decision 1: LLM-first classification with deterministic validation.** QuickBooks account names are messy and ambiguous. "Tournament Fees" could be revenue or an expense depending on the business. My first approach used pure pattern matching. It classified tournament fees as revenue for a baseball academy, producing a 95%+ operating margin ,  a cascading error that corrupted every downstream metric. The fix: let the LLM classify first (it understands business context), then validate against a deterministic rule set (it catches hallucinated categories). Neither layer alone is reliable. Together, they catch what the other misses.

**Decision 2: Staged validation gates instead of end-of-pipeline checking.** I positioned 73 validation checks at three gates where bad data would be most expensive to let through: after normalization (schema and structural integrity), after metrics (computed value ranges and primary key uniqueness), and before output (full traceability chain across all tables). Every recommendation traces back through a claim to a metric to the raw data that supports it. If any link in that chain breaks, the system halts. It won't produce output it can't verify.

The system operates in three depth modes based on data availability. A single quarter gets a snapshot analysis with point-in-time metrics. Twelve to twenty-four months of data enables full trend analysis, seasonal pattern detection, and variance decomposition. And an ongoing advisory mode ,  currently architected but not yet fully deployed ,  will track changes across recurring analyses for the same client, monitoring whether recommendations were acted on and what shifted.

---

## What Broke

Two failures shaped this system more than any success.

**The single-LLM disaster.** My original approach was one LLM call with the entire dataset. It worked on small test cases. On real data with 24 months of P&L, the context window overflowed at roughly 3,500 lines of computation. Revenue figures from early months disappeared. Recommendations referenced findings that had been silently truncated. That failure forced the pipeline architecture. Instead of one model doing everything, I decomposed into 27 specialized nodes where Python handles the math and LLMs only handle judgment.

**The account classification cascade.** One misclassified account name ,  "Tournament Fees" coded as revenue instead of expense ,  produced a 95%+ operating margin and made every downstream metric wrong. That failure produced the two-layer classification system and reinforced a principle I now apply to every AI system I build: never trust a single source of judgment. Always validate with a second method.

---

## Real Results: Beach City Baseball Academy

The system proved itself on 24 months of real QuickBooks data from a service business running approximately $1.41M in trailing twelve-month revenue.

**The labor cost pattern was more extreme than anyone expected.** Labor as a percentage of revenue ranges from 16% to 92% month to month, with essentially no correlation to revenue levels. The pipeline identified that labor costs don't flex with revenue at all. The correlation between labor exceeding 50% of revenue and negative operating income is r = 0.89. Every time labor spikes relative to revenue, the business loses money.

**The system identified $116K in annual losses from four predictable months.** Four months per year produce guaranteed losses averaging $29K each, creating a $116K annual operating cash deficit. The pipeline generated a specific recommendation: implement flex-staffing aligned to the seasonal revenue pattern. That single recommendation, if implemented, would recover the full deficit.

**The most surprising finding came from something the system didn't know.** The pipeline noticed revenue drops sharply in certain months while labor stays flat. It had no context that the business runs a large BOGO promotion twice a year ,  one that compresses revenue without reducing labor demand. The owners knew the BOGO was an issue but had no idea how much profitability it was costing them. The system made it visible and quantifiable without anyone telling it to look for it.

**The seasonal index prevented a false alarm.** When December 2025 revenue came in down 14.3% year-over-year, the system's seasonal revenue index showed December is historically a weak month and framed it accordingly: *this decline should not be read as representative of business trajectory.* That's exactly what a CFO would tell a panicking business owner. The system generated it automatically from the data.

Overall data quality score: **96.2 out of 100.**

---

## Validating the Validators

I also built a standalone External Validation Script (EVS) in Python to test whether the validation framework itself works. The EVS is a separate codebase with 43 pytest tests that runs against sample data containing 12 intentionally planted errors ,  schema violations, referential integrity breaks, and out-of-range values.

Result: 100% detection rate on all 12 planted errors, plus 2 additional data quality issues the framework identified independently. The code is public at [github.com/maxwellwilber-cpu/evs](https://github.com/maxwellwilber-cpu/evs).

---

## What This Proves

I'm not an engineer. I have a business degree from Loyola Marymount University. I built this system because I understood the problem from the business side and was willing to learn enough about AI systems to design a solution that's actually reliable.

The hardest part of building AI systems isn't the AI. It's the validation architecture that catches hallucinations, the pipeline design that keeps each component testable, the traceability chain that lets you follow any recommendation back to the raw data that supports it, and the fail-closed behavior that ensures the system never produces output it can't verify.

Anyone can call an LLM API and get a plausible-sounding financial analysis. The difference between a demo and a system is the 73 checks that make it trustworthy.

---

Maxwell Wilber builds AI automations and agents for businesses. He is currently an AI implementation specialist at a Seattle AI startup, and previously built AI systems for small business owners through an independent consulting practice.

[LinkedIn](https://linkedin.com/in/maxwellwilber) · [GitHub](https://github.com/maxwellwilber-cpu)
