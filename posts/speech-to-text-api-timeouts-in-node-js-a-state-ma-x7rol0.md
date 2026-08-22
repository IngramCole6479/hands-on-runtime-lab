# Speech to Text API Timeouts in Node.js: A State Machine for Long Audio

Long audio transcription is a state-tracking problem before it is a timeout problem. **Short answer: measure which part of the request was reached, reject an oversized file before multipart allocation, and retry only a response that the service contract identifies as transient.** That gives a Node.js `fetch` client a useful next action instead of a larger number on one timer.

This matters for a private knowledge-base assistant. A recording can be accepted by the upload path, never reach inference, or finish inference while the client is no longer waiting. Those states look similar from a browser spinner. They are not similar to the worker that must decide whether another upload is safe.

## What does a speech to text API timeout actually tell you?

Almost nothing by itself. The word timeout describes the observer, not the remote state.

I split one transcription operation into five local states: `preflight`, `uploading`, `accepted`, `transcribing`, and `complete`. A sixth state, `ambiguous`, covers a client abort after bytes may have reached the service. The state is not a claim about what the remote system did; it is a record of the last evidence the client had.

That distinction makes logs actionable. A file-size rejection is a product and input decision. A connection that ends during `uploading` belongs to the transport path. A response after the body was sent belongs to the service contract. An aborted request after the server may have accepted the body is a replay decision. Imagine a long recording whose progress reaches the end of the local stream, followed by a client-side timeout before a response arrives. The tempting log line is `timeout`; the useful one is `state=ambiguous bytes=... attempt=1 deadline=...`. That record tells the worker to stop automatic replay until it knows whether the remote operation can be looked up or safely deduplicated. Now compare it with a file rejected by the byte gate: no multipart body was built, no network call happened, and the correct metric is an input rejection rather than a failed transcription. Those two paths may take the same amount of wall-clock time in a test, but they require opposite recovery behavior.

For each operation, log a non-secret operation ID, byte count, attempt number, client deadline, last local state, response status when present, and whether the request body finished. Do not log the audio or credentials. Keep the same operation ID for all attempts; changing it turns one uncertain operation into several unrelated-looking jobs.

Three words: accepted is not complete.

## How should Node.js fetch handle multipart file size, long audio, and retry backoff?

Put the byte gate before `readFile` and before `FormData`. The ceiling must come from the selected transcription service and the narrowest proxy in your deployment. There is no universal multipart limit that a Node.js client can safely invent. A check against the wrong limit is false confidence, so keep the value in configuration and test a file near it.

Use two budgets. The attempt deadline covers one upload-and-response try. The operation budget covers retries and waits. A 20-second attempt repeated three times is already 60 seconds before backoff, and that may be unacceptable for an interactive knowledge-base workflow. I am not sure what deadline fits your hosting path; measure upload time with representative recordings, including the slowest network you actually support. Your mileage may vary when a reverse proxy buffers the body.

The sample deliberately leaves the URL, multipart field, and file-size ceiling outside the code. Those values are part of a particular service's contract. It also avoids treating every `fetch` exception as permission to resend bytes. A network exception can mean the request was never sent, or that the response was lost after acceptance.

