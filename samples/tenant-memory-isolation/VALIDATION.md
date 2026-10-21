<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. -->
<!-- SPDX-License-Identifier: MIT-0 -->
# Validation evidence — 22 October 2026

| Check | Observed outcome |
| --- | --- |
| `npm ci` / pinned Cedar WASM 4.14.0 | Installed successfully |
| `npm test` | 37 real Cedar evaluation cases passed |
| `python -m unittest discover -s test -p 'test_*.py'` | 31 tests passed |
| `python -m compileall -q src test` | Passed |
| `python -m src.validate_live --config config/live-config.example.json` | Dry run passed; no network calls or tokens read |
| SDK configuration shapes | CreateMemory, CreateGateway, CreateGatewayTarget, and CreatePolicy validated locally and submitted successfully with boto3 1.43.116 / botocore 1.43.117 |
| `npm audit --omit=dev --audit-level=high` | No reported vulnerabilities |
| `bandit 1.8.6 -r src -q` | No findings |
| Disposable Cognito issuer | Three signed access tokens issued with the expected actor, session, namespace, and client claims |
| Live provisioning in `ap-southeast-2` and `us-west-2` | Memory, native HTTP target, and Gateway READY; Policy engine and three policies ACTIVE with strict validation |
| Gateway and Memory fence preflight, initial design | PASS: `CUSTOM_JWT`, `ENFORCE`, native connector, and expected resource-policy document |
| Permitted `CreateEvent` | One live write succeeded under the execution-role fence; a repeat deployment still returned 403 after bounded retries |
| Permitted `GetEvent` | FAILED after the successful write: Policy returned 403, “No policy applies to the request (denied by default)” |
| Oregon comparison | Permitted `CreateEvent` still denied after five minutes of readiness retries |
| Narrow `GetEvent` forbid diagnostics in Oregon | Caller/tenant-claim/action rule matched. Separate expected Memory ID, actor ID, and session ID equality rules did not match |
| Shared recall and tenant/session rejection | NOT RUN: the complete matrix has not progressed past the positive event setup checks |
| Separate direct IAM Memory probe in `us-west-2` | PASS: `RetrieveMemoryRecords` denied with `AccessDeniedException`; service diagnostic identified an explicit resource-policy deny |
| Separate direct execution-role assumption in `us-west-2` | PASS: deployment caller's `AssumeRole` attempt denied with `AccessDenied` |
| Independent teardown audit | 152 recorded-resource checks: 150 absent; only two IAM roles remain, with no inline or attached permissions. Their deletion is denied by the deployment session policy |
| Cost | Not measured |

The 37 Cedar cases cover positive session access, intra-tenant shared recall,
changed actors/sessions/namespaces, missing claims, excluded administrative and
record-ID operations, alternate resources, IAM principal mismatch, and namespace
variable override rejection. They use Cedar's evaluator, not a Python
reimplementation of its decision logic.

Python tests cover SDK wire serialization, error classification, redirect
handling, endpoint/target validation, generated ENFORCE configuration, the IAM
resource fence, malformed live identity configurations, fixed issuer scopes,
idempotent write retries, bounded positive-read retries, diagnostic redaction,
and failure cleanup.
The driver now checks every issued fixture token's actor, session, namespace,
client, and token type after provisioning. This decodes claims for fixture
readiness; it does not replace Gateway's signature verification. A support event
seeds the separate support session before its positive `ListEvents` probe.

Local tests do not validate the service's action schema, JWT claim-to-tag mapping,
HTTP routing, resource-policy behavior, semantic extraction, or direct bypass.
The attempted live run does not establish the runtime sharing or isolation
claims. The separate direct-call probes establish those individual negative
outcomes, not intra-tenant sharing or cross-tenant enforcement. An ACTIVE policy
alone is not runtime evidence. Resolve the persistent
positive-request denials, then rerun the complete matrix before treating this
design as cloud verified.

Live setup exposed and corrected three deployment requirements: Policy
evaluation permissions on the Gateway role, omission of the MCP-only protocol
setting for native HTTP targets, and a separate resource-scoped `GetMemory`
metadata grant. The driver pins bootstrap permissions to the actual Gateway ARN
before adding the Memory target. These fixes have regression tests.

## Unresolved runtime discrepancy

The [documented request context](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-gateway-fgac.html)
combines URI parameters and body fields under `context.input`. The SDK client
sends the documented method and URI template. Deployed policy statements matched
the rendered source, and the issued fixture claims matched their expected scopes.
Nevertheless, permitted requests were denied in both test Regions.

The diagnostics added only narrow temporary `forbid` rules, without widening
permits or disabling enforcement. On `GetEvent`, the OAuth principal, tenant
claim, action, and Gateway matched one diagnostic rule; the expected path-field
equality rules did not. A separate `CreateEvent` rule comparing `memoryId` to the
Memory ARN also did not match. These observations do not prove that fields are
absent; their representation, population, and policy propagation need inspection.

Next, inspect the connector's actual authorization context and path-parameter
mapping with the AgentCore service team, then rerun all positive and negative
cases. Do not remove actor/session checks to force a successful request.

Later comparisons reused two empty, test-owned IAM roles because the deployment
session denied their deletion. Their trust and inline permissions were refreshed
and checked by the harness preflight. The diagnostic rules were removed during
teardown; retained roles have no inline or attached permissions.

Signed tokens, account/resource identifiers, diagnostics, and raw cloud reports
stay in ignored local files. This document records only sanitized results.
