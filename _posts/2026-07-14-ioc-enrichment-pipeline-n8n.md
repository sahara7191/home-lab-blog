---
title: "Building an IOC Enrichment Pipeline with n8n and a Local LLM (Part1)"
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
<br>

## Workflow Overview

![IOC Workflow]({{ "/assets/images/IOC-Workflow.png" | relative_url }})
<br>

## Prerequisites

- A self-hosted n8n instance and Ollama with a capable model. I use `sylink:8b` *(enterprise cybersecurity AI for threat intelligence, incident response, and security operations)*.

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

<br>

## Step 2: Route with a Switch

The pipeline routes IPs and hashes to different endpoints. A **Switch** node with regex does the detection and routing in one node.

Add a **Switch** node and set parameters in Mode **Rules**:

**<u>Routing Rule 1 — IP addresses</u>**

- **Value 1** (Expression): {% raw %}`{{ $json.ioc }}`{% endraw %}
- **Operator**: *String - matches regex*
- **Value 2** (Fixed): `^(\d{1,3}\.){3}\d{1,3}$`
- **Rename Output**: *ON*
- **Output Name**: *ip*
<br>

**<u>Routing Rule 2 — Hashes</u>** 

- **Value 1** (Expression): {% raw %}`{{ $json.ioc }}`{% endraw %}
- **Operator**: *String - matches regex*
- **Value 2** (Fixed): `^[a-fA-F0-9]{32}$|^[a-fA-F0-9]{40}$|^[a-fA-F0-9]{64}$`
- **Rename Output**: *ON*
- **Output Name**: *hash*

*The hash regex cover MD5 (32), SHA1 (40), and SHA256 (64)*

<br>

## Step 3: The IP Path

Three HTTP Request nodes chained off the Switch's `ip` output. 


**<u>AbuseIPDB</u>**

- **Method**: `GET`
- **URL**: `https://api.abuseipdb.com/api/v2/check`
- **Authentication**: *Generic Credential Type - Header Auth* (Name: `Key`)
- **Send Query Parameters**: *ON*
  - Query Parameter 1 *(Name=Value)*: `ipAddress` = {% raw %}`{{ $json.ioc }}`{% endraw %}
  - Query Parameter 1 *(Name=Value)*: `maxAgeInDays` = `90`
- **Send Headers**: *ON*
  - Header *(Name=Value)*: = `Accept` = `application/json`
<br>

**<u>VirusTotal (VT-IP)</u>**

- **Method**: `GET`
- **URL**: {% raw %}`https://www.virustotal.com/api/v3/ip_addresses/{{ $('Switch').item.json.ioc }}`{% endraw %}
- **Credentials**: *Your API for VirusTotal*
<br> 

**<u>Shodan</u>**

- **Method**: `GET`
- **URL**: {% raw %}`https://api.shodan.io/shodan/host/{{ $('Switch').item.json.ioc }}`{% endraw %}
- **Authentication**: *Generic Credential Type - Query Auth* (Name: `key`, lowercase)


> Shodan's parameter name is case-sensitive. `Key` instead of `key` returns 401 Unauthorized with a misleading "check your credentials" message even though the key is fine.

<br>

## Step 4: Correlate the IP Results

Add a **Code** node for JavaScript after Shodan (I renamed mine to ***IP merge***). It pulls the useful fields from all three responses and builds the prompt.

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

// Decide the verdict in code — the model must not override it
let verdict = 'CLEAN';
if (summary.abuseConfidenceScore >= 75 || summary.vtMalicious >= 5) {
  verdict = 'MALICIOUS';
} else if (summary.abuseConfidenceScore >= 25 || summary.vtMalicious >= 2) {
  verdict = 'SUSPICIOUS';
}
summary.verdict = verdict;

summary.prompt = `You are a SOC analyst. The verdict has already been determined: ${verdict}. Do not change it. Write a report in EXACTLY this format:

## Verdict: ${verdict}
- fact one
- fact two
- fact three

**Recommendation:** One short sentence.

Rules: line 1 must be exactly "## Verdict: ${verdict}". Use 4 to 6 bullets, each one fact under 15 words. The bullets must justify the ${verdict} verdict using the data below. No paragraphs, no introduction, no text after the recommendation.

Threat intelligence data:
IOC (IP address): ${summary.ioc}
AbuseIPDB Confidence Score: ${summary.abuseConfidenceScore}/100 (${summary.totalReports} reports)
VirusTotal: ${summary.vtMalicious} malicious, ${summary.vtSuspicious} suspicious, ${summary.vtHarmless} harmless (reputation: ${summary.vtReputation})
Country: ${summary.countryCode}
ISP/Org: ${summary.isp}
Open Ports (Shodan): ${summary.shodanPorts.join(', ') || 'none found'}`;