```ts
import { readFile, stat } from "node:fs/promises";
import { basename } from "node:path";

type LocalState = "preflight" | "uploading" | "accepted" | "complete" | "ambiguous";

const [audioPath] = process.argv.slice(2);
const url = process.env.TRANSCRIPTION_URL;
const apiKey = process.env.TRANSCRIPTION_API_KEY;
const fileField = process.env.TRANSCRIPTION_FILE_FIELD;
const maxBytes = Number(process.env.MAX_AUDIO_BYTES);
const attemptLimit = Number(process.env.ATTEMPT_LIMIT ?? "3");
const attemptTimeoutMs = Number(process.env.ATTEMPT_TIMEOUT_MS ?? "20000");

if (
  !audioPath ||
  !url ||
  !apiKey ||
  !fileField ||
  !Number.isSafeInteger(maxBytes) ||
  maxBytes <= 0 ||
  !Number.isSafeInteger(attemptLimit) ||
  attemptLimit < 1 ||
  !Number.isSafeInteger(attemptTimeoutMs) ||
  attemptTimeoutMs < 1
) {
  throw new Error("Set the transcription URL, credentials, file field, and positive limits");
}

const metadata = await stat(audioPath);
if (metadata.size > maxBytes) {
  throw new Error(`Audio is ${metadata.size} bytes; the configured ceiling is ${maxBytes}`);
}

const audio = await readFile(audioPath);
const operationId = `${metadata.size}-${metadata.mtimeMs}`;
let state: LocalState = "preflight";

for (let attempt = 0; attempt < attemptLimit; attempt += 1) {
  const form = new FormData();
  form.set(fileField, new Blob([audio]), basename(audioPath));
  state = "uploading";

  try {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": operationId,
      },
      body: form,
      signal: AbortSignal.timeout(attemptTimeoutMs),
    });

    if (response.ok) {
      state = "accepted";
      const result = await response.text();
      state = "complete";
      process.stdout.write(`${result}\n`);
      break;
    }

    const detail = await response.text();
    const retryable = response.status === 429 || response.status >= 500;
    if (!retryable || attempt === attemptLimit - 1) {
      throw new Error(`Transcription stopped in ${state} (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  } catch (error) {
    if (state === "uploading") {
      state = "ambiguous";
    }
    throw error;
  }
}
```

The `Idempotency-Key` header in this example is a contract boundary, not a guarantee supplied by HTTP itself. Only send it when the chosen service documents the header and its replay semantics. The file metadata expression is merely a local example; a production job should derive its identifier from an immutable upload record, not assume that two files with the same size and modification time are the same audio.

There is a catch. Aborting `fetch` stops waiting in this process; it does not prove that a remote write was undone. If idempotent replay is not documented, store `ambiguous` and require reconciliation through the service's documented operation lookup or a human-visible retry decision. A blind second multipart upload can create duplicate work.

## Which responses deserve another attempt?

Retry classification should be based on evidence, not optimism. A `429` is a rate-limit signal and may carry `Retry-After`; honor it within the operation budget. A transient server response can be retried with bounded exponential delay. An input that exceeds the configured file ceiling, an invalid multipart shape, unsupported audio, or an unavailable capability will not become valid because the loop ran again.

Treat errors as data. Keep the response category separate from the local state:

| Evidence | Local action | User-facing result |
| --- | --- | --- |
| File is above the configured ceiling | Stop before allocation | Ask for a shorter recording or another ingestion path |
| No response while uploading | Mark the attempt ambiguous | Reconcile before replaying when semantics are unknown |
| `429` with `Retry-After` | Wait within the operation budget | Keep the job pending, with a visible deadline |
| Transient server response | Retry a bounded number of times | Preserve the operation ID and attempt history |
| Invalid input or unsupported capability | Stop retrying | Explain the constraint and offer a configured fallback |
| Successful response | Validate the returned transcript state | Mark complete only after usable text is present |

The exact status categories are service-specific, so map them from the current contract rather than copying this table blindly. A useful adapter exposes categories such as `rate_limited`, `transient`, `invalid_input`, `unsupported`, and `ambiguous`; the rest of the application should not need to know how one provider spells its multipart fields.

## What should be tested before shipping the transcription worker?

Build a small corpus that exercises boundaries, not just a clean demo clip: a short recording, a normal long recording, a file just below the configured byte ceiling, one just above it, and an upload interrupted at different points. For each case, assert the local state sequence, number of attempts, backoff behavior, and whether a duplicate could be created.

For a private knowledge base, correctness has a second edge. A transcript that arrives with a successful HTTP status can still be unusable if the worker has not validated the response shape or if the indexing step has not completed. Keep transcription completion separate from chunking, embedding, and retrieval indexing. That lets a retry repair the right stage instead of resending the original audio.

Watch four measurements in production: bytes rejected before upload, time spent uploading, time from acceptance to usable text, and the count of ambiguous operations. Add the operation ID to each stage so a dashboard can show one timeline. This is more useful than a single p95 for “transcription latency,” which hides where the budget went.

The catch is operational scope. This policy is not suitable when the product cannot tolerate a reconciliation queue, when the selected service offers no documented replay semantics, or when recordings routinely exceed the deployment's memory budget. In those cases, keep the audio in a durable upload path and use the service's documented asynchronous workflow, or choose a self-hosted pipeline whose storage and job states you can operate. Stick with the simpler synchronous request only for files and latency targets your tests actually cover.

## Sources

- https://docs.cohere.com/docs/rerank-overview
- https://www.promptingguide.ai
