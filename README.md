---
title: "L4S User-Space Congestion Control Demo"
subtitle: "Engineering Reference & Build Guide — Project Initialization, Maven Reactor, Quality Gates, and CI/CD Pipelines"
date: "July 2026"
---

# Preface — About This Document

This document is an educational engineering reference for **`l4s-user-space-demo`**, a
portfolio-grade project that implements a user-space **L4S (Low Latency, Low Loss, Scalable
Throughput)** congestion controller running over **QUIC/UDP** in Java, exposes its telemetry
through **Redis Pub/Sub**, and surfaces that telemetry on a **real-time web dashboard**. The
document is written the way a Principal Software Architect would write an internal engineering
handbook before a team starts coding: it explains *why* the system is shaped the way it is before
it shows *how* to build it, and it hands over the exact configuration artifacts — Maven POMs,
static-analysis rulesets, and CI/CD pipelines — needed to start an incremental, production-quality
implementation.

**Scope of this revision.** This revision covers the *initialization phase* of the project:

- The full system architecture and the reasoning behind each technology choice.
- The repository layout and the dependency rules between modules.
- A phased, incremental roadmap from an empty Git repository to a deployable system.
- The complete root (parent) Maven POM, with dependency management, plugin management, and
  quality-gate profiles.
- The complete module POMs for `engine`, `web-dashboard`, `benchmarks`, and `tests-integration`.
- Static analysis configuration: Checkstyle, PMD, SpotBugs, JaCoCo, Maven Enforcer, and OWASP
  Dependency-Check.
- The complete GitHub Actions CI/CD pipelines: build, unit tests, integration tests, static
  analysis, coverage, Docker build, and release.
- The initial engine telemetry domain model — the first vertical slice of production code,
  written to Java 21, SOLID, and Javadoc standards, with accompanying unit tests.

**Explicitly out of scope for this revision.** In line with the project's own engineering rule of
building "one subsystem at a time," this revision does **not** contain the React/TypeScript
dashboard implementation, the Netty QUIC pipeline internals, the Prague congestion-control
algorithm implementation, or the Spring Boot controllers. Those are the subjects of subsequent
revisions once the scaffolding in this document is in place. Frontend and UI code are intentionally
excluded from this document by request; the frontend's *responsibilities* are described in the
architecture chapter so that backend and engine contracts can be designed to satisfy them.

**How to use this document.** Read Chapters 1–4 before writing any code — they are the
architectural contract that every later decision must remain consistent with. Chapters 5–9 are
copy-ready configuration and source artifacts. Chapters 10–12 describe the engineering practices
(testing, containerization, observability) that the roadmap assumes. Chapter 13 is a glossary and
reference appendix for readers who are new to L4S, QUIC, or Prague congestion control.

\newpage

# Chapter 1 — Introduction & Educational Objectives

## 1.1 What Problem Does This Project Solve?

Traditional loss-based congestion control (Reno, CUBIC) trades latency for throughput: routers
build deep queues to avoid dropping packets, and applications infer congestion only after a queue
overflows and a packet is lost. This produces **bufferbloat** — multi-hundred-millisecond queueing
delay under load — which is fatal to latency-sensitive traffic such as video conferencing, cloud
gaming, and remote-control systems.

**L4S** (documented in **RFC 9330**, the L4S architecture, alongside **RFC 9331**, the "Prague"
requirements for L4S congestion controllers, and **RFC 9332**, the Dual-Queue Coupled AQM) is an
IETF architecture that resolves this trade-off by combining three ingredients:

1. **A new Explicit Congestion Notification (ECN) codepoint**, ECT(1), which routers use to mark
   packets as belonging to a scalable, low-latency traffic class instead of the classic ECT(0)/CE
   marking used by RFC 3168 ECN.
2. **A Dual-Queue Coupled Active Queue Management (AQM)** at the bottleneck, which keeps a very
   shallow marking threshold (typically ~1 ms) for the L4S queue while remaining fair to classic
   (non-L4S) traffic sharing the same link.
3. **A "Prague" congestion controller** at the sender, which reacts to *the proportion of
   ECN-marked packets* rather than to packet loss, and does so in a way whose steady-state window
   is **independent of RTT** — a crucial scalability property that classic Reno-style AIMD lacks.

This project reimplements the sender-side half of that architecture — the Prague congestion
controller, RTT estimation, ECN/ACK processing, and CWND management — **in user space**, riding on
top of Netty's QUIC implementation rather than requiring kernel patches (which is how most
reference L4S implementations, e.g. Linux `tcp_prague`, are built). This lets the whole system run
identically on a developer's laptop, in CI, and on a plain Ubuntu VPS with no special kernel
module, which is precisely what makes it viable as a **portfolio / learning project** rather than
a kernel-networking research artifact.

## 1.2 Engineering Rules Governing This Project

The specification this document is built from encodes a set of non-negotiable engineering rules.
They are restated here because every chapter that follows is a direct consequence of them:

1. Build incrementally, one subsystem at a time — never "big bang" the whole system.
2. Explain the architecture before writing implementation code.
3. Write production-quality code that follows SOLID principles.
4. Favor interfaces over concrete implementations at every module boundary.
5. Use dependency injection wherever object graphs are assembled.
6. Every class and public method carries Javadoc.
7. Every component ships with unit tests, and cross-component behavior ships with integration
   tests.
8. Maven configuration is provided in full whenever dependencies change — never "add this
   dependency" without showing the resulting POM.
9. The `engine` module must never depend on the `web-dashboard` module (or any Spring Framework
   type). Telemetry flows from `engine` to `web-dashboard` only through Redis, as data — never
   through a shared Java type from a web framework.
10. Architectural quality is never sacrificed for shorter code.

\newpage