<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. -->
<!-- SPDX-License-Identifier: MIT-0 -->
# Implemented architecture

**Status:** Draft solution sample; local tests pass on 2026-10-22.
**Last cloud integration:** Attempted in `ap-southeast-2` on 2026-10-22.
Provisioning reached READY resources and ACTIVE policies. One authorized write
succeeded, but its subsequent read was denied; a repeat deployment still denied
the write after bounded retries. Separate direct Memory access and execution-role
assumption probes were denied in `us-west-2`. Shared recall and cross-tenant
runtime rejection remain unverified. The independent teardown audit found all
recorded service resources absent; only two permissionless IAM roles remain
because the deployment session denies their deletion. See `VALIDATION.md`.

## Purpose and identity

The vendor pools one Memory resource for synthetic tenant organisations.
The issuer supplies full memory scopes; Gateway validates signed JWTs and
enforces Cedar. Both A agents have the same actor/shared path and distinct
session claims (ADR-0001). Clients cannot construct a broader permit.

## Components

`src/gateway_memory.py` routes SDK-serialized native HTTP requests through the
named connector. `src/render_config.py` emits resource, target, role, and policy
configuration without submitting it. `src/validate_live.py` explicitly opts
into real calls and reports observed status. `src/live_environment.py` creates
the dedicated environment and tears it down in a `finally` block.
`src/test_identity.py` supplies Cognito access-token scopes for three fixed
synthetic identities (ADR-0003); production tenant membership is an external
responsibility.

## Data model and workflows

Events belong to an actor and session. The strategy extracts tenant-wide facts
to `/tenants/{actorId}/shared/`. Retrieval uses only the exact issuer-provided
namespacePath; descendants must have the same sharing entitlement.
CreateEvent cannot supply extractionConfig in this kit.

Event Read/Delete and Create have separate permits because their request
schemas differ. Retrieve/List records share a namespace permit. Record-ID
access is excluded because the ID alone does not prove namespace ownership.
Resource-based IAM denies the listed Memory data APIs outside the exact
Gateway execution role, using `aws:PrincipalArn`. Its trust allows only the
AgentCore service with the exact source account and Gateway/Policy engine
ARNs. Gateway's role grants only the demonstrated Memory operations.

## Deployment details

The native Memory target uses the HTTP connector. Gateway creation omits
`protocolType`; an explicitly MCP-only Gateway rejects that target.
Policy engine and policy names use letters, digits, and underscores.

The Gateway role also needs `AuthorizeAction`, `PartiallyAuthorizeActions`, and
`GetPolicyEngine` for its Policy engine. During creation, Policy evaluation
temporarily permits `gateway/*` in the selected account and Region, while Memory
permissions stay pinned to the dedicated resource. Before adding a target, the
driver replaces the wildcard with the actual Gateway ARN and narrows role trust
to the Gateway and engine ARNs. JWT authentication and ENFORCE remain enabled
throughout. The role separately allows `GetMemory` on the dedicated resource
for target metadata validation; the six tenant data actions stay unchanged.

## Validation

37 policy cases run in Cedar WASM 4.14.0. Thirty-one Python tests cover
SDK wire serialization, HTTP error classification, redirects, endpoint/target
validation, generated IAM/connector configuration, live harness inputs,
issuer-controlled scopes, idempotent readiness retries, diagnostic redaction,
failure cleanup, and the combined resource-policy/role-trust boundary.
The results are recorded in `VALIDATION.md`.

The cloud harness requires the real Gateway URL and connector configuration to
match its inputs, three separately scoped signed JWTs, and AWS read permissions
for preflight checks. It submits synthetic facts, waits for both tenants'
canaries, probes cross-tenant and cross-session access, and tests a direct IAM
request outside Gateway and an attempted direct role assumption. Failed
extraction is a failed/incomplete test, not a
successful isolation outcome.

This draft intentionally stops short of the full deployable sample standard
(ADR-0002); the README lists remaining merge gates.