return [{ json: summary }];
```

Building the prompt here (not in the LLM node) lets each branch supply its own prompt while one LLM node serves both. A few things in this prompt took several rewrites to get right:

> **Give the model a precise instruction, not an opinion.** My first prompt just asked for "a verdict on whether this IP is safe, suspicious, or malicious." Running the same IP twice resulted in a verdict of MALICIOUS one time and SUSPICIOUS the next. The "Verdict criteria" block fixes that: the model applies thresholds I chose instead of deciding for itself.

> **Format by example.** Describing the format in words ("be concise, use markdown") barely worked. Showing the exact skeleton plus strict counts ("4 to 6 bullets, under 15 words") is what made local models comply. Moreover, my original prompt ended with a simple "Verdict:" line, which invited prose instead of a ## Verdict: heading. Deleting that line was half the fix.

<br>

## Step 5: The Hash Path

Hashes I connected only VirusTotal, so this branch is shorter.

**<u>VirusTotal (VT-Hash)</u>**

- **Method**: `GET`
- **URL**: {% raw %}`https://www.virustotal.com/api/v3/files/{{ $('Switch').item.json.ioc }}`{% endraw %}
- **Credential**: *Your API for VirusTotal*
<br>

**<u>Hash Summary</u>** — a Code node that mirrors IP merge. 

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

// Decide the verdict in code — the model must not override it
let verdict = 'CLEAN';
if (summary.vtMalicious >= 5) {
  verdict = 'MALICIOUS';
} else if (summary.vtMalicious >= 2) {
  verdict = 'SUSPICIOUS';
}
// Known AV test files (EICAR) are always CLEAN regardless of detections
const lowerLabel = (summary.threatLabel || '').toLowerCase();
const lowerTags = (summary.tags || '').toLowerCase();
const lowerNames = (summary.names || '').toLowerCase();
const isTestFile = lowerLabel.includes('eicar') || lowerTags.includes('eicar') || lowerNames.includes('eicar');
if (isTestFile) {
  verdict = 'CLEAN';
}
summary.verdict = verdict;
summary.isTestFile = isTestFile;

summary.prompt = `You are a SOC analyst. The verdict has already been determined: ${verdict}. Do not change it. Write a report in EXACTLY this format:

## Verdict: ${verdict}
- fact one
- fact two
- fact three

**Recommendation:** One short sentence.

Rules: line 1 must be exactly "## Verdict: ${verdict}". Use 4 to 7 bullets, each one fact under 15 words. The bullets must justify the ${verdict} verdict using the data below. Prioritize: malware family or threat label, detection count, threat categories, notable tags, first seen date, file identity.${isTestFile ? ' Include a bullet noting this is a known antivirus test file.' : ''} No paragraphs, no introduction, no text after the recommendation.

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

> Early on I fed the model only detection counts and file size, and verdicts were correct but thin ("57 engines flagged it"). This version pulls the malware family label, threat categories, tags, and first-seen date.

<br>

## Step 6: The Local LLM Verdict

Both branches converge here. Add a **Basic LLM Chain** node, connect both `IP merge` and `Hash Summary` to its input, and connect an **Ollama Chat Model** (mine is `sylink:8b`) to its Model input.

**<u>Basic LLM Chain</u>**:

- **Source for Prompt**: `Define below`
- **Prompt**: {% raw %}`{{ $json.prompt }}`{% endraw %}
- **Chat Messages**: *(leave empty)*

> **Leave Chat Messages empty.** My earlier build had a System message reading "Write in plain text without markdown formatting." Once I switched to a markdown display (Step 7), that message fought the branch prompts: they asked for markdown, it forbade markdown, and the model produced malformed output with no verdict heading to style. Deleting message from this node fixed everything. 

**<u>Ollama Chat Model</u>**:

- **Sampling Temperature**: `0` *(Same input gives the same output)*
- **Enable Thinking**: *off*.  *(some models reason before answering, which a rubric does not need, and it runs faster off)*

> **Do not cap "Max Tokens to Generate" on a thinking model.** The cap counts hidden reasoning tokens too, so a limit of 400 gets eaten by the thinking phase and leaves a gutted verdict with no explanation. Leave it at -1, or turn thinking off first. Temperature 0 plus the strict format keeps length under control anyway.

> **Connecting n8n to Ollama:** n8n runs in Docker, so `localhost` is the container, not the host. Use the host IP: `hostname -I | awk '{print $1}'`, then set the Ollama credential base URL to `http://YOUR_IP:11434`. 

<br>

## Step 7: Style the Verdict on the Form

A raw text dump is hard to read; an analyst wants red or green at a glance. Three nodes turn the model's markdown into a coloured banner.

**<u>Markdown node</u>** (after Basic LLM Chain)

