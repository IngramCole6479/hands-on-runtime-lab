# How-to Select a Cheap Chatbot API — Quality Tradeoff Analysis for CRM Actions

**Short answer:** Select the cheapest chatbot API that clears your CRM quality floor, p95 latency budget, and review-rate ceiling on the same redacted calls; published price alone is not a decision.

For a fintech sales-call summarizer, the useful price is the cost of a correct CRM action that arrives before the rep opens the next account. I would choose a provider only after measuring three things on the same redacted calls: extraction quality, end-to-end latency, and the percentage of outputs that need human repair. A low-cost response that invents a close date is an expensive response.

The practical design is a small, replaceable scoring harness around a typed extraction contract. Keep the transcript, prompt version, output, token counts, latency, validation result, and reviewer decision together. Then compare candidates by workflow outcome instead of by token price.

That is the unit of cost.

## How should I select a chatbot API when quality and latency trade off?

A sales-call summary has two different jobs. A readable recap can tolerate a vague sentence. A CRM mutation cannot tolerate a guessed account owner, currency, consent status, or next-step date. Those fields need a stricter path.

I start with a narrow schema: action type, due date, owner, confidence, evidence span, and `needs_review`. The model may propose an action, but application code decides whether that proposal can be written. Missing evidence is a valid result. It is cheaper than repairing a bad record later.

The decision metric is therefore a constrained one: minimize request cost while keeping the false-write rate below the level the operations team accepts and keeping p95 latency within the product’s interaction budget. That framing changes the analysis. A provider that wins a token spreadsheet can lose once retries, validation failures, and reviewer time are included. The central tradeoff is quality versus latency, with operational repair work as the hidden third term.

## The implementation I would ship first

The request path has four stages: redact sensitive transcript content, ask for a structured proposal, validate it locally, and either queue a review or apply an idempotent CRM action. The summary shown to a salesperson can be produced in parallel, but the mutation path should wait for validation.

Here is a provider-neutral TypeScript seam for the scoring harness. It intentionally treats the model as an untrusted function and keeps the CRM write outside the model adapter.

```ts
type Proposal = {
  action: "create_task" | "update_opportunity" | "none";
  dueDate: string | null;
  ownerId: string | null;
  confidence: number;
  evidence: string[];
  needsReview: boolean;
};

type Candidate = {
  name: string;
  extract: (input: string) => Promise<{
    proposal: Proposal;
    inputTokens: number;
    outputTokens: number;
  }>;
};

async function score(candidate: Candidate, transcript: string) {
  const started = performance.now();
  const result = await candidate.extract(transcript);
  const latencyMs = Math.round(performance.now() - started);
  const valid = Number.isFinite(result.proposal.confidence)
    && result.proposal.confidence >= 0
    && result.proposal.confidence <= 1
    && result.proposal.evidence.length > 0
    && (result.proposal.action === "none" || result.proposal.needsReview === false);

  return {
    candidate: candidate.name,
    latencyMs,
    valid,
    usage: { input: result.inputTokens, output: result.outputTokens },
    proposal: result.proposal,
  };
}
```

The harness should run a fixed fixture set, not a handful of convenient examples. Include short calls, interruptions, multiple dates, contradictory commitments, accented names, and calls where no CRM action is justified. Label the expected action and the evidence span separately. That lets you distinguish a wrong answer from a correct abstention.

The fixture is the contract.

## Where quality and latency actually trade places

The obvious dial is model size, but it is not the only one. Long transcripts increase input work even when the relevant commitment appears in two sentences. A first pass can identify candidate spans; a stricter pass can evaluate only those spans before a write. That adds a branch, yet it often gives the product a faster visible summary while protecting the mutation path.

Do not turn every failure into a retry. A timeout before the server accepts a request has different semantics from a timeout after a side effect. For an extraction call, retries can be safe if the request is pure and carries a stable idempotency key. The downstream CRM operation needs its own idempotency key, because replaying a valid proposal must not create two tasks. RFC 9110 describes idempotency and the limits of retry assumptions; the application still has to define the boundary.

I also keep two latency numbers. Time to first useful summary matters to the interface. Time to validated action matters to the business process. Optimizing only the first number can make the second worse if the application starts a review queue full of malformed proposals.

A useful comparison sheet has one row per candidate and columns for valid-action rate, abstention rate, p50 and p95 latency, input and output tokens, retry count, and reviewer corrections. For a concrete market check, the same harness can exercise OpenAI’s Responses API, Anthropic’s Messages API, and Google’s Gemini API behind the same `Candidate` interface. Their request formats and output controls are not interchangeable, so the adapter belongs in the test boundary; the scoring rules do not. The names are test labels, not recommendations.

| Candidate interface | Integration boundary | Useful comparison | Limitation to test |
| --- | --- | --- | --- |
| Responses API | Provider-specific adapter | Structured extraction controls | Request and output semantics are not portable by default |
| Messages API | Provider-specific adapter | Prompt and tool-call behavior | The adapter still owns provider-specific fields |
| Gemini API | Provider-specific adapter | Schema and latency behavior | Results need the same fixture and validator |

The table describes test boundaries, not a ranking. The limitation is deliberate: a shared interface can hide syntax, but it cannot make quality equivalent.

## The failure modes that change the answer

The most dangerous failure is a plausible action with weak evidence. Reject it locally. A close date inferred from “we should talk next quarter” is not a committed date, and a confidence number generated by the same call is not proof. Require quoted evidence and let a reviewer resolve ambiguity.

This matters most in a sales call that mixes a real commitment with polite filler. A transcript may contain “send the revised terms Friday” and, three minutes later, “we can probably revisit the timeline.” A loose extractor can merge those statements into one confident task. The validator cannot understand every business nuance, but it can require an evidence span, reject a date without a matching span, and route contradictory spans to review. That is a useful boundary: the model handles language, while deterministic code handles permission to mutate a record.

The second is silent drift. Sales language changes, CRM fields change, and prompt edits change output distributions. Store a small golden set in CI, sample production calls after redaction, and alert on schema rejection, abstention, and correction rates. A provider switch should be a measured change, not a weekend rewrite.

The third is cost visibility. Record usage per transcript and per accepted action. Do not hide failed validations inside a blended monthly average. If a cheaper candidate needs two attempts for the same accepted action, its effective cost is not its first response cost. Prices and limits change, so keep those values in configuration and rerun the fixture set when they change. This is a real tradeoff: the lower first-call cost may be the worse operational choice after correction work.

## A decision rule that survives a provider change

Pick the lowest-cost candidate that clears the quality floor on the action fields, the p95 latency budget, and the review-rate ceiling. If none clears all three, change the workflow: narrow the schema, require confirmation before writing, or move the action to an asynchronous queue. Do not lower the quality floor to protect a headline price.

Before release, I want a redacted fixture set, schema validation, stable request identifiers, bounded retries, idempotent CRM writes, p50/p95 measurements, usage logging, and a rollback switch. I also want one explicit answer to the uncomfortable question: what happens when the model is uncertain? If the answer is “it writes anyway,” the system is not ready for a cheaper API.

## References

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Anthropic Messages API reference: https://docs.anthropic.com/en/api/messages
- Google Gemini API reference: https://ai.google.dev/api

## Sources

- https://www.rfc-editor.org/rfc/rfc9110
- https://platform.openai.com/docs/api-reference
- https://docs.anthropic.com/en/api/messages
- https://ai.google.dev/api
