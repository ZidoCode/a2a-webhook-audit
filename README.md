# a2a-webhook-audit

**Audit A2A push-notification webhook URLs for SSRF risk. Offline, deterministic, zero dependencies.**

[![Tests](https://img.shields.io/badge/tests-24%20passed-brightgreen)]() [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)]()

Tenth in the a2a family: [a2a-tax](https://github.com/ZidoCode/a2a-tax), [a2a-stream-tax](https://github.com/ZidoCode/a2a-stream-tax), [a2a-reconnect-tax](https://github.com/ZidoCode/a2a-reconnect-tax), [a2a-artifact-audit](https://github.com/ZidoCode/a2a-artifact-audit), [a2a-task-state-audit](https://github.com/ZidoCode/a2a-task-state-audit), [a2a-context-audit](https://github.com/ZidoCode/a2a-context-audit), [a2a-card-lock](https://github.com/ZidoCode/a2a-card-lock), [mcp-collision](https://github.com/ZidoCode/mcp-collision), [a2a-doctor](https://github.com/ZidoCode/a2a-doctor).

A2A push-notification configs carry a **client-supplied webhook URL that the agent server calls back**. If the server doesn't validate it, an attacker points the webhook at cloud metadata, localhost admin endpoints, or internal databases. This is not theoretical:

- [a2a-go#373](https://github.com/a2aproject/a2a-go/issues/373) — push-notification webhook URLs are **not SSRF-validated**
- [a2a-python#786](https://github.com/a2aproject/a2a-python/issues/786) — SSRF protection is a proposal
- [a2a-python#1023](https://github.com/a2aproject/a2a-python/issues/1023) — URL validation infrastructure still a task

a2a-webhook-audit audits every webhook URL in a recorded stream or config file — **offline, no requests are ever made**.

## Install

```bash
pip install a2a-webhook-audit   # or: copy the .py file
```

## Usage

```bash
$ a2a-webhook-audit report stream.jsonl
a2a-webhook-audit report — 5 URL(s) audited, 8 finding(s)

  [HIGH  ] line 2 (task 't2') 'http://169.254.169.254/latest/meta-data/': webhook targets private/loopback/metadata IP
  [HIGH  ] line 3 (task 't3') 'https://admin:secret@db.internal.corp:6379/x': credentials embedded in URL
  [HIGH  ] line 3 (task 't3') 'https://admin:secret@db.internal.corp:6379/x': webhook targets risky port 6379
  [HIGH  ] line 4 (task 't4') 'http://hooks.example.com/cb': plain http webhook — cleartext
  [INFO  ] line 5 (task 't5') 'http://localhost:8080/dev-hook': acceptable for local dev only

  verdict: 6 HIGH — webhook URLs can be abused for SSRF (a2a-go#373).
$ a2a-webhook-audit audit stream.jsonl && echo safe   # CI gate: exit 2 on HIGH
```

## What it catches

| Finding | Severity | The bug class |
|---|---|---|
| Private/loopback/link-local/metadata IP | **HIGH** | SSRF — cloud metadata (169.254.169.254), internal services |
| Known metadata hostnames | **HIGH** | `metadata.google.internal` & friends |
| Internal hostname suffixes | **HIGH** | `.internal`, `.local`, `.localhost`, `.zone` |
| Credentials in URL | **HIGH** | `user:pass@host` — leaks into logs |
| Plain http (non-localhost) | **HIGH** | tokens in cleartext |
| Risky ports | **HIGH** | 6379 (redis), 3306, 5432, 2375 (docker), 9200, ... |
| Punycode hostname | MEDIUM | homograph risk |
| Plain http to localhost | INFO | acceptable for dev |
| No hostname / unparseable | HIGH | malformed config |

Input: a recorded JSONL stream, a push-notification config JSON, or any JSON doc with webhook URLs — they are found wherever they hide.

## Honest limits

This is an **offline** audit: it catches URL-shape risks with zero network activity and full determinism. DNS rebinding and open redirects are resolved at callback time and need a live probe — pair with a network-level egress policy. MIT license.
