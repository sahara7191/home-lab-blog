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
<br>

Using n8n for Indicator of Compromise (IOC) enrichment allows you to build powerful, automated threat intelligence pipelines. Instead of analysts manually querying multiple sources, such as  VirusTotal, AbuseIPDB, Shodan, etc., n8n automatically queries intelligence platforms to determine if an IP, domain, hash, or URL is malicious.

This post builds a simple n8n pipeline that automates the loop: paste an indicator, it queries threat intel sources, correlates them, and has a local LLM write a plain-language verdict. 

Runs on the Fedora AI workstation from my previous posts: [Part 1](/home-lab-blog/homelab/fedora-ai-setup-part1/) (OS and ROCm), [Part 2](/home-lab-blog/homelab/fedora-ai-setup-part2/) (Ollama, Open WebUI, n8n).

## Workflow Overview

![IOC Workflow]({{ "/assets/images/IOC-Workflow.png" | relative_url }})


## Prerequisites

- A self-hosted n8n instance and Ollama with a capable model. I use `qwen3.6:35b`.

- API keys from sources:

| Source | Provides | Link | 
| --- | --- | --- | 
| AbuseIPDB | Reputation data (abuse confidence score) | [abuseipdb.com](https://www.abuseipdb.com) | 
| VirusTotal | Vendor detection (flags across ~90 engines) | [virustotal.com](https://www.virustotal.com) | 
| Shodan | Port and service exposure | [shodan.io](https://www.shodan.io) |

One source alone is easy to misread. Together they let the model weigh agreement and conflict between signals before committing to a verdict.
<br>


## Step 1: The Form Trigger

Add an **On form submission** node:

- **Form Title**: `IOC Enrichment`
- **Form Description**: `Paste an IP address or file hash`
- **Form Element**: Text Input - Label `ioc`

Execute the step, open the Test URL, and submit `8.8.8.8`. The output shows your input echoed back (`ioc: 8.8.8.8`).


## Step 2: Route with a Switch

The pipeline routes IPs and hashes to different endpoints. A **Switch** node with regex does the detection and routing in one node.

Add a **Switch** node and set parameters in Mode **Rules**:

**Routing Rule 1 — IP addresses**

- **Value 1** (Expression): {% raw %}`{{ $json.ioc }}`{% endraw %}
- **Operator**: *String - matches regex*
- **Value 2** (Fixed): `^(\d{1,3}\.){3}\d{1,3}$`
- **Rename Output**: *ON*
- **Output Name**: *ip*

**Routing Rule 2 — Hashes** *(The hash regex cover MD5 (32), SHA1 (40), and SHA256 (64))*

- **Value 1** (Expression): {% raw %}`{{ $json.ioc }}`{% endraw %}
- **Operator**: *String - matches regex*
- **Value 2** (Fixed): `^[a-fA-F0-9]{32}$|^[a-fA-F0-9]{40}$|^[a-fA-F0-9]{64}$`
- **Rename Output**: *ON*
- **Output Name**: *hash*


## Step 3: The IP Path

Three HTTP Request nodes chained off the Switch's `ip` output.

**<u>AbuseIPDB</u>**

- **Method**: *GET*
- **URL**: `https://api.abuseipdb.com/api/v2/check`
- **Authentication**: *Generic Credential Type - Header Auth* (Name: `Key`)
- **Send Query Parameters**: *ON*
  - Query Parameter 1 *(Name=Value)*: `ipAddress` = {% raw %}`{{ $json.ioc }}`{% endraw %}
  - Query Parameter 1 *(Name=Value)*: `maxAgeInDays` = `90`
- **Send Headers**: *ON*
  - Header *(Name=Value)*: = `Accept` = `application/json`
<br>
<br>

**<u>VirusTotal (IP)</u>**

- **Method**: *GET*
- **URL**: {% raw %}`https://www.virustotal.com/api/v3/ip_addresses/{{ $('Switch').item.json.ioc }}`{% endraw %}
- **Credential**: *VirusTotal*

> Note the `$('Switch')` reference. AbuseIPDB's response replaced the item data, so the IP has to be pulled back from the Switch node, not the previous node.
<br>

**<u>Shodan</u>**

- **Method**: *GET*
- **URL**: {% raw %}`https://api.shodan.io/shodan/host/{{ $('Switch').item.json.ioc }}`{% endraw %}
- **Authentication**: *Generic Credential Type - Query Auth* (Name: `key`, lowercase)

> Shodan's parameter name is case-sensitive. `Key` instead of `key` returns 401 Unauthorized with a misleading "check your credentials" message even though the key is fine.

## Step 4: Correlate the IP Results

Add a **Code** node named `IP merge` after Shodan. It pulls the useful fields from all three responses and builds the prompt.

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

summary.prompt = `You are a SOC analyst reviewing threat intelligence for an IP address. Respond in EXACTLY this format, nothing else:

## Verdict: CLEAN
- fact one
- fact two
- fact three

**Recommendation:** One short sentence.

Rules: line 1 must be exactly "## Verdict: MALICIOUS" or "## Verdict: SUSPICIOUS" or "## Verdict: CLEAN". Use 4 to 6 bullets, each one fact under 15 words. No paragraphs, no introduction, no extra text after the recommendation.

Verdict criteria: MALICIOUS if AbuseIPDB confidence is 75 or higher, OR VirusTotal malicious count is 10 or higher. SUSPICIOUS if AbuseIPDB confidence is 25-74, OR VirusTotal malicious count is 3-9. CLEAN otherwise.

Threat intelligence data:
IOC (IP address): ${summary.ioc}
AbuseIPDB Confidence Score: ${summary.abuseConfidenceScore}/100 (${summary.totalReports} reports)
VirusTotal: ${summary.vtMalicious} malicious, ${summary.vtSuspicious} suspicious, ${summary.vtHarmless} harmless (reputation: ${summary.vtReputation})
Country: ${summary.countryCode}
ISP/Org: ${summary.isp}
Open Ports (Shodan): ${summary.shodanPorts.join(', ') || 'none found'}`;

return [{ json: summary }];
```

Building the prompt here (not in the LLM node) lets each branch supply its own prompt while one LLM node serves both. Two things in this prompt took several rewrites to get right:

> **Give the model a rubric, not an opinion.** My first prompt just asked for "a verdict on whether this IP is safe, suspicious, or malicious." Running the same IP twice gave MALICIOUS one time and SUSPICIOUS the next. The "Verdict criteria" block fixes that: the model applies thresholds I chose instead of deciding for itself. Those thresholds are the part worth tuning to your own environment.

> **Format by example.** Describing the format in words ("be concise, use markdown") barely worked. Showing the exact skeleton plus hard counts ("4 to 6 bullets, under 15 words") is what made local models comply. Also: my original prompt ended with a lone `Verdict:` line, which invited prose instead of a `## Verdict:` heading. Deleting that line was half the fix.

## Step 5: The Hash Path

Hashes only work with VirusTotal, so this branch is shorter.

**VirusTotal (file)**

- **Method**: *GET*
- **URL**: {% raw %}`https://www.virustotal.com/api/v3/files/{{ $('Switch').item.json.ioc }}`{% endraw %}
- **Credential**: *VirusTotal*
- **Send Query Parameters**: *OFF* (the hash is in the URL path)
<br>

**Hash Summary** — a Code node that mirrors IP merge. Early on I fed the model only detection counts and file size, and verdicts were correct but thin ("57 engines flagged it"). This version pulls the malware family label, threat categories, tags, and first-seen date too, which turns a detection count into actual intelligence.

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
  fileSize: vt.size || 0,
  threatLabel: (vt.popular_threat_classification && vt.popular_threat_classification.suggested_threat_label) || 'none',
  threatCategories: (vt.popular_threat_classification && vt.popular_threat_classification.popular_threat_category)
    ? vt.popular_threat_classification.popular_threat_category.map(c => c.value).join(', ')
    : 'none',
  tags: (vt.tags || []).join(', ') || 'none',
  firstSeen: vt.first_submission_date ? new Date(vt.first_submission_date * 1000).toISOString().split('T')[0] : 'unknown',
  lastSeen: vt.last_analysis_date ? new Date(vt.last_analysis_date * 1000).toISOString().split('T')[0] : 'unknown',
  names: (vt.names || []).slice(0, 5).join(', ') || 'unknown'
};

summary.prompt = `You are a SOC analyst reviewing threat intelligence for a file hash. Respond in EXACTLY this format, nothing else:

## Verdict: CLEAN
- fact one
- fact two
- fact three

**Recommendation:** One short sentence.

Rules: line 1 must be exactly "## Verdict: MALICIOUS" or "## Verdict: SUSPICIOUS" or "## Verdict: CLEAN". Use 4 to 7 bullets, each one fact under 15 words. Prioritize: malware family or threat label, detection count, threat categories, notable tags, first seen date, file identity. No paragraphs, no introduction, no extra text after the recommendation.

Verdict criteria: MALICIOUS if VirusTotal malicious count is 10 or higher. SUSPICIOUS if 3-9. CLEAN if 0-2. Exception: known antivirus test files (such as EICAR) are CLEAN regardless of detection count, with a bullet noting it is a test file.

Threat intelligence data:
IOC (file hash): ${summary.ioc}
VirusTotal: ${summary.vtMalicious} malicious, ${summary.vtSuspicious} suspicious, ${summary.vtHarmless} harmless, ${summary.vtUndetected} undetected
Threat label: ${summary.threatLabel}
Threat categories: ${summary.threatCategories}
Tags: ${summary.tags}
File Name: ${summary.fileName}
Known aliases: ${summary.names}
File Type: ${summary.fileType}
File Size: ${summary.fileSize} bytes
First submitted to VT: ${summary.firstSeen}
Last analyzed: ${summary.lastSeen}`;

return [{ json: summary }];
```

> The EICAR exception in the criteria matters. Without it, "10 or more detections is MALICIOUS" would flag the standard antivirus test file as a critical threat, exactly the naive behavior the rubric is meant to prevent.

## Step 6: The Local LLM Verdict

Both branches converge here. Add a **Basic LLM Chain** node, connect both `IP merge` and `Hash Summary` to its input, and connect an **Ollama Chat Model** to its Model input (`qwen3.6:35b`).

Basic LLM Chain settings:

- **Source for Prompt**: *Define below*
- **Prompt**: {% raw %}`{{ $json.prompt }}`{% endraw %}
- **Chat Messages**: *leave empty*

> **Leave Chat Messages empty.** My earlier build had a System message reading "Write in plain text without markdown formatting." Once I switched to a markdown display (Step 7), that message fought the branch prompts: they asked for markdown, it forbade markdown, and the model produced malformed output with no verdict heading to style. Deleting it fixed everything. When two instructions reach the model, they have to agree.

On the **Ollama Chat Model** node, add these options:

- **Sampling Temperature**: `0`. Same input gives the same output, which a verdict tool needs.
- **Enable Thinking**: *off*. `qwen3.6` reasons before answering, which a rubric does not need, and it runs faster off.

> **Do not cap "Max Tokens to Generate" on a thinking model.** The cap counts hidden reasoning tokens too, so a limit of 400 gets eaten by the thinking phase and leaves a gutted verdict with no explanation. Leave it at -1, or turn thinking off first. Temperature 0 plus the strict format keeps length under control anyway.

> **Connecting n8n to Ollama:** n8n runs in Docker, so `localhost` is the container, not the host. Use the host LAN IP: `hostname -I | awk '{print $1}'`, then set the Ollama credential base URL to `http://YOUR_IP:11434`. `host.docker.internal` does not work on Linux.

## Step 7: Style the Verdict on the Form

A raw text dump is hard to read; an analyst wants red or green at a glance. Three nodes turn the model's markdown into a coloured banner.

**Markdown node** (after Basic LLM Chain)

- **Mode**: *Markdown to HTML*
- **Markdown**: {% raw %}`{{ $json.text }}`{% endraw %}
- **Destination Key**: `data`
<br>

**Style Verdict** — a Code node that tags the verdict heading with a CSS class. n8n's Markdown converter adds an `id` to headings, so this matches with a regex that tolerates attributes.

```javascript
let html = $input.first().json.data;
html = html.replace(/<h2([^>]*)>(\s*Verdict:\s*MALICIOUS)/i, '<h2$1 class="mal">$2');
html = html.replace(/<h2([^>]*)>(\s*Verdict:\s*SUSPICIOUS)/i, '<h2$1 class="sus">$2');
html = html.replace(/<h2([^>]*)>(\s*Verdict:\s*CLEAN)/i, '<h2$1 class="clean">$2');
return { json: { data: html } };
```

> My first version matched a bare `<h2>` and silently did nothing, leaving a white-on-white invisible banner. Match the attributes.

**Form Ending node** — set Operation to **Show Text**, and paste this in the Text field (expression mode). The CSS colours the banner, with a grey fallback if the class ever fails to attach.

{% raw %}
```html
<style>
  .verdict-wrap { font-family: system-ui, sans-serif; max-width: 700px;
    margin: 20px auto; padding: 0 16px; text-align: left; }
  .verdict-wrap h2 { padding: 12px 16px; border-radius: 8px; color: #111; background: #e5e7eb; }
  .verdict-wrap h2.mal   { background: #b91c1c; color: #fff; }
  .verdict-wrap h2.sus   { background: #b45309; color: #fff; }
  .verdict-wrap h2.clean { background: #15803d; color: #fff; }
  .verdict-wrap li { margin: 6px 0; }
  .verdict-wrap code { background: #eee; padding: 2px 5px; border-radius: 4px; }
</style>
<div class="verdict-wrap">
  <p><strong>IOC:</strong> <code>{{ $('Switch').item.json.ioc }}</code></p>
  {{ $json.data }}
</div>
```
{% endraw %}

Final chain: **Basic LLM Chain → Markdown → Style Verdict → Form Ending.** Submit an indicator and the styled verdict renders in the browser: green for clean, amber for suspicious, red for malicious.

## Testing

Three cases, each probing a different behavior. Space tests a minute apart to respect VirusTotal's 4/min limit.

| Test IOC | Expected | What it proves |
| --- | --- | --- |
| `8.8.8.8` | Green CLEAN | Reasons past 120 abuse reports, weights the 0/100 score and Google ownership |
| `45.155.205.233` | Red MALICIOUS | Rubric resolves conflicting sources (VT detections vs low AbuseIPDB score) the same way every run |
| `44d88612fea8a8f36de82e1278abb02f` (EICAR) | Green CLEAN | Recognizes the AV test file despite dozens of detections, recommends hygiene not IR |

The EICAR case is the one that convinced me this was worth building: the criteria carve out known test files, so the tool understands *why* it is flagged rather than pattern-matching on a number.

## Things That Bit Me

| Symptom | Cause | Fix |
| --- | --- | --- |
| Banner never colours, output subtly wrong | Leftover "plain text" System message fighting the markdown prompt | Delete the System message |
| Empty or stunted verdict | Token cap counting hidden thinking tokens | Leave cap at -1, or disable thinking |
| Same IOC, different verdicts | Non-deterministic sampling | Temperature 0 + explicit rubric |
| "Referenced node doesn't exist" | Stale node name in a Code node (`$('If')`, wrong VT name) | Match exact canvas names |
| Code node errors on `.attributes` | Rate-limited API returned an error object, no `data` field | Wait out the 4/min limit |
| Invisible white-on-white heading | Style regex matched bare `<h2>`, missed the `id` attribute | Match with attributes |

## Why This Matters for DFIR

This is not a forensics tool. It does not carve disks or parse memory. It automates the *glue* around forensics: enrichment, correlation, documentation, where a lot of investigation time actually goes.

Functionally it is a small SOAR platform. Ingest an indicator, enrich from multiple sources, correlate, produce a verdict, that is what Splunk SOAR and Cortex XSOAR do at enterprise scale. Building it by hand is a good way to understand what those platforms do under the hood.

> **One honest limitation.** The rubric lives inside the prompt, so the model still does the final classification. That is fine for a portfolio tool. The more robust design computes the verdict in the Code node with plain `if` statements and lets the LLM only *explain* the evidence: deterministic logic decides, the model narrates. That is the direction for anything near production.

And because the verdict step runs on a local model, no indicator data ever leaves the machine.

## What's Next

**A security-specialized model.** I run `qwen3.6:35b`, a general model. Cisco's Foundation-Sec is a Llama-3.1-8B continued-pretrained on security data. The plain base model is not instruction-tuned, so a drop-in verdict node needs an instruct or reasoning variant. Worth a future test: does a small security-tuned model match a larger general one here? (Skip the "uncensored" or "blackhat" models. They are usually old base models with safety training stripped, which degrades instruction-following, and running unknown model files on a lab network is its own risk. Defensive enrichment is not something mainstream models refuse anyway.)

**Stop typing indicators by hand.** I have an Ubuntu VM running Atomic Red Team and a Wazuh deployment in the lab, which suggests a full purple-team loop:

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
