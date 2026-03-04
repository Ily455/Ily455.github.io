---
title: "Full SIEM Deployment"
date: 2023-06-15
draft: false
tags: ["siem", "elastic", "wazuh", "misp", "thehive", "suricata"]
description: "Deployment of a complete SOC-oriented monitoring and incident analysis stack."
---

A full-stack SIEM/SOC lab deployment to practice detection engineering, event correlation, and incident workflow.

## Stack

- Elastic Stack for ingestion, indexing, and dashboards
- Wazuh for host-based monitoring and security telemetry
- Suricata for network detection
- MISP for IoC management and enrichment
- TheHive for investigation and case handling

## What I worked on

- Building and integrating the full pipeline from telemetry to investigation
- Writing and tuning detection rules
- Correlating events across host and network data
- Running IoC-driven analysis workflows

## What I learned

- SIEM value depends heavily on data quality and correlation logic
- Detection tuning is iterative and context-dependent
- Incident tooling is only useful when workflows are documented and repeatable
