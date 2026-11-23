---
title: "Verification Is Not Authorization"
date: 2026-11-23
description: "Why proving that an operation is correct does not mean it should be allowed to execute."
slug: verification-is-not-authorization
categories:
  - Engineering Notes
  - Software Architecture
tags:
  - verification
  - authorization
  - security
  - AI systems
draft: false
---

# Verification Is Not Authorization

*Why proving that an operation is correct does not mean it should be allowed to execute.*

There is a subtle mistake that can appear in almost any software system, particularly one that relies heavily on automation and AI-assisted execution.

We build a verification mechanism. We validate an operation. The checks pass. The operation is considered safe, and execution proceeds.

It feels reasonable.

But there is a fundamental problem with this reasoning:

**Verification does not grant authority.**

A system can verify that an operation is structurally valid, logically consistent, and compliant with a technical specification without establishing that the operation is authorized to happen.

These are two separate responsibilities. Treating them as one creates a dangerous architectural assumption.

## The difference between correctness and authority

Consider a simple example.

A software component submits a request to modify a system configuration.

The verification mechanism checks:

- Is the request correctly formatted?
- Does the requested operation exist?
- Are the parameters valid?
- Does the operation satisfy the applicable constraints?
- Is the proposed execution plan internally consistent?

Every check passes.

But one question remains:

**Does this component have the authority to request this operation?**

A technically valid request can still come from an untrusted producer. A correctly signed instruction can still be outside the signer's authority. A valid execution plan can still violate the system's authorization policy.

Verification answers whether an operation satisfies specified conditions.

Authorization answers whether an actor is permitted to perform it.

Neither answer implies the other.

## Why this distinction matters more in autonomous systems

In conventional software, developers often establish authority through application permissions, operating-system identities, access-control lists, and service boundaries.

In an autonomous system, the situation becomes more complicated.

A system may contain multiple independently developed capabilities, automated planners, execution engines, external tools, model providers, and persistent state services.

These components may generate instructions, propose actions, exchange data, and initiate workflows.

The system must not assume that an instruction is trustworthy simply because it came from a recognized component or passed a verification check.

An AI model can propose an operation. A planner can produce a valid execution plan. A capability can return a successful result.

None of those events should independently grant permission to act.

The architecture needs an explicit authority boundary.

## A separation of responsibilities

In autonomous and extensible systems, it is useful to distinguish the responsibilities involved in moving from a proposed operation to an executed one.

Verification evaluates whether an operation satisfies its specified conditions. Authorization determines whether the actor and operation are permitted. Execution carries out the operation within those established boundaries.

These responsibilities should not be collapsed into one decision.

A successful verification result is evidence about correctness. It is not a permission grant.

## Registration is not authorization either

The same mistake appears in capability systems.

Suppose a new capability is successfully registered.

What does that actually mean?

It means the system has recorded the component and its descriptive information through its registration process.

It does not automatically mean:

- The capability is authorized to perform every operation it declares.
- The capability is active.
- The capability can execute immediately.
- The capability can communicate directly with other capabilities.
- The capability can modify system state outside its permitted interface.

Registration establishes recognition. Authorization and activation are separate decisions.

The registry's role is to maintain an accurate record of components known to the system. It should not be treated as an automatic grant of operational permission.

## The role of cryptographic evidence

The distinction becomes particularly important when dealing with durable records.

A record may be correctly serialized. Its hash may be valid. Its signature may verify. Its contents may conform to the published schema.

But those properties do not necessarily prove that the producer was authorized to create the record.

Nor do they prove that the signing key was used under the correct authority.

A valid cryptographic signature establishes a relationship between a key and a message. It does not, by itself, establish that the key holder was authorized to perform the action.

That is an authorization question, not a cryptographic shortcut.

## The danger of combining these responsibilities

When verification and authorization are combined, several failure patterns become possible:

1. **Trust by successful validation:** Any structurally valid request is treated as trustworthy.
2. **Caller-controlled identity:** A component supplies its own identity and the system accepts it without independently establishing authority.
3. **Authority through registration:** A registered capability receives more privileges than its admission actually grants.
4. **Execution bypass:** A component reaches an execution interface without the required authorization decision.
5. **Evidence mistaken for permission:** A valid record or signature is treated as proof that the underlying operation was authorized.

These are not merely implementation bugs. They are failures to establish clear architectural boundaries.

## Designing for independent decisions

A robust system should be able to answer these questions separately:

| Question | Responsibility |
|---|---|
| Is the request structurally valid? | Validation |
| Does the operation satisfy its specification? | Verification |
| Who is making the request? | Identity and authentication |
| Is the request permitted? | Authorization |
| Has the component been registered? | Registration |
| Is the component currently enabled? | Activation |
| Can the operation be executed now? | Execution control |
| What happened, and what evidence was retained? | Audit and records |

The exact implementation may vary between systems. The separation of responsibilities should not.

This also improves auditability. When an operation fails, the system can identify whether the failure was caused by invalid input, failed verification, missing authority, rejected authorization, an execution constraint, or a runtime failure.

Without this separation, all these conditions tend to collapse into a vague notion of a failed operation.

## A principle worth preserving

As autonomous systems become more capable, one principle deserves to remain explicit:

**Correctness is not permission.**

Verification must not become authorization. Registration must not become activation. Identity must not automatically become trust. A successful execution must not become proof of legitimacy.

Each decision needs a defined responsibility and an independently enforceable boundary.

The more capable a system becomes, the more important it is to distinguish what it *can* do from what it is *allowed* to do.

A system that can verify its actions is useful.

A system that also understands where its authority comes from is safer.

And a system that keeps those responsibilities separate is easier to reason about, audit, and evolve.

---

*Engineering Notes — Software architecture, verification, authority, and autonomous execution.*
