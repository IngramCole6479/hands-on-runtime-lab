# LLM Code Review Streaming: Measure First Useful Output Across Portable Chat Runtimes

A fast first token helps a chat feel responsive, but a code review needs a finding the application can actually display. TL;DR: choose a streaming runtime by measuring both the first visible text and the first validated finding on representative diffs. Keep the review contract independent of the provider so that an early fragment does not decide an otherwise expensive migration.

For a B2B SaaS chatbot reviewing code changes, the useful milestone is concrete: a file, a line, a severity, and an explanation that survive validation. Treat that milestone as a separate application measurement. It answers a different question from time to first token, and it gives a solo builder a practical way to weigh responsiveness against structured output and provider portability. No universal fastest runtime follows from the evidence here. That decision requires comparable measurements on the actual review workload.

## How should you measure latency for streaming LLM chat responses?

The application receives a diff, passes it to a provider adapter, consumes normalized text fragments, and assembles complete finding records before showing them in the chat. The adapter owns provider-specific stream decoding and termination checks. The review layer owns validation, timing, and the decision to publish a provisional finding. This is a proposed application contract: it does not assume that different providers expose the same wire format or support the same structured streaming features.

Count useful work.

Define the clocks before comparing candidates. Start the request clock immediately before invoking the adapter. Record the first nonempty normalized fragment, then the first complete finding accepted by the validator, and finally successful completion. Call the first measurement application-observed first-text latency. It includes the adapter and its buffering; it is not a measurement of the model's internal first token. If the browser receives the finding through another service, also measure when that finding becomes visible there.

## Implement the finding boundary before selecting the runtime

The following TypeScript example runs against a local fixture and prints a finding with timing fields. It needs an environment that can execute TypeScript and supplies the standard Performance API. The fixture deliberately splits one record across two chunks; its timings are demonstration output, not performance evidence. The line-delimited JSON format belongs to this application. A production adapter must explicitly implement that contract.

```ts
type Finding = {
  path: string;
  line: number;
  severity: "warning" | "error";
  message: string;
};

type ReviewInput = { diff: string };
type Adapter = (input: ReviewInput) => AsyncIterable<string>;

function parseFinding(record: string): Finding {
  const value: unknown = JSON.parse(record);
  if (typeof value !== "object" || value === null) {
    throw new Error("Finding must be an object");
  }
  const item = value as Record<string, unknown>;
  const { path, line, severity, message } = item;
  if (
    typeof path !== "string" || path.trim() === "" ||
    typeof line !== "number" || !Number.isInteger(line) || line < 1 ||
    (severity !== "warning" && severity !== "error") ||
    typeof message !== "string" || message.trim() === ""
  ) {
    throw new Error("Invalid finding fields");
  }
  return { path, line, severity, message };
}

async function review(adapter: Adapter, input: ReviewInput) {
  const started = performance.now();
  let firstTextMs: number | null = null;
  let firstFindingMs: number | null = null;
  let pending = "";
  const findings: Finding[] = [];
  const maxRecordChars = 32_768; // An application limit, not a provider limit.

  for await (const chunk of adapter(input)) {
    if (chunk.length > 0 && firstTextMs === null) {
      firstTextMs = performance.now() - started;
    }
    pending += chunk;
    let boundary: number;
    while ((boundary = pending.indexOf("\n")) !== -1) {
      if (boundary > maxRecordChars) {
        throw new Error("Finding exceeds the record limit");
      }
      const record = pending.slice(0, boundary);
      pending = pending.slice(boundary + 1);
      if (record.trim() === "") continue;
      const finding = parseFinding(record);
      firstFindingMs ??= performance.now() - started;
      findings.push(finding);
    }
    if (pending.length > maxRecordChars) {
      throw new Error("Incomplete finding exceeds the record limit");
    }
  }
  if (pending.trim() !== "") {
    throw new Error("Stream ended before the record delimiter");
  }
  return {
    findings,
    firstTextMs,
    firstFindingMs,
    completedMs: performance.now() - started,
  };
}

const fixture: Adapter = async function* (_input) {
  const record = JSON.stringify({
    path: "src/review.ts",
    line: 12,
    severity: "warning",
    message: "Validate the line number before displaying the finding.",
  }) + "\n";
  yield record.slice(0, 23);
  yield record.slice(23);
};

review(fixture, { diff: "Example diff supplied by the review application" })
  .then(result => console.log(result))
  .catch(error => { console.error(error); });
```

The parser waits for a newline before calling JSON.parse.

That boundary matters.

