<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. -->
<!-- SPDX-License-Identifier: MIT-0 -->
# Tenant memory isolation through Gateway

A draft solution code sample for multiple agents sharing a tenant's long-term Memory
namespace through the native `agentcore-memory` Gateway connector. It includes
real Cedar evaluation tests, a native HTTP client, generated configuration, and a
three-identity cloud test. Live deployment in `ap-southeast-2` reached READY
resources and ACTIVE policies on 22 October 2026. One authorized `CreateEvent`
succeeded, but its subsequent `GetEvent` was denied; a repeat deployment still
denied the write after bounded readiness retries. An Oregon comparison also
denied the write; separate direct IAM and role-assumption probes were denied
as expected. The full live matrix has
**not passed**.
See [validation evidence](VALIDATION.md). This is not a complete CDK application
or a merge-ready sample.

## Architecture

![JWT-scoped tenant memory through Gateway](diagrams/agentcore-memory-tenant-isolation.png)

[Editable AWS architecture diagram](diagrams/agentcore-memory-tenant-isolation.drawio).

Tenant agents forward issuer-signed tokens to Gateway; enforced Cedar policies
check actor, session, and namespace before its execution role calls Memory.

- The vendor's OIDC issuer assigns tenant membership and allowed memory scopes.
- Gateway uses `CUSTOM_JWT`, a Policy engine in `ENFORCE`, and its execution role.
- Gateway creation omits `protocolType`, following the native Memory setup.
  An explicitly MCP-only Gateway rejects the Memory HTTP target.
- Memory stores separate short-term sessions and shared tenant facts.
- Memory's resource policy explicitly denies data-plane calls outside Gateway.

## How it works

1. The vendor issues three distinct tokens: Tenant A shopping, Tenant A support,
   and Tenant B shopping. Claims must be issuer controlled:

   | Identity | memory_actor | memory_session | memory_namespace |
   | --- | --- | --- | --- |
   | A shopping | tenant-a | shopping-001 | /tenants/tenant-a/shared/ |
   | A support | tenant-a | support-001 | /tenants/tenant-a/shared/ |
   | B shopping | tenant-b | shopping-001 | /tenants/tenant-b/shared/ |

2. The native HTTP connector routes `/{targetName}/{Memory request path}`.
   [gateway_memory.py](src/gateway_memory.py) uses botocore's Memory model for
   path escaping, parameter validation, and timestamp serialization.
3. The event policies require the exact actor **and** session in the token.
   The recall policy requires the exact namespace path. Unknown operations and
   absent claims have no permit.
4. A semantic extraction strategy uses `/tenants/{actorId}/shared/`.
   Both A agents can retrieve A's shared facts after asynchronous extraction.
   B uses a separate path. All descendants of a shared path must belong to
   that same sharing domain.
5. The downstream execution role grants six Memory data operations. Its trust
   permits only the AgentCore service with the exact account and Gateway/Policy
   engine source ARNs. Memory explicitly denies data calls whose
   `aws:PrincipalArn` differs from this role. Caller IAM session tags do not
   supply the tenant scope; the signed JWT and Cedar do.

The execution role separately grants `GetMemory` on this resource for connector
metadata validation and the three Policy evaluation permissions on its engine
and Gateway. These grants do not add tenant-facing Cedar permits.

Tenant-wide organisational facts are deliberately shared here. Do not put private
tenant-user preferences into this actor. Give private data a distinct actor/path
and issue an appropriately scoped token. See [architecture.md](architecture.md).

## Deploy

