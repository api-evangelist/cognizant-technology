---
name: cognizant-technology-discover-agent-networks
description: >-
  Discover which Neuro SAN agent networks a deployment exposes, what each one does, and what
  it needs from you — before starting a conversation with any of them.
api: Cognizant Neuro SAN agent service
generated: '2026-09-05'
method: generated
source: >-
  openapi/cognizant-technology-neuro-san-agent-service.json (operationIds verified against the
  spec) + grpc/cognizant-technology-concierge.proto + grpc/cognizant-technology-agent.proto
operations:
  - ConciergeService_List
  - AgentService_Function
  - AgentService_Connectivity
---

# Discover Neuro SAN agent networks

Neuro SAN is self-hosted. Before anything else, get the base URL from whoever runs the
deployment — Cognizant publishes the software, not a hosted endpoint, and the contract ships
with no `servers[]` block. The provider's own examples use `http://localhost:8080`.

All paths below are under `/api/v1/`.

## Step 1 — list what is available

`ConciergeService_List` — `GET /api/v1/list`

Returns `ConciergeResponse.agents[]`, an array of `AgentInfo` with `agent_name`,
`description` and optional `tags[]`.

Two things to understand about this list:

- **`agent_name` is the key to everything else.** It is the `{agent_name}` path parameter on
  every other operation, and it is also the MCP tool name if you are on that surface instead.
- **The list is not necessarily complete.** Only networks the server marks public appear. The
  provider states plainly that a client is not meant to see every agent registered with a
  service. An absent network is not proof it does not exist.

There is no pagination — no `limit`, `offset`, `cursor` or `page` parameter exists, and
`ConciergeResponse` carries no paging fields. You get the whole array.

## Step 2 — find out what a network needs from you

`AgentService_Function` — `GET /api/v1/{agent_name}/function`

Returns `FunctionResponse.function`, a `Function` with four fields, and the last two are the
ones agents usually miss:

- `description` — outward-facing statement of what the network does.
- `parameters` — what the network needs through the natural-language chat channel. Shaped like
  a pydantic/OpenAI function description.
- `sly_data_schema` — what the network expects through the **private** `sly_data` channel.
- `sly_data_output_schema` — what it will hand back on that channel.

Read `sly_data_schema` before you call. It tells you whether the network needs something —
a credential, a login, a session id — that must *not* go into the chat stream. Sending it as
chat text instead is the most common way to get this wrong.

## Step 3 — inspect the topology, if you need it

`AgentService_Connectivity` — `GET /api/v1/{agent_name}/connectivity`

Returns `ConnectivityResponse.connectivity_info[]`, each a `ConnectivityInfo` with `origin`
(the node being described), `tools[]` (nodes reachable from it) and `display_as`.

Treat a thin result as normal, not as an error. Per the contract's own comment, a server may
withhold connectivity it considers private or an implementation detail — what you get is only
as much as the server chose to reveal.

Entries in `tools[]` may point into **external agent networks on other servers**. Those are
separate deployments; you would have to call their own connectivity operation to learn
anything about them.

## Error handling

Every one of these three operations declares exactly two responses: `200`, and a single
`default` error carrying a `Status` body — `code` (a `google.rpc.Code` integer), `message`
(English, developer-facing) and `details[]`.

This is **not** RFC 9457. Do not look for `type`, `title`, `status`, `detail` or `instance`.
The contract also declares no specific status codes at all — no 400, 401, 403, 404, 429 or
5xx — so read the `code` out of the body rather than branching on the HTTP status.

## What you cannot learn here

- **Auth.** The contract declares no `securitySchemes` and applies no security. That does not
  mean the deployment is open — it means the contract cannot tell you. Ask the operator.
- **Rate limits.** None published, no `X-RateLimit-*` or `Retry-After` headers, no 429. The
  real ceiling is the operator's downstream LLM provider.
