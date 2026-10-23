---
layout: post
title: "Tracing an AI agent loop in Node.js: tool calls, policy verdicts and token spend as spans"
date: 2026-10-23 09:00:00 +0000
tags: [nodejs, opentelemetry, ai-agents, vercel-ai-sdk, observability]
---

An agent turn is not one request. In [Chaser](https://github.com/vigohan95/chaser), an accounts-receivable agent on PayPal, a single "run the after-final review" message became four model calls, one tool that spent 3.6 s talking to a news API, three policy verdicts and a pause for a human. Timing the HTTP request that carried it tells you it took 10.0 s. It does not tell you which of those parts to fix, or that the same message had taken 23.0 s two minutes earlier. This post wires OpenTelemetry into that loop with the Vercel AI SDK 7, adds the two spans the SDK cannot produce on its own, and reads the numbers that came out.

Everything below ran against the demo data with NVIDIA Nemotron 3 Nano (30B-A3B) on Nebius Token Factory on 2026-10-23. The script that produced the traces is in the repo as `scripts/trace-smoke.ts`; the JSON dumps are what the tables quote.

## The loop under test

Chaser's agent is a `generateText` / `streamText` call with 14 tools and a `toolApproval` hook. The hook is the policy engine: for every tool call the model proposes, it reads the current invoices and disputes, evaluates a written policy and answers `approved`, `denied` or `user-approval`. The last answer pauses the run until a person clicks Approve in the UI, which arrives as a second HTTP request.

```ts
// src/lib/agent/run.ts (trimmed)
export function agentSetup(opts: { unattended?: boolean } = {}) {
  const store = getStore();
  return {
    model: getModel(),
    tools: buildTools(),
    system: systemPrompt(store.policy, currentMode()),
    stopWhen: stepCountIs(14),
    toolApproval: async ({ toolCall }) => {
      const [invoices, disputes] = await Promise.all([getInvoices(), getDisputes()]);
      const ev = evaluateAction(toolCall.toolName, toolCall.input, { policy: store.policy, invoices, disputes });
      if (ev.decision === "deny") return { type: "denied", reason: ev.reason };
      if (ev.decision === "approve") return { type: "user-approval", reason: ev.reason };
      return { type: "approved", reason: ev.reason };
    },
  };
}
```

Three things in there cost time or money and are invisible from outside: the model calls (tokens), the tools (external APIs), and the gate (a database read plus a human, sometimes).

## Registering a tracer: 20 lines

AI SDK 7 exposes a `Telemetry` interface with lifecycle callbacks (`onStart`, `onStepStart`, `onLanguageModelCallEnd`, `onToolExecutionEnd`, ...) and ships an implementation of it that emits OpenTelemetry spans, `@ai-sdk/otel`. You register it once per process.

```ts
// scripts/trace-smoke.ts (setup)
import { trace } from "@opentelemetry/api";
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { InMemorySpanExporter, SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { OpenTelemetry } from "@ai-sdk/otel";
import { registerTelemetry } from "ai";

const exporter = new InMemorySpanExporter(); // swap for an OTLP exporter in production
const provider = new NodeTracerProvider({ spanProcessors: [new SimpleSpanProcessor(exporter)] });
provider.register();
registerTelemetry(new OpenTelemetry({ tracer: trace.getTracer("ai-sdk"), usage: true }));
```

`usage: true` adds the SDK's own token details on top of the GenAI semantic conventions. Nothing in the agent code changes. With the provider registered, one chase turn produces this tree (durations in ms, from the exported spans):

```
invoke_agent nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B   5405   in=47387 out=520  finish=stop
  step 1                                             2295
    chat nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B       2273   in=22125 out=17   finish=tool-calls
    execute_tool propose_collections                   17   tool=propose_collections
  step 2                                             3098
    chat nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B       3098   in=25262 out=503  finish=stop
```

The `chat` spans carry `gen_ai.request.model`, `gen_ai.response.id`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` and, because Token Factory reports it, `gen_ai.usage.cache_read.input_tokens`. The `execute_tool` spans carry `gen_ai.tool.name`, `gen_ai.tool.call.id`, the arguments and the result. By default the spans also record the system prompt, the tool definitions and every message; that is a lot of bytes per span (the system prompt alone is a few thousand tokens here), so in production pass `recordInputs: false` and `recordOutputs: false` in the call's `experimental_telemetry` options and keep the ids.

## What the SDK cannot see, part 1: the policy verdict

The SDK traces the tool *execution*. It does not trace the decision about whether to execute, because that is your code in `toolApproval`. Wrapping it is a dozen lines:

```ts
// src/lib/telemetry.ts
import { SpanStatusCode, trace, type Attributes, type Span } from "@opentelemetry/api";

export async function withSpan<T>(name: string, attributes: Attributes, fn: (span: Span) => Promise<T>): Promise<T> {
  return trace.getTracer("chaser").startActiveSpan(name, { attributes }, async (span) => {
    try {
      return await fn(span);
    } catch (e) {
      span.recordException(e as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: e instanceof Error ? e.message : String(e) });
      throw e;
    } finally {
      span.end();
    }
  });
}
```

```ts
// src/lib/agent/run.ts, inside toolApproval
const ev = await withSpan(
  "chaser.policy.decide",
  { "chaser.tool": toolCall.toolName, "gen_ai.tool.call.id": toolCall.toolCallId },
  async (span) => {
    const [invoices, disputes] = await Promise.all([getInvoices(), getDisputes()]);
    const verdict = evaluateAction(toolCall.toolName, toolCall.input, { policy: store.policy, invoices, disputes });
    span.setAttributes({ "chaser.policy.decision": verdict.decision, "chaser.policy.rule": verdict.rule ?? "", "chaser.policy.summary": verdict.summary });
    return verdict;
  },
);
```

One detail bit me immediately: these spans came out as roots, not as children of the step. The SDK invokes `toolApproval` outside the active span context it uses for steps and tool executions, so `startActiveSpan` has no parent to attach to. Rather than fight the context, the span carries the same `gen_ai.tool.call.id` the SDK puts on its `execute_tool` span, and the backend joins them. Here is the after-final turn with the policy spans:

```
invoke_agent nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B   9997   in=92013 out=390
  step 1                                             4622
    chat ...                                          977   in=22120 out=17   finish=tool-calls
    execute_tool propose_after_final                 3643
  step 2                                             1466
    chat ...                                         1465   in=23129 out=58   finish=tool-calls
  step 3                                             1233
    chat ...                                         1232   in=23308 out=67   finish=tool-calls
  step 4                                             2668
    chat ...                                         2668   in=23456 out=248  finish=stop