Use Python 3.12+, Node.js 20+, and AWS credentials for a dedicated test account.
Choose a Region supporting Memory, Gateway's native Memory connector, and Policy.
Install the dependencies and run the local checks:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
npm ci
npm test
python3 -m unittest discover -s test -p 'test_*.py'
```

For an isolated live test, the SDK driver creates a temporary Cognito user pool,
a token customization Lambda, a scoped Gateway role, and the Memory/Gateway/Policy
resources in one Region:

```bash
python3 -m src.live_environment --region "$AWS_REGION"
python3 -m src.live_environment --region "$AWS_REGION" --execute
```

The first command prints the plan without AWS calls. The second requires
deployment permissions for IAM, Cognito, Lambda, CloudWatch Logs, and AgentCore.
It runs the authorization matrix and removes its test resources in a `finally`
block. Passwords and signed tokens stay in process memory. Inspect ignored
`live-report.json` for the matrix and `live-state.json` for cleanup status.
The driver issues tokens after provisioning and checks their fixed fixture scopes.
This readiness check does not verify signatures; Gateway performs JWT validation.
On failure, ignored `live-diagnostics.json` contains bounded service diagnostics
with bearer tokens redacted.
If execution is interrupted, resume cleanup with:

```bash
python3 -m src.live_environment --region "$AWS_REGION" --execute --cleanup
```

The test issuer assigns fixed scopes to three administrator-created synthetic
identities; callers cannot choose claims through attributes or client metadata.
This fixture is for validation. A production issuer must derive entitlements from
the vendor's verified tenant membership and session assignment (ADR-0003).
The SDK driver does not satisfy the repository's CDK/integ-runner merge gates.

For manual deployment, follow the sequence below.

Supply an existing OIDC issuer and a Gateway execution role trusted by
`bedrock-agentcore.amazonaws.com`. During creation, scope its trust to the exact
`aws:SourceAccount` and `arn:aws:bedrock-agentcore:REGION:ACCOUNT:*` source ARN,
as documented for Policy evaluation. After Gateway is created, narrow the allowed
source ARNs to its Gateway ARN and the Policy engine ARN.
The issuer must validate membership and agent/session entitlement; a tenant user
must not be able to edit custom claims. Supply supported token client/audience
restrictions and token lifetimes.

The renderer generates reviewable configuration. Provision in this
order with the AWS CLI or equivalent SDK:

1. Create a Policy engine (`create-policy-engine --name tenant_memory_validation`).
2. Create a dedicated Memory with the `TenantFacts` semantic strategy and seven-day
   event expiry shown in `src/render_config.py`. Wait until Memory is ACTIVE.
3. Render configuration with the actual identifiers. Use `pending` for Gateway ID
   on this first pass because Gateway has not been created:

   ```bash
   python3 -m src.render_config \
     --region "$AWS_REGION" --account "$TEST_ACCOUNT_ID" \
     --gateway-id pending --memory-id "$MEMORY_ID" \
     --role-arn "$GATEWAY_ROLE_ARN" --discovery-url "$OIDC_DISCOVERY_URL" \
     --client-id "$OIDC_CLIENT_ID" --policy-engine-arn "$POLICY_ENGINE_ARN"
   ```

4. Apply `gateway-role-policy.json` as the role's Memory and Policy permissions, then create
   Gateway with `create-gateway --cli-input-json file://generated/create-gateway.json`.
   Policy evaluation temporarily uses `gateway/*` in this account and Region,
   because creation validates it before returning a Gateway ID. Memory permissions
   remain pinned to the dedicated Memory. ENFORCE is attached immediately; an
   empty engine denies traffic.
5. Wait for Gateway READY. Re-render using its actual Gateway ID and reapply
   `gateway-role-policy.json` to pin Policy evaluation to that Gateway. Create the
   native target with `create-gateway-target --cli-input-json file://generated/create-target.json`.
   Wait until the target is READY.
6. Apply each `*-policy.json` using `create-policy`; require ACTIVE status and
   `FAIL_ON_ANY_FINDINGS`. A rejected schema is a failed deployment: do not
   bypass validation or replace the policy with a broad permit.
7. Install `memory-resource-policy.json` using `put-resource-policy` on the exact
   Memory ARN. Pin the role's trust to the actual Gateway and Policy engine ARNs.
8. Fill ignored `live-config.json` from `config/live-config.example.json` using
   GetGateway, GetGatewayTarget, and GetMemory output. Set `targetId` to the
   actual target identifier. Load the three signed JWTs into the named environment
   variables without printing them or storing them in source.

