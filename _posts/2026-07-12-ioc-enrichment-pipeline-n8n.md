---
title: "Building an IOC Enrichment Pipeline with n8n and a Local LLM"
date: 2026-07-12
categories:
  - homelab
  - dfir
permalink: /homelab/ioc-enrichment-pipeline/
tags:
  - n8n
  - dfir
  - threat-intel
  - virustotal
  - abuseipdb
  - shodan
  - ollama
  - automation
  - soar
layout: single
author_profile: true
toc: true
---

IOC enrichment is one of those tasks that eats an analyst's day. You get an IP address or a file hash, and then you open VirusTotal in one tab, AbuseIPDB in another, Shodan in a third, copy and paste between them, and try to form a judgment. Repeat fifty times during an investigation and it stops being analysis and starts being data entry.

This post covers a pipeline I built in n8n that automates the whole loop: paste an indicator, and it queries three threat intelligence sources, correlates the results, and has a local LLM write a plain-language analyst verdict. Everything runs on my own hardware except the three reputation lookups.

This runs on the Fedora AI workstation I set up in my previous posts: [Part 1](/home-lab-blog/homelab/fedora-ai-setup-part1/) covers the OS and ROCm, [Part 2](/home-lab-blog/homelab/fedora-ai-setup-part2/) covers Ollama, Open WebUI, and n8n.

---

## What it does

```
Form (paste an IP or file hash)
        |
     Switch (regex routing)
        |-- ip   --> AbuseIPDB --> VirusTotal --> Shodan --> IP merge ---+
        |-- hash --> VirusTotal (file) --> Hash Summary ----------------+
                                                                         |
                                                        Basic LLM Chain (Ollama)
                                                                         |
                                                          Verdict shown on the form
```

Three sources, each answering a different question:

| Source | Question it answers |
|---|---|
| AbuseIPDB | Has this IP been reported for abuse? (reputation) |
| VirusTotal | Do security vendors flag it? (detections) |
| Shodan | What is this host actually running? (exposure) |

The local model then reads all three and writes the verdict. That last step is what turns raw data into something an analyst can act on.

---

## Prerequisites

Three free API keys:

