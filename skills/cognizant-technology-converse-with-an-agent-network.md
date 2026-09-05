---
name: cognizant-technology-converse-with-an-agent-network
description: >-
  Hold a multi-turn conversation with a Neuro SAN agent network over the streaming HTTP API —
  reading the stream correctly, continuing the conversation, and keeping private data out of
  the LLM chat stream.
api: Cognizant Neuro SAN agent service
generated: '2026-09-05'
method: generated
source: >-
  openapi/cognizant-technology-neuro-san-agent-service.json (operationIds verified against the
  spec) + grpc/cognizant-technology-agent.proto + grpc/cognizant-technology-chat.proto +
  conventions/cognizant-technology-conventions.yml
operations:
  - AgentService_StreamingChat
  - AgentService_Function
---

# Converse with a Neuro SAN agent network

`AgentService_StreamingChat` — `POST /api/v1/{agent_name}/streaming_chat`

This is the operation that does the work. Everything else in the API is discovery.

Run `AgentService_Function` first if you have not already — see the discovery skill — so you
know what the network expects on each of its two channels.

## Build the request

`ChatRequest` has four members:

- `user_message` — a `ChatMessage`. Set `type: HUMAN` and put your text in `text`.
- `chat_context` — omit on the first turn. On every later turn, see continuation below.
- `chat_filter` — optional. `chat_filter_type` accepts `MINIMAL` or `MAXIMAL` (or `UNKNOWN`)
  and controls how much of the internal message traffic is streamed back to you. Use
  `MINIMAL` when you only want the answer and `MAXIMAL` when you are debugging.
- `sly_data` — the private channel. See below.

## Read the stream correctly

This is a server-streaming operation, and there are three rules that are easy to get wrong:

1. **The answer is in the LAST streamed `AGENT_FRAMEWORK` message.** Not the first message,
   and not necessarily the final message of the stream. Buffer and take the last one.
2. **Each response is on exactly one line.** The contract guarantees a single response is
   never split across lines in the HTTP response, so line-delimited parsing is safe.
3. **`ChatMessage.type` tells you what you are looking at**: `SYSTEM`, `HUMAN`, `AI`, `AGENT`,
   `AGENT_FRAMEWORK`, `AGENT_TOOL_RESULT`, `AGENT_PROGRESS`, `UNKNOWN`. Messages of type `AI`
   are the front-man agent answering on behalf of the rest of the network — the ones a user
   most wants to see. `AGENT_PROGRESS` is for progress display, not for the final answer.

Each message also carries `origin[]`, an array of `Origin` objects with `tool` and a 0-based
`instantiation_index`. That is your tracing signal: it reconstructs the exact path a message
took through the network, and the index distinguishes repeat invocations of the same tool
reached by different paths. There is no request-id header on this API, so `origin[]` is what
you have.

## Continue the conversation

The last `AGENT_FRAMEWORK` message before the stream closes has its `chat_context` populated.

Copy that whole structure, unmodified, into the `chat_context` of your next `StreamingChat`
request. That is the entire continuation mechanism — there is no session id, no conversation
id and no cookie. Drop it and you start a fresh conversation.

## Keep private data out of the chat stream

`sly_data` is a map that is deliberately excluded from the LLM chat stream. The keys may be
referenced in the stream; the values are handed to tools programmatically and never enter it.

Put credentials, logins, tokens and session ids here — not in `user_message.text`. This is
also how a client supplies its own LLM-provider API key under the framework's
bring-your-own-key support, which is why the key never appears as a header or a
`securityScheme`.

Check `Function.sly_data_schema` for what the network expects, and
`Function.sly_data_output_schema` for what it will return on the same channel.

## Before you call: two safety facts

**There is no replay protection.** No `Idempotency-Key`, no request de-duplication, no
at-most-once guarantee. Calling this twice starts two conversations and bills the operator's
LLM provider twice. If a request times out you cannot tell whether the first one landed — so
decide deliberately whether to retry rather than retrying by reflex.

**A chat turn can have irreversible side effects.** The API itself creates no resource you can
delete, and there is no cancel, undo or rollback operation — but agent networks call
CodedTools, which the provider documents as able to "effectuate change via a web API". Those
effects live in third-party systems, and neither neuro-san nor this contract tracks or
reverses them. Read the network's `description` and `parameters` before invoking one you do
not control.

## Errors

One `default` response, carrying `Status` (`code`, `message`, `details[]`) — the
`google.rpc.Status` model, not RFC 9457. No 429 is declared, so throttling is not
distinguishable from any other failure without reading `code` from the body.

## The MCP alternative

If the deployment runs with `--mcp_enable=true`, the same capability is reachable as an MCP
tool named after the agent network, called via `tools/call` at `POST /mcp` with protocol
version `2025-06-18`. One behavioural difference matters: **MCP does not stream.** It returns
a single JSON-RPC payload with the content blocks and an `isError` boolean, so you get the
answer without the intermediate message traffic — and without `origin[]` tracing.