Use the CLI's `--region "$AWS_REGION"` on every regional command.
Configuration for GovCloud/China partitions would need ARN/domain changes.

## Deployment integration test

The SDK driver provisions a throwaway identity provider and drives the dedicated
environment's lifecycle (ADR-0003). The standalone harness below can also use an
existing configured test deployment. This draft does not run integ-runner or
satisfy the repository's full CDK contribution standard yet.

```bash
python3 -m src.validate_live --config config/live-config.example.json
python3 -m src.validate_live --config live-config.json --execute
```

The first command makes no network calls and reads no tokens. The second writes
three synthetic events, allows up to ten minutes for extraction, and runs the
allow/deny matrix. It checks the actual Gateway configuration and resource
fence before testing data. An empty retrieval is **not** evidence of isolation;
shared recall must contain the synthetic canaries.

Expected positive event writes and reads allow up to five minutes for readiness.
Write retries reuse one `clientToken` to avoid duplicate events. A persistent
403 fails the run. Negative probes make one request and require the actual denial.

Negative cases require an observed 403; malformed input, 404, throttling, or 5xx
fail the test. An optional already-expired `EXPIRED_TEST_TOKEN` checks token
expiry; the report records NOT_RUN if absent. Do not call those unrun cases passed.

Cleanup failures also fail the driver. If the deployment session cannot delete
a resource, the journal records it for a later `--cleanup` attempt. Confirm
`cleanedUp: true`; a completed authorization matrix alone does not prove teardown.

Allow roughly 15–20 minutes, depending on extraction and setup. No cost figure
has been measured. Charges include Memory events, extraction/retrieval, Gateway,
Policy, and any chosen identity provider. Use current regional pricing before
running. Test calls stay in the single Region in `live-config.json`.

## End-to-end demo

1. Run local tests and inspect the supplied three Cedar policies.
2. Provision the dedicated test environment and confirm all policies are ACTIVE.
3. Run the live harness. It prints case names and status without token or
   memory contents. Inspect `live-report.json`, including NOT_RUN cases.
4. Compare A shopping/A support recall with the denied A→B path change.
5. Tear down the dedicated test Memory, Gateway target, Gateway, and Policy
   engine/policies. Remove the role policy/role only if created for this test.
   The harness deletes its event IDs, but extracted long-term records persist
   until the dedicated Memory is deleted. Retain the issuer only if shared.

## Security notes

See [threat-model.md](threat-model.md). Local tests prove Cedar decisions against
synthetic entities, not the deployed Gateway's context mapping or JWT handling.
Record-ID operations, tenant enumeration, extraction administration, and batch
writes are omitted from tenant permits. A future permit can widen this boundary;
review the whole policy set after changes.

## Production hardening checklist

- Implement and test membership-derived claims and revocation (T-1).
- Enforce private-user sharing boundaries and tenant-aware caching (T-2, T-8).
- Validate connector action schemas and all bypass routes live (T-3, T-4).
- Add least-privilege deployment roles, policy drift monitoring, and log retention (T-5).
- Monitor extraction delays, expiry, cleanup, and cost (T-6).
- Add input provenance and protections against poisoning shared facts (T-7).
- Complete CDK, diagram, scanner, and lifecycle gates before proposing a PR (ADR-0002).

## Design trail

- [Specification](design/tenant-memory-isolation.spec.md)
- [Architecture and validation status](architecture.md)
- [Architecture decisions](adr/README.md)

## References

- [Native Memory connector](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-gateway-connector.html)
- [Memory action/context reference](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-gateway-fgac.html)
- [Policy examples](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-gateway-fgac-policy-examples.html)
- [HTTP Gateway target routing](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-http-passthrough.html)
- [Restrict direct Memory access](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-gateway-restrict-access.html)
- [Policy evaluation IAM permissions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-permissions.html)
- [Cognito pre-token customization](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html)