- **VirusTotal** — [virustotal.com](https://www.virustotal.com), profile icon then API Key. Free tier: 4 requests/minute, 500/day
- **AbuseIPDB** — [abuseipdb.com](https://www.abuseipdb.com), Account then API tab. Free tier: 1,000 checks/day
- **Shodan** — [shodan.io](https://www.shodan.io), key is on your account page

Plus a self-hosted n8n instance and Ollama with a capable model. I use `qwen3.6:35b`.

> **Store these as n8n credentials, not as plain text in node parameters.** I learned this the hard way. More on that at the end.

---

## Step 1 — The form trigger

Create a new workflow and add an **On form submission** node.

- Form Title: `IOC Enrichment`
- Form Description: `Paste an IP address or file hash`
- Add one Form Element: Text Input, labeled `ioc`
- Respond When: **Workflow Finishes** (this matters later, so the verdict can be shown on the form)

Execute the step, open the Test URL, and submit `8.8.8.8`. The output should show your input:

```
ioc: 8.8.8.8
submittedAt: 2026-07-12T...
formMode: test
```

---

## Step 2 — Routing with regex

The pipeline needs to know whether the input is an IP or a hash, because they hit different endpoints. My first attempt used a Code node to detect the type, then an IF node to route. It worked, but a **Switch node with regex matching** does both jobs in one node and makes the canvas self-documenting.

Add a **Switch** node, Mode: Rules.

**Routing Rule 1** (rename output to `ip`):
- Value: `{{ $json.ioc }}` (expression)
- Condition: String, **matches regex**
- Pattern: `^(\d{1,3}\.){3}\d{1,3}$`

**Routing Rule 2** (rename output to `hash`):
- Value: `{{ $json.ioc }}` (expression)
- Condition: String, **matches regex**
- Pattern: `^[a-fA-F0-9]{32}$|^[a-fA-F0-9]{40}$|^[a-fA-F0-9]{64}$`

The three alternations in the hash pattern cover MD5 (32 chars), SHA1 (40), and SHA256 (64).

Renaming the outputs means the canvas shows `ip` and `hash` instead of `Output 0` and `Output 1`. Small thing, big difference when you come back to a workflow weeks later.

> **A note on the "is equal to" operator.** I originally tried routing on a `type` field with the String "is equal to" condition. The input was correct, the expression preview showed the right value, and the output was still empty. I never resolved it. Regex matching worked immediately and is a better fit anyway.

---

## Step 3 — The IP path

Three HTTP Request nodes chained off the Switch's `ip` output.

### AbuseIPDB

- Method: GET
- URL: `https://api.abuseipdb.com/api/v2/check`
- Authentication: Generic Credential Type, **Header Auth** (credential Name: `Key`, Value: your API key)
- Send Query Parameters: ON
  - `ipAddress` = `{{ $json.ioc }}` (expression)
  - `maxAgeInDays` = `90`
- Send Headers: ON
  - `Accept` = `application/json`

For 8.8.8.8 this returns an abuse confidence score of 0, whitelisted status, ISP, country, and report counts.

### VirusTotal (IP)

- Method: GET
- URL: `https://www.virustotal.com/api/v3/ip_addresses/{{ $('Switch').item.json.ioc }}`
- Credential: VirusTotal (n8n has a dedicated credential type for this)

Note the `$('Switch')` reference. AbuseIPDB's response replaced the item data, so the IP has to be pulled back from the Switch node rather than the previous node's output.

The response is large. The field that matters is `last_analysis_stats`, which gives malicious, suspicious, harmless, and undetected counts across roughly 90 vendors.

### Shodan

- Method: GET
- URL: `https://api.shodan.io/shodan/host/{{ $('Switch').item.json.ioc }}`
- Authentication: Generic Credential Type, **Query Auth** (credential Name: `key`, lowercase, Value: your API key)

> Shodan's parameter name is case-sensitive. Using `Key` instead of `key` returns a 401 Unauthorized with a confusing "check your credentials" message even though the key itself is fine.

---

## Step 4 — Correlating the results

Add a **Code** node named `IP merge` after Shodan. It pulls the useful fields out of all three responses and builds a single clean object, plus the prompt that will be sent to the model.

```javascript
const abuse = $('AbuseIPDB').item.json.data;
const vt = $('VT-IP').item.json.data.attributes;
const shodan = $input.first().json;

const summary = {
  ioc: $('Switch').item.json.ioc,
  abuseConfidenceScore: abuse.abuseConfidenceScore,
  totalReports: abuse.totalReports,
  countryCode: abuse.countryCode,
  isp: abuse.isp,
  vtMalicious: vt.last_analysis_stats.malicious,
  vtSuspicious: vt.last_analysis_stats.suspicious,
  vtHarmless: vt.last_analysis_stats.harmless,
  vtReputation: vt.reputation,
  shodanPorts: shodan.ports || [],
  shodanOrg: shodan.org || 'unknown'
};

summary.prompt = `You are a SOC analyst. Based on the following threat intelligence data, write a concise verdict (2-3 sentences) on whether this IP is safe, suspicious, or malicious. Then give a one-line recommendation.

IOC (IP address): ${summary.ioc}
AbuseIPDB Confidence Score: ${summary.abuseConfidenceScore}/100 (${summary.totalReports} reports)
VirusTotal: ${summary.vtMalicious} malicious, ${summary.vtSuspicious} suspicious, ${summary.vtHarmless} harmless (reputation: ${summary.vtReputation})
Country: ${summary.countryCode}
ISP/Org: ${summary.isp}
Open Ports (Shodan): ${summary.shodanPorts.join(', ')}

Verdict:`;

return [{ json: summary }];
```

Building the prompt here rather than in the LLM node is deliberate. It means each branch can supply its own tailored prompt, and a single LLM node can serve both paths.

For 8.8.8.8 the output is a tidy intelligence summary:

```json
{
  "ioc": "8.8.8.8",
  "abuseConfidenceScore": 0,
  "totalReports": 120,
  "countryCode": "US",
  "isp": "Google LLC",
  "vtMalicious": 0,
  "vtSuspicious": 0,
  "vtHarmless": 54,
  "vtReputation": 550,
  "shodanPorts": [443, 53],
  "shodanOrg": "Google LLC"
}
```

---

## Step 5 — The hash path

Hashes only work with VirusTotal, so this branch is simpler. Off the Switch's `hash` output:

### VirusTotal (file)

- Method: GET
- URL: `https://www.virustotal.com/api/v3/files/{{ $('Switch').item.json.ioc }}`
- Credential: VirusTotal
- Send Query Parameters: **OFF** (the hash goes in the URL path, not as a query string)

### Hash Summary

A Code node that mirrors the IP merge, with its own prompt:

```javascript
const vt = $('VT-Hash').item.json.data.attributes;

const summary = {
  ioc: $('Switch').item.json.ioc,
  type: 'file hash',
  vtMalicious: vt.last_analysis_stats.malicious,
  vtSuspicious: vt.last_analysis_stats.suspicious,
  vtHarmless: vt.last_analysis_stats.harmless,
  vtUndetected: vt.last_analysis_stats.undetected,
  fileName: vt.meaningful_name || 'unknown',
  fileType: vt.type_description || 'unknown',
  fileSize: vt.size || 0
};

summary.prompt = `You are a SOC analyst. Based on the following threat intelligence data, write a concise verdict (2-3 sentences) on whether this file is safe, suspicious, or malicious. Then give a one-line recommendation.

IOC (file hash): ${summary.ioc}
VirusTotal: ${summary.vtMalicious} malicious, ${summary.vtSuspicious} suspicious, ${summary.vtHarmless} harmless, ${summary.vtUndetected} undetected
File Name: ${summary.fileName}
File Type: ${summary.fileType}
File Size: ${summary.fileSize} bytes

Verdict:`;

return [{ json: summary }];
```

---

## Step 6 — The local LLM verdict

Both branches converge here.

Add a **Basic LLM Chain** node and connect both `IP merge` and `Hash Summary` to its input. Connect an **Ollama Chat Model** to its Model input, using the Ollama credential and `qwen3.6:35b`.

In the Basic LLM Chain:
- Source for Prompt: **Define below**
- Prompt: `{{ $json.prompt }}` (expression)

That is the whole configuration. Each branch already built its own prompt, so the LLM node just consumes whatever arrives.

One addition worth making: under **Chat Messages**, add a **System** message with:

```
Write in plain text without markdown formatting.
```

Without this the model wraps its verdict in markdown (`**safe**`), and the form renders those asterisks literally. Putting it in the system message applies it to both branches from one place.

> **Connecting n8n to Ollama:** n8n runs in Docker, so `localhost` inside the container is not your host machine. Use your machine's LAN IP: run `hostname -I | awk '{print $1}'` and set the Ollama credential base URL to `http://YOUR_IP:11434`. `host.docker.internal` does not work on Linux.

---

## Step 7 — Show the verdict on the form

Without this step the verdict only exists inside n8n, which defeats the purpose of a form.

Add an **n8n Form** node after the Basic LLM Chain, set to the completion/ending operation:

- Completion Title: `IOC Enrichment Complete`
- Completion Message: `{{ $json.text }}` (expression)

The Basic LLM Chain outputs its result in a field called `text`. With the form trigger's **Respond When** set to **Workflow Finishes**, the verdict now renders directly in the browser after submission.

---

## Testing it

Three test cases, chosen to probe different behaviors.

### A clean IP: 8.8.8.8

> Verdict: This IP is classified as safe and represents Google's legitimate public DNS infrastructure rather than a threat. Despite 120 AbuseIPDB reports, the 0/100 confidence score, zero malicious detections across platforms, and association with Google LLC confirm it is essential internet backbone infrastructure serving standard DNS and secure web ports.
>
> Recommendation: Whitelist this address to prevent unintended DNS resolution failures for critical internal and external services.

Note that it reasoned *past* the 120 abuse reports rather than being alarmed by the raw number.

### A malicious IP: 185.220.101.34 (a Tor exit node)

> Verdict: This IP is malicious, identified as a Tor exit node heavily associated with abuse (96/100 confidence on AbuseIPDB) and confirmed detections on VirusTotal, indicating frequent use for anonymized attacks or malicious infrastructure.
>
> Recommendation: Block this IP immediately at the perimeter and audit internal logs for any recent connections to assess potential compromise or data exfiltration.

The "audit internal logs" recommendation is not something I prompted for. That is incident response thinking.

### A tricky hash: 44d88612fea8a8f36de82e1278abb02f (EICAR)

This is the EICAR antivirus test file. Every engine flags it. VirusTotal reported **61 malicious detections**, which a naive tool would report as a critical threat.

> Verdict: This file is safe and expected; it is the standardized EICAR antivirus test string, which is intentionally crafted to trigger detection by every major security vendor without containing any actual malicious payload. The high VirusTotal score reflects this predictable, non-malicious behavior rather than a genuine threat.
>
> Recommendation: Quarantine and delete the file as part of routine AV validation, but no further incident response or user notification is required.

This is the test that convinced me the pipeline was worth building. The model recognized EICAR, understood *why* it was flagged, correctly downgraded the verdict, and still recommended sensible hygiene. That is context-aware judgment, not pattern matching on a number.

---

## Things that bit me

**Stale node references.** Both bugs I hit were the same class of mistake: a Code node referencing `$('If')` after I deleted the IF node, and `$('VirusTotal File')` when the node was actually named `VirusTotal HTTP Request`. If a node errors with "Referenced node doesn't exist" or "Cannot read properties of undefined," check the exact node names on the canvas first.

**VirusTotal's rate limit.** The free tier allows 4 requests per minute. Testing rapidly returns an error object with no `data` field, which crashes the merge Code node when it tries to read `.attributes`. If a Code node suddenly fails on a field that worked a minute ago, suspect the rate limit before suspecting the code.

**API keys in screenshots.** During the build I had keys sitting as plain text in HTTP Request node parameters, and they ended up visible in screenshots. I rotated all three keys and moved them into n8n credentials, where they are stored encrypted and display as hidden.

Set up credentials *before* you start taking screenshots:

| Service | Credential type |
|---|---|
| VirusTotal | Dedicated VirusTotal credential |
| AbuseIPDB | Generic Credential Type, Header Auth (`Key`) |
| Shodan | Generic Credential Type, Query Auth (`key`, lowercase) |

---

## Why this matters for DFIR

This is not a forensics tool. It does not carve disks or parse memory. What it automates is the *glue* around forensics: enrichment, correlation, and documentation. That glue is where a lot of investigation time actually goes.

It is also, functionally, a small SOAR platform. The pattern here (ingest an indicator, enrich from multiple sources, correlate, produce a verdict) is exactly what Splunk SOAR and Cortex XSOAR do at enterprise scale. Building it by hand in n8n is a good way to understand what those platforms are doing under the hood.

And because the verdict step runs on a local model, no indicator data from an investigation ever leaves the machine.

---

## What's next

The obvious extension is to stop typing indicators in by hand. I have an Ubuntu VM running Atomic Red Team and a Wazuh deployment already in the lab, which suggests a full purple-team loop:

```
Atomic Red Team fires an ATT&CK technique
        |
   Wazuh detects it and generates an alert
        |
   n8n receives the alert via webhook
        |
   This enrichment pipeline runs on the IOCs in the alert
        |
   Enriched verdict lands in my notifications
```

Simulate, detect, enrich, respond. That is the next project.