chaser.policy.decide   0   tool=propose_after_final  policy=allow (read-only)
chaser.policy.decide   1   tool=resolve_after_final  policy=approve (collections.afterFinal.writeOffUnder)
```

`approve` is the engine's word for "a human must approve". The verdict itself costs under a millisecond; the tool it gates never runs in this unattended script, which is why step 3 has a `chat` span and no `execute_tool` child.

## What the SDK cannot see, part 2: the human

When the verdict is `user-approval`, the run stops, the UI shows a card, and the user's click arrives in a new request. No span can stay open across that. What you can do is reconstruct it: the policy writes an audit entry with a timestamp when it asks, and the request that carries the answer can start a span at that timestamp and end it now.

```ts
// src/lib/telemetry.ts
export function approvalWaitSpan(request: AuditEntry | undefined, response: { tool: string; approved: boolean; rule?: string }) {
  if (!request) return undefined; // no request entry, no span: a guessed start time is worse than none
  const start = Date.parse(request.ts);
  const end = Date.now();
  const span = trace.getTracer("chaser").startSpan("chaser.approval.wait", {
    startTime: start,
    attributes: {
      "chaser.tool": response.tool,
      "chaser.approval.decision": response.approved ? "approved" : "denied",
      "chaser.approval.wait_ms": end - start,
      ...(response.rule ? { "chaser.policy.rule": response.rule } : {}),
    },
  });
  span.end(end);
  return end - start;
}
```

```ts
// src/app/api/agent/route.ts, when a tool part arrives in state "approval-responded"
const key = JSON.stringify(part.input);
const request = store.audit.find((a) => a.decision === "approval-required" && a.tool === tool && JSON.stringify(a.input) === key);
const waitMs = approvalWaitSpan(request, { tool, approved: part.approval.approved, rule: ev.rule });
```

`startSpan` with an explicit `startTime` is the whole trick; the OpenTelemetry API accepts epoch milliseconds there. In a hosted session the day before, the hand-off approval was requested at 02:20:06 and clicked at 02:20:22: a 16 s span, which in that run was longer than every model call in the turn put together.

## Reading the numbers

Two runs of the same two prompts, two minutes apart, with totals computed over the `chat` spans only (the `invoke_agent` span repeats the sum of its steps, which doubled my first total):

| run | turn | steps | wall | model time | tool time | input tokens | output tokens |
|---|---|---|---|---|---|---|---|
| 1 | chase | 3 | 6.6 s | 6.6 s | 0.02 s | 73,143 | 525 |
| 1 | after-final | 5 | 23.0 s | 9.1 s | 13.9 s | 115,693 | 515 |
| 2 | chase | 2 | 5.4 s | 5.4 s | 0.02 s | 47,387 | 520 |
| 2 | after-final | 4 | 10.0 s | 6.3 s | 3.6 s | 92,013 | 390 |

Three things fall out that the final answer hides.

**Input tokens are the bill, and they are mostly the same bytes.** Every step re-sends the system prompt (the policy JSON is embedded in it) and all 14 tool definitions: 22,120 to 25,756 input tokens per step against 17 to 503 output tokens. Token Factory's cache helps, `gen_ai.usage.cache_read.input_tokens` was 12,672 of 22,125 on the first step, but 139,400 input tokens for two turns is still the number to attack: trim tool descriptions, or hand the model fewer tools per turn with `activeTools`.

**The slow part of the after-final turn is not the model.** `propose_after_final` runs one news search per client before it proposes anything; that tool took 13.9 s in run 1 and 3.6 s in run 2, while no single model call exceeded 3.2 s. Without the tool span the obvious diagnosis would have been "the 30B model is slow". It was the third-party API, and the fix is a cache keyed by client.

**Runs differ, and the trace is the evidence.** Run 1 took five steps, run 2 took four, and in run 2 the model first called `accept_dispute_claim` with an invoice id and the note "Write off". That call failed input validation (the note schema allows no spaces), so it produced a `chat` span with a `tool_call` in its output and no `execute_tool` child, no policy span, nothing executed; the model then called the right tool. The final text in both runs reads as if the policy was followed cleanly. Keep the traces from a known-good run and diff step counts and tool names against them; that is a regression test for agent behaviour that costs nothing extra.

## Production notes

- In Next.js, register the provider from `instrumentation.ts` (or `@vercel/otel`) and call `registerTelemetry` there; the agent code stays as above. Without a registered provider `@opentelemetry/api` returns no-op tracers, so the spans in `run.ts` and the route cost nothing in environments that do not export.
- Sample by run, not by span: a parent-based sampler keeps a turn's steps together. Decide on the `invoke_agent` span.
- Keep `gen_ai.tool.call.id` on anything you add; it is the join key the SDK already gives you.
- Record token usage even if you drop inputs and outputs. The usage attributes are small and they are where the money is.

Package versions: `ai` 7.0.150, `@ai-sdk/otel` 1.0.150, `@opentelemetry/sdk-trace-node` 2.12.0, Node 24. The full script, the agent loop and the span helpers are in [vigohan95/chaser](https://github.com/vigohan95/chaser) under MIT.
