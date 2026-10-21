<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. -->
<!-- SPDX-License-Identifier: MIT-0 -->
# ADR-0001: Use issuer-scoped claims with the native Memory connector

- Status: Proposed
- Date: 2026-10-21
- Deciders: Pending technical review

## Context

Several agents in one tenant need shared facts while sessions remain private.
The application authenticates callers with OAuth tokens.

## Options considered

### Option 1: IAM session tags and a token vending service

Fits direct IAM callers but requires credential issuance and renewal. Caller
session tags do not become the outbound execution role's tags automatically.

### Option 2: Native Memory connector and JWT-aware Cedar

Allows actor/session/namespace checks against the authenticated token before
Gateway assumes downstream responsibility. Requires trustworthy full-path
claims, connector schema validation, and a resource-policy fence.

## Decision

Implement option 2 for this draft. Use a tenant actor for deliberately shared
organisational facts, one issuer-scoped session per agent, exact namespace
claims, default deny, and ENFORCE. Keep direct IAM/ABAC as a separately
documented choice, not a hidden dependency.

## Consequences

Agents share facts without sharing session history or IAM credentials.
Private user facts require separate actors/paths. Cedar validation and real
Gateway tests remain necessary; offline evaluation is insufficient.
