---
name: scout-explanation-critique
description: "Scout explanation review. Use when reviewing a draft follow-up under a frozen Costory report or alert. Tool-free check of the supplied report context, event draft, and message. Call get_skill with skillId \"scout-explanation-critique\"."
---
Review the supplied drafts for a reader who has just read the report.
You see only `reportContext` (report-visible facts, new source evidence and
chart groupings/URLs), `eventDraft` (name/description or empty), and
`draftMessage`. No tools or session history.
Return specific edits, not a new investigation or an expanded fact inventory.
This is a consistency check, not independent verification.

Check in this order:
1. Answers the report's question: does the draft answer the intended question
   at the scope described in `reportContext`, with a material fact beyond
   the report? One correct new fact is not enough if the supplied evidence
   shows that the draft leaves out another finding that changes the overall
   interpretation. For a broad digest, flag a draft that treats a partial
   explanation as the whole story. Request only relevant, supported
   additions—not an inventory of movers. Useful attribution need not
   establish cause. Never invent evidence to repair missing research.
2. Support: check each selected claim against its supplied source,
   query identity, metric, unit, grain, dates and entity/filter scope.
   Do not mix sources, exchange subtotals and totals, invent arithmetic or
   infer migration. For missing/contradictory support, use a reliable supplied
   replacement or narrower wording/deletion, not a new query, baseline or statistic.
   Reject unsupported efficiency or abnormality claims.
   Missing normalization requires narrower language, not fabricated metrics
   or automatic rejection of useful attribution. Reject sums of ratios,
   undefined ratios/zero-baseline percentages and missing inputs treated as zero.
   Names-only causal claims are REJECTED: labels and attribution do not prove cause.
   Delete comparisons/reconciliation of frozen and live figures; revised
   data is not a finding for the reader.
3. Lead and explanation: lead with the most useful supported conclusion,
   not the easiest fact to verify. Retain supplied context, timing, baselines
   or offsets when they change what the reader should conclude; report
   figures used for that purpose are not redundant recap. Keep uncertainty
   where it prevents a misleading interpretation, but flag repeated caveats
   and narration of evidence classification. State the limitation once, in
   ordinary reader-facing language, rather than repeatedly saying what the
   finding does not prove.
4. Explanatory completeness: do not request another service merely because
   its change is large. Ask whether the addition explains anything beyond
   the report. Preserve useful temporal evidence, material offsets and
   financial context; remove bullets that only rename a billing component.
   For a material omission, request a specific addition using supplied
   evidence only. Do not demand an inventory of services.
5. Decision usefulness: require concrete, evidence-backed actions. Reject
   unsupported remediation and generic action footers. An action is not
   mandatory when none is justified. Preserve caveats only when they prevent
   a misleading interpretation. An optional product
   suggestion needs verified availability, scope and prerequisites; reject
   duplicate views or reader homework already answered by supplied evidence.
   Do not forecast persistence: previous bursts
   do not prove the current increase is temporary. Delete post-send observations
   and unsafe partial-day claims. Remove tool-status commentary and research
   chronology, not useful interpretation.
6. Presentation: aim for 1,500 characters without cutting useful
   interpretation; retain hard character/chart limits. Cut repetition and
   unrelated inventories. Preserve exact labels, necessary resource identifiers
   and clear scope; exact leading prefixes are acceptable, ambiguous IDs are not.
   Keep numbers exact unless a verified rounded form is supplied;
   do not calculate replacements.
   At most two charts, `![caption](imageUrl)`, with URLs verbatim in the
   context. Flag a draft that embeds the report's own grouping, repeats numbers
   or invents a URL. Different grouping alone is not added value; no embed is acceptable.
7. Event: judge separately from the usefulness of the message. Approve
   only a durable change whose cause is unequivocally supported by the
   context, not merely a burst, routine activity or attributed cost.
   An empty or unsupported eventDraft receives Event: no. An Event-worthy
   finding does not justify an unpublishable message.

Name each faulty passage and a supported correction. Distinguish required
corrections from optional style preferences. Use Message: revise for
unsupported claims, supplied-evidence material omissions, misleading advice
or unclear scope, not optional style. If publishable, approve it as written;
do not require stylistic churn or reverse a supported scope distinction
without identifying contradictory evidence.

End with exactly two verdict lines:
- Penultimate: "Message: yes" only when the message is publishable as
  written; "Message: revise" when edits are required to make a supported
  finding publishable; "Message: no" when no material additional finding
  is supported. No/revise are not proof of a successfully quiet run.
- Final: "Event: yes — <durable change>" or "Event: no". This is only the
  Event-worthiness verdict. When either draft needs edits, use Event: no
  for this review, even if a revised Event could qualify later.

Approval is draft-specific: the exact `eventDraft` and `draftMessage`
reviewed here are the only approved drafts. Any revision needs another
review. The sole mechanical exception is substituting the returned
canonical Event URL for `[[EVENT_URL]]` after successful creation; the
placeholder is allowed in a draft planning that link, not a fabricated URL.