JSON.parse accepts JSON text; it does not turn an arbitrary partial fragment into a complete object. A finding may span multiple chunks, while one chunk may contain multiple findings. The loop handles both cases. The record limit is an illustrative guard against an indefinitely growing unfinished record, measured in JavaScript string length rather than network bytes. A production deployment also needs limits on total input, total output, and finding count. Consider the fixture's split after character 23: the first fragment is available to the adapter, yet the parser cannot accept a finding from it. Publishing that fragment as a finding would move parsing responsibility into the interface and leave the user looking at an unfinished object. Waiting for the delimiter adds a publication boundary, but keeps incomplete fields out of the review UI. That's the trade-off this example makes explicit.

The example collects validated findings so it can stay focused on the boundary. For incremental display, invoke an application callback immediately after validation, marking the finding provisional until the request completes successfully. The demo checks field shape only. Before publishing a real review, verify that the path belongs to the submitted diff and that the line is a valid review location. Neither check proves the finding is correct.

## An earlier fragment can hide a later review

Imagine two candidate responses to the same diff. One begins with introductory prose; the other begins with a finding record. The first visible character and the first actionable review can favor different candidates. This is a hypothetical test case, not a benchmark result. It is useful because it exposes a decision error: selecting a runtime from a metric that the feature does not actually use.

For the first release, request bounded findings without an introductory paragraph. That reduces the amount of nonessential output the application asks for. The trade-off is less conversational context, which may matter if users expect an explanation of the review's scope. Put that context in the interface when it is fixed application text; keep generated explanations attached to individual findings. Token consumption should be measured alongside valid results, especially when malformed output triggers another generation. Unit price alone cannot describe that workflow.

There is another cost.

A requirement to emit independently parseable records becomes part of the adapter's acceptance criteria. Some candidate integrations may need to wait for a complete response before they can produce validated records. Test that behavior explicitly. Such an adapter can preserve the finding contract while delivering a different latency profile, so portability should mean stable application semantics rather than an assumption of identical timing.

Keep the adapter small enough to replace. It should decode text, interpret provider completion or failure signals, and propagate cancellation; it should not decide which source-code lines are reviewable. If the underlying transport delivers bytes, perform incremental text decoding before handing strings to the record parser. The Streams and Encoding specifications describe those primitives. Transport chunks are not application record boundaries.

## Compare complete review outcomes on a fixed workload

Use the same diffs, review instructions, output limits, and acceptance checks for each candidate. Include a short change, a larger change, a change with no expected findings, and a deliberately interrupted response. Those are proposed fixtures, not claims about any provider. Version them with the application so a later prompt change does not silently invalidate the comparison.

Report distributions for first text, first validated finding, and completion, together with failure counts and output consumption. Keep requests without a finding in a separate outcome category. A successful review with zero findings has no first-finding timestamp; treating it as zero milliseconds would distort the result. An interrupted review is different again. Preserve that distinction in telemetry.

For a small team, the useful dashboard can be modest: request identifier, adapter revision, workload category, the three timing fields, finding count, and terminal outcome. Avoid recording raw customer diffs merely to diagnose latency. Record enough metadata to reproduce an authorized test, and use controlled fixtures to investigate parser behavior. Browser-visible timing needs its own measurement because server completion does not establish when the user saw the result.

Reject invalid reviews first.

Then compare useful-result latency and resource consumption among the remaining candidates. Set acceptance thresholds from the product's needs. The sample's constants are not service targets.

## Ship with explicit completion and cancellation rules

Before rollout, confirm that an adapter throws on provider failure or incomplete termination instead of making an interrupted response look like success. A final newline proves that the last record is delimited; it does not prove that the provider finished the requested review. The application needs both checks. Preserve already displayed findings as provisional when completion fails, and expose the incomplete status to the user.

Then verify cancellation from the chat interface through the adapter, including the case where the user leaves while a record is unfinished. Set a request deadline, bound accumulated records, and decide how a retry replaces provisional output. Automatically retrying after findings are visible can create confusing duplicates unless the application explicitly replaces or reconciles them.

Finally, exercise the replacement adapter against the same fixtures before deployment and watch completion outcomes during rollout. Choose on the basis of valid review results delivered within the application's latency budget. The first token remains a useful signal. The first usable finding is the milestone this feature should optimize.

## Sources

References for the parsing, timing, and streaming primitives used in this engineering note:

- JSON.parse behavior: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- High Resolution Time specification: https://www.w3.org/TR/hr-time-3/
- Streams Standard: https://streams.spec.whatwg.org/
- Encoding Standard, including streaming text decoding: https://encoding.spec.whatwg.org/
