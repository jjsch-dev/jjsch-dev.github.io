---
layout: default
title: "Firmware + AI"
---

# Firmware + AI

A case study series documenting the ES242F Smart Lock project — a firmware retrofit built with dual LLM agents, oscilloscope validation, and a method that emerged from failure, not theory.

## About This Series

This is not a tutorial on "how to use AI for embedded development." It is not a benchmark comparing LLMs. It is a single project, documented with evidence: scope traces, bus logs, runbook results, CI screenshots, and typed decision records.

Every claim is backed by something reproducible on a bench. There are no "it probably works" or "it should work" statements. There is only "the scope said" or "the test refuted it."

## Articles

1. **[Part 0: The Old Workshop and the New Hammer](/articles/arco-0-old-workshop)** — Why one engineer didn't delete 3,000 lines of AI-generated code, and what happened when the AI wrote faster than understanding could follow.

## The Project

- **Hardware:** ESP32-S3 retrofit of a Tuya smart lock
- **Firmware:** 49K+ lines of C, bare-metal with ESP-IDF
- **Backend:** TypeScript (tRPC / Drizzle / Hono), 293 tests
- **Mobile:** Flutter app for virtual credentials (NFC-HCE + BLE beacon)
- **Method:** Dual-agent architecture with Phase 0 gates, containment audits, and physical bench validation

## Repositories

- [lockdrv_test](https://github.com/jjsch-dev/lockdrv_test) — Hardware test bench for the ES242F
- [uart_sniffer_pio](https://github.com/jjsch-dev/uart_sniffer_pio) — Logic analyzer firmware for RP2040

## Contact

Feedback, corrections, and technical discussion are welcome via GitHub issues.