- **Mode**: `Markdown to HTML`
- **Markdown**: {% raw %}`{{ $json.text }}`{% endraw %}
- **Destination Key**: `data`
<br>

**<u>Style Verdict</u>** — a Code node that tags the verdict heading with a CSS class. n8n's Markdown converter adds an `id` to headings, so this matches with a regex that tolerates attributes.

```javascript
let html = $input.first().json.data;
html = html.replace(/<h2([^>]*)>(\s*Verdict:\s*MALICIOUS)/i, '<h2$1 class="mal">$2');
html = html.replace(/<h2([^>]*)>(\s*Verdict:\s*SUSPICIOUS)/i, '<h2$1 class="sus">$2');
html = html.replace(/<h2([^>]*)>(\s*Verdict:\s*CLEAN)/i, '<h2$1 class="clean">$2');
return { json: { data: html } };
```

> My first version matched a bare `<h2>` and silently did nothing, leaving a white-on-white invisible banner. Match the attributes.

**<u>Complete Form (n8n Form)</u>** 

- **Page Type**: `Form Ending`
- **On n8n Form Submission**: `Show Text`
- **Text field** (expression mode): *paste the HTML block provided below*

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
<br>

Final chain: **<u>Basic LLM Chain → Markdown → Style Verdict → Form Ending.</u>** Submit an indicator and the styled verdict renders in the browser: green for clean, amber for suspicious, red for malicious.

<br>

## Testing

Four cases, each probing a different behavior *(see screenshots below)*.

`8.8.8.8`
- Verdict: 🟢 CLEAN
- Proof: Weights the 0/100 abuse score and Google ownership past 120 stale reports
<br>

`92.141.179.85`
- Verdict: 🟠 SUSPICIOUS
- Proof: Lands in the middle band (VT 1–4 or AbuseIPDB 25–74)
<br>

`db349b97c37d22f5ea1d1841e3c89eb4` (WannaCry)
- Verdict: 🔴 MALICIOUS
- Proof: High detection count and a clear malware-family label drive an unambiguous verdict
<br>

`44d88612fea8a8f36de82e1278abb02f` (EICAR)
- Verdict: 🟢 CLEAN
- Proof: Recognized the AV test file despite dozens of detections.

<br>

![Test for Google IP]({{ "/assets/images/test-googleIP.png" | relative_url }})

![Test for suspicious IP]({{ "/assets/images/test-suspiciousIP.png" | relative_url }})

![Test for WannaCry hash]({{ "/assets/images/test-wannahash.png" | relative_url }})

![Test for EICAR hash]({{ "/assets/images/test-eicarhash.png" | relative_url }})

<br>

> The EICAR case is the one that convinced me this was worth building: the criteria carve out known test files while VirusTotal's malicious score is 66, so the tool understands why it is flagged rather than pattern-matching on a number.

> Do not trust the banner alone during testing! After every test *(I ran much more than I provided here)*, I checked the VT node's outputs and confirmed the malicious count matches on the VirusTotal website. 

<br>

## Conclusion

Early versions of this project asked the model to judge the indicator, and it flip-flopped: the same IP came back CLEAN one run and MALICIOUS the next, once even ignoring 17 VirusTotal detections. Moving the verdict into the Code nodes fixed that. Plain "if" statements apply the thresholds I chose, and the model only explains the evidence behind a decision already made. Deterministic logic decides, the model narrates.
Those thresholds are my policy, not universal truth, and they live in one place where anyone can read and tune them. That is the real lesson: a good automated verdict is not the model being clever; it is an explicit rule applied the same way every time, with the reasoning written in plain language on top. The lesson in this lab was figuring out who should make the call: the model or the code. AI is just a good assistant, not a replacement. 

This is my first automation attempt, and I know it looks very simple. I'm very excited to learn something new and have hands-on practice.

## What's Next

This pipeline is Part 1 of a series. The roadmap:

**Part 2: URL and domain branches.** Two new indicator types on the same pattern: a new Switch rule, new HTTP requests, a new merge node, and the same LLM chain. 

**Part 3: Phishing email triage.** The bigger jump: upload an `.eml` file instead of typing an indicator, parse out the sender IP, embedded links, and attachment hashes, then feed each extracted IOC through the existing branches. 

**The finale: a purple-team loop.** I have an Ubuntu VM running Atomic Red Team and a Wazuh deployment in the lab:

​```
Atomic Red Team fires an ATT&CK technique
        |
   Wazuh detects it and generates an alert
        |
   n8n receives the alert via webhook
        |
   This enrichment pipeline runs on the IOCs in the alert
        |
   Enriched verdict lands in my notifications
​```

Simulate, detect, enrich, respond. No indicator typed by hand anywhere in the chain.
