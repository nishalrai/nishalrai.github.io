---
title: When DDoS Protection Starts Hurting Operations
description: >-
  This article focuses on the operational design decisions behind enhancing an existing DDoS deployment. All infrastructure identifiers, IP addresses, interface labels, domains, and event values shown here have been sanitized and replaced with dummy data for safe public sharing..
author: nishalrai
date: 2026-04-17 10:55:00 +0800
categories: [DDoS, Automation]
tags: [ddos mitigation, automation]
---

In DDoS defense, getting the appliance deployed is rarely the end of the story. In many environmentrs especially in telecom, banking and large distributed infrastructures, the real challenge starts after the platform is already in place. Detection begins to work, alerts begin to flow, baselines start learning, and then operations team realize something important: the appliance is detecting, but is it not neccessarily helping them decide fast enough. 

That being said, this article is not meant to be **another deep dive into DDoS attack types, packet anatomy, or protocol behavior**. It is about something more operationally painful: what happens after deploying a DDoS solution, when the solution itself starts becoming part of the operational problem.

The discussion is based on real implementation pattern, but all sensitive details, *including IP addresses, hostnames, interface names, router labels, URLs, and traffic-specific identifiers, have been replaced with dummy values for safe public sharing.*
<br>

# The Problem Was Not Detection Alone

*DDoS, or Distributed Denial of Service*, remains one of the most visible threats in today’s security landscape. With the rapid growth of compute power, cheap bot infrastructure, and large attack surfaces across internet-facing services, operators are now dealing with mixed traffic patterns, sudden surges in ingestion volume, and an increasingly noisy detection environments.

Banking, telecom, and government sectors continue to face these attacks more frequently than most others. The motives vary, but financial disruption and activism still remain among the most common. In many of these environments, the challenge is not simply whether traffic can be detected as suspicious. The challenge is whether the detection output is useful for the operator to act on without wasting time, missing real incidents, or drowing in false positives.

> That was the exact problem. we started facing in our environment.

The original appliance was doing, what it was designed to do: collecting telemetry, identifying anomalies, and producing event notifications, but operationally, the result became difficult to consume at scale.

At one stage, the event volume and alerting behavior became noisy enough that the DDoS protection stack itself started creating operational fatigue. The issue was no longer only about attack traffic. It was about what happended after the platform raised the event.

![](/assets/img/images/ddos/n8n/emailAnomaly.png){: width="1200" height="800" .shadow .rounded-3}

<br>

# A Quick Look at the Deployment Context
I think it would be useful to breifly explain the environment to stay on the same page before moving forward.

DDoS detection and mitigation deployments generally fall into two broad models:

- **Inline deployment**, which is more common across enterprise environments, where traffic passes directly through mitigation infrastructure.  
- **Out-of-band deployment**, which is usually more appropriate for highly distributed network environments, where traffic telemetry is collected from multiple network points and analyzed centrally before mitigation decisions are pushed elsewhere.

The environement discussed here falls into the second category - **out-of-band deployment.**


A **GenieATM Controller and Collector architecture** was used. In that model:
- **Collectors** receive telemetry such as **sFlow** and **SNMP** from network devices
- they classify traffic and rank Top-N patterns
- they forward processed observations to the **Controller**
- the **Controller** aggregates the data, detects anomalies, and provides the management plane for monitoring and reporting

Once suspicious behavior is detected, traffic can be redirected or mitigation actions can be coordinated with downstream platforms such as **F5 AFM, Radware DefensePro**, or other scrubbing and protection systems.


![](/assets/img/images/ddos/n8n/genieatm.png){: width="1200" height="800" .shadow .rounded-3}

> In terms of solution design perspective, that is already a valid and capable DDoS mitigation framework. But from an operations perspective, it still needed help.


[More details about the GenieATM Networks](https://www.genie-networks.com/) 
<br>

## Where the Real Pain Started
From my experience working across security consulting engagements and multiple vendor platforms, one of the most critical parts of any deployment is not just understanding the product, but understanding the client’s actual problem context.

The four questions that matter most are still the simplest ones:
- **Why is this happening?**
- **Where is it happening?**
- **How is it happening?**
- **When is it happening?**

These four questions usually define whether a deployment becomes operationally useful or just technically complete.

> A successful deployment is never just about getting the appliance online. That is only half the work. 

The second half starts during optimization - tuning thresholds, validating edge cases, mapping false positve, adapting to traffic patterns, and helping operations teams trust the output enough to take action quickly. 

*That was exactly where the crack began to appear.*

The appliance was generating anomaly events, but many of those events still required too much manual effort to understand. In distributed environments with bursty traffic behavior, automatic baselines alone were not enough. 

Some traffic patterns were legitimate but irregular and some events were suspicious but not actionable. Some detections looked correct numerically, but lacked enough surrounding context for analysts to make a confident call.

Over time, the platform became less of a direct answer and more of a partial answer. That is when it became clear that the DDoS solution did not need replacement. It needed reinforcement.

<br>

## Why the Additional Stack Was Necessary
The purpose of the enhancement was not to build another dashboard for the sake of visibility, instead the goal was much more specific:

- Receive anomaly events automatically
- Pull richer event details from the DDoS platform API
- Preserve the important attributes for later analysis
- Enrich the top source IPs with quick reputation context
- Present the event back in a format that an analyst could review fast

In other words, the missing piece was event correlation for operational decision-making.

The existing platform could detect anomalies, but what it could not fully do on its own, in a way that matched the operational need, was turn those anomalies into a quick and readable verdict.

That gap is what the additional stack was built to close.

<br>

## Baselines, False Positives, and the Limits of “Learning”
In theory, baseline learning sounds like the right answer for the opeartion.

A platform observes traffic behavior over time, understands what is normal, and raises a signal when the traffic deviates in a suspicious way. That works reasonably well in stable environments.

But highly distributed network environments are not always stable in that sense.

Traffic changes by region, service, time slot, subscriber behavior, content delivery patterns, upstream events, and transient network conditions. In such environments, automatic learning can still produce noisy outcomes, especially when the platform sees intermittent spikes that are not necessarily attacks but are abnormal enough to trigger detection logic.

That is why most real DDoS operations eventually end up balancing between two modes:

- **Automatic baseline learning**
- **Manual threshold tuning**

Manual thresholds are not always elegant, but they remain operationally necessary. The objective is not perfection. The objective is to keep the platform within a usable detection range — **low enough to catch real attacks, but not so sensitive that every abnormal burst becomes a flood of alerts**.

The enhancement stack was built with this operational pain in mind.

It was designed to help analysts answer:\
**"Was this likely a real attack, an environmental anomaly, or simply a false positive that needs no escalation?"**

<br>


# Stack Choice: Idea and Decision
The stack selection was driven by practicality.


## Discord as the intake channel
Discord turned out to be the most reliable source for event intake from GenieATM webhook notifications in this implementation. It fit well with the workflow approach being built around it.

Microsoft Teams was also considered, but it introduced some limitation during testing and was not as straightforward for the event-driven workflow I needed.

## OpenSearch instead of Elasticsearch
My initial thought wa to use Elasticsearch and Kibana for anomalay event log storage and visualization. However, some of the functions that would have been useful in this use case were either limited or tied to commercial licensing. 

Since the requirement here was to build a flexible operational visibility layer without adding unnecessary subscription friction, OpenSearch and OpenSearch Dashboards became the preferred choice.

## Why n8n instead of an MCP-style approach
A question naturally comes up here: why use n8n?\
This is beacause the workflow was event-driven, structured, and privacy-sensitrive.

This system was handling internal network metadata, protected resources, infrastructure paths, source concentration data, and client-related operational context. Thus, pushing that data into an external AI-driven architecture was not appropriate and running a self-hosted model was also out of scope for the immediate requirement.

n8n offered a practical middle ground:

- event-triggered
- controllable
- easy to extend
- not dependent on external AI processing for decision flow

## IP enrichment sources

For lightweight reputation checks and context enrichment, the integration used:
- **AbuseIPDB**
- **GreyNoise**
- **IPinfo**

These were used to provide lightweight enrichment rather than deep forensic intelligence. Even basic reputation and ASN visibility added enough value to improve the analyst’s initial judgment.
> For this implementation and testing stage, free usage tiers were sufficient.


## Cloudflare for access control and exposure management
The worflow platform had to reamin reachable while still being protected, thus, the setup used:
- **Cloudflare Tunnel**
- **Cloudflare Access policies**
- **Cloudflare WAF rules**

This allowed the workflow engine to remain reachable where necessary, while still restricting access to the n8n portal and isolating webhook paths appropriately.

<br>

# Final Architecture in Practice
The system was built in the following stages:

1. **Set up Discord server, channel, and webhook**
2. **Configure GenieATM webhook notifications**
3. **Deploy Cloudflare Tunnel and access controls**
4. **Deploy n8n in a containerized environment**
5. **Create the Discord bot**
6. **Build the n8n workflow**
7. **Deploy OpenSearch and OpenSearch Dashboards**
8. **Generate a final correlated anomaly report**

That final summary became the analyst-facing output instead of reading a raw event and then manually pivoting across multiple systems, the analyst could now see the key context in one place.

<br>

## Setting Up Discord Server, Channel, and Webhook
The first step was straightforward.

Create a Discord server, create a dedicated channel, and configure a webhook on that channel. Since the webhook is bound per channel, it also gives a clean way to separate event sources and reporting outputs later.

![](/assets/img/images/ddos/n8n/discord.png){: width="1200" height="800" .shadow .rounded-3}


On the GenieATM side, the webhook can then be configured through:

**Preferences → Notification → Webhook → Create New**

![](/assets/img/images/ddos/n8n/genieatm-webhook.png){: width="1200" height="800" .shadow .rounded-3}

Once the Discord webhook URL is copied into GenieATM, anomaly events can start flowing into the selected channel.

<br>

## Cloudflare Tunnel and Access Policies
Create the Cloudflare tunnel and publish the application.

![](/assets/img/images/ddos/n8n/cloudflare.png){: width="1200" height="800" .shadow .rounded-3}


[Create a tunnel (dashboard)](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)

[Cloudflare - Onboard a domain](https://developers.cloudflare.com/fundamentals/manage-domains/add-site/)

>[!Note]
Before publishing the application through Cloudflare Tunnel, the domain needs to be onboarded into Cloudflare.



Two subdomains were used in this design.

- `n8n.example.com` **for the n8n web interface**
- `webhook.example.com` **for inbound webhook communication.**

While publshing the application with the new newly created sub-domain for later integration with n8n docker coantiner running on `http://localhost:5678`

![](/assets/img/images/ddos/n8n/cloudflare00.png){: width="1200" height="800" .shadow .rounded-3}

This separation is important.

If only the main n8n interface is published and protected, then interactive access works, but webhook communication may break under strict access enforcement. On the other hand, if the entire application is broadly exposed for webhook compatibility, the n8n login portal becomes unnecessarily reachable from the internet.

![](/assets/img/images/ddos/n8n/cloudflare01.png){: width="1200" height="800" .shadow .rounded-3}

[More Info: Cloudflare Application - Access Control](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)

The cleaner design was:
- protect the **n8n editor portal** with **Cloudflare Access**
- expose a separate webhook FQDN
- restrict that second FQDN with WAF rules
- only allow requests where the URI begins with paths such as:
  - /webhook
  - /api/v1

![](/assets/img/images/ddos/n8n/cloudflare02.png){: width="1200" height="800" .shadow .rounded-3}

This kept the editor protected while still allowing machine-to-machine webhook flow.

![](/assets/img/images/ddos/n8n/cloudflare03.png){: width="1200" height="800" .shadow .rounded-3}

<br>

## Deploying n8n on the VM
The n8n workflow engine was deployed as a Docker container on a Linux VM `RHEL 9.x`.

A dedicated path was created first for persistent storage, followed by permission adjustment for the container runtime user. 
```
sudo mkdir -p /home/<user>/n8n
sudo chown -R 1000:1000 /home/<user>/n8n
sudo chmod 750 /home/<user>/n8n
ls -la /home/<user>/n8n
```
Random secrets were generated for encryption and token management `openssl rand -hex 32`.


Below is a sanitized example of the container run pattern:
```
sudo docker run -d \
  --name ddos-workflow \
  --restart unless-stopped \
  -p 127.0.0.1:5678:5678 \
  -e GENERIC_TIMEZONE="Asia/Kathmandu" \
  -e TZ="Asia/Kathmandu" \
  -e N8N_HOST="n8n.example.com" \
  -e N8N_PROTOCOL="https" \
  -e N8N_PORT=5678 \
  -e N8N_EDITOR_BASE_URL="https://n8n.example.com" \
  -e WEBHOOK_URL="https://webhook.example.com" \
  -e N8N_PROXY_HOPS=1 \
  -e N8N_ENCRYPTION_KEY="<generated-secret>" \
  -e N8N_USER_MANAGEMENT_JWT_SECRET="<generated-secret>" \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_BLOCK_ENV_ACCESS_IN_NODE=true \
  -e N8N_RUNNERS_ENABLED=true \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER="<admin-user>" \
  -e N8N_BASIC_AUTH_PASSWORD="<strong-password>" \
  -e N8N_PUBLIC_API_DISABLED=false \
  -e N8N_API_DISABLED_BASIC_AUTH=true \
  -e N8N_PAYLOAD_SIZE_MAX=64 \
  -e EXECUTIONS_DATA_PRUNE=true \
  -e EXECUTIONS_DATA_MAX_AGE=168 \
  -e EXECUTIONS_DATA_SAVE_ON_SUCCESS=none \
  -e EXECUTIONS_TIMEOUT=3600 \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
>[!Note] 
For public sharing, all real usernames, passwords, FQDNs, tokens, internal paths, and infrastructure-specific values should be masked or replaced with placeholders.
<br>

![](/assets/img/images/ddos/n8n/n8n.png){: width="1200" height="800" .shadow .rounded-3}

<br>

## Creating the Discord Bot

To allow event-driven automation from Discord messages, a bot was created through the Discord Developer Portal.

The process was simple:
- create the application
- enable Bot under OAuth2 scopes
- assign message-related permissions
- generate the install URL
- authorize it into the owned Discord server


![](/assets/img/images/ddos/n8n/discord00.png){: width="1200" height="800" .shadow .rounded-3}
![](/assets/img/images/ddos/n8n/discord01.png){: width="1200" height="800" .shadow .rounded-3}

Once the URL are generated and the copied OAuth2 URL is opened on another browser tabs, with authorization prompt request to the suers, then the bot appears online in the owned Discord server and can participate in the workflow trigger process.

<br>

## Deploying OpenSearch and OpenSearch Dashboards

The backend storage and visibility layer was deployed on RHEL 9.x.

This article is not meant to fully document the OpenSearch installation process, so I will keep this section brief and focused on the points that mattered operationally.


### A few important notes from the deployment
1. Update the operating system first.
2. On RHEL 9+, SHA-1 signature validation becomes an issue for some package flows. In this setup, gpgcheck=0 was used specifically to bypass that installation constraint in the lab/controlled deployment context.
3. Disable swap for OpenSearch stability and performance expectations.
4. Set `vm.max_map_count=262144`, otherwise OpenSearch may fail to start properly.
5. Add the OpenSearch 2.x repository.
6. Be aware that from **OpenSearch 2.12** onward, a custom admin password must be supplied during installation of the demo configuration.
7. Use:
`sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD='<strong-password>' dnf install opensearch -y`
> This avoids the silent post-install script issue that can otherwise occur.
8. Adjust `/etc/opensearch/opensearch.yml`
9. Tune heap in `/etc/opensearch/jvm.options` according to available memory
10. Run the demo security installer if using the bundled security setup
11. Define users, roles, and role mappings
12. Install OpenSearch Dashboards
13. Configure `/etc/opensearch-dashboards/opensearch_dashboards.yml`
14. Create index templates, ingest pipelines, and lifecycle policies before pushing production-style sample data.

> That backend then became the long-term storage and analysis layer for anomaly history, event metadata, source trends, and later dashboarding.

<br>

![](/assets/img/images/ddos/n8n/opensearch.png){: width="1200" height="800" .shadow .rounded-3}

<br>

## n8n Workflow Overview

By default, n8n does not provide native Discord message trigger behavior for this exact use case, so the community node `n8n-nodes-discord-trigger` was installed. You can install by selecting the **Settings** options in the n8n, and select the "Community Nodes", then install by browsing to the "n8n-nodes-discord-trigger" and setup the credential to work.

![](/assets/img/images/ddos/n8n/n8n00.png){: width="1200" height="800" .shadow .rounded-3}
![](/assets/img/images/ddos/n8n/n8n01.png){: width="1200" height="800" .shadow .rounded-3}

Once that was available, the workflow operated as follows:

1. A new Discord message is received in the selected channel used by GenieATM webhook notifications.
2. The workflow first checks whether the message pattern contains an expected event structure.
3. It validates that the incoming body is proper JSON and matches the expected anomaly format.
4. The workflow then checks the event status. In this implementation, the workflow continues only when the anomaly reaches a meaningful state for correlation, such as a recovered or finalized event state.
5. The anomaly ID is extracted from the incoming message.
6. An authenticated HTTP request is made to the GenieATM API to fetch the full event details.
A sanitized example of the API pattern would look like:
`https://genieatm-controller.example.local/api/anomalyevent/application/<anomaly-id>`
7. The JSON response is parsed.
8. One branch sends the structured event into OpenSearch.
9. Another branch extracts the top source IPs and traffic characteristics for enrichment and report generation.
10. If source IPs exist, they are checked against AbuseIPDB, GreyNoise, and IPinfo.
11. All outputs are merged and normalized into a final analyst-facing summary.
12. The report is posted back to a dedicated Discord reporting channel using webhook delivery.

<br>

![](/assets/img/images/ddos/n8n/n8n02.png){: width="1200" height="800" .shadow .rounded-3}

>[!Note]
One important operational note from this workflow: if MFA is enabled on the GenieATM local user being used for the API query, the automated API call may fail depending on the platform configuration. In this case, a dedicated integration user without MFA enforcement may be required, depending on product behavior and policy constraints.

<br>

## Sanitized Example of Event Data for Public Sharing
```
{
  "response": {
    "status": "success",
    "result": {
      "output_file": "/tmp/sample_anomaly_event.json",
      "data": [
        {
          "event": {
            "id": "EVT-10021",
            "ip": "198.51.100.10",
            "status": "Burst",
            "datetime": {
              "start_time": "2026-04-15 10:43:48",
              "end_time": "2026-04-15 10:45:00",
              "duration": 72
            },
            "severity": {
              "type": "yellow",
              "watermark": "over",
              "threshold_value": "10000",
              "detected_value": "58000",
              "max_value": "305000"
            },
            "attack": {
              "type": "DDOS",
              "name": ["UDP Flooding"],
              "counter": "bps"
            },
            "direction": "To Protected Zone",
            "resource": {
              "name": ["PUBLIC-WEB-01"],
              "ip": "198.51.100.10"
            }
          },
          "traffic_characteristics": [
            {
              "name": "Source IP",
              "data_field": ["IP Address(Source)", "bps", "pps"],
              "data": [
                ["203.0.113.10", 40666.66, 16.66],
                ["203.0.113.25", 40400.00, 16.66],
                ["203.0.113.40", 37200.00, 16.66]
              ]
            },
            {
              "name": "Destination Protocol Port",
              "data_field": ["Protocol Port(Destination)", "bps", "pps"],
              "data": [
                ["TCP(6):443", 304800, 416.66]
              ]
            },
            {
              "name": "TCP Flag",
              "data_field": ["TCP Flag", "bps", "pps"],
              "data": [
                ["-A----(16)", 111733.33, 266.66],
                ["----S-(2)", 8000, 16.66]
              ]
            }
          ],
          "network_elements": [
            {
              "router": {
                "id": 201,
                "name": "EDGE-RTR-01",
                "bps": 77066.66,
                "pps": 183.33
              },
              "interface": [
                {
                  "input_interface": {
                    "id": 301,
                    "name": "UPLINK-A"
                  },
                  "output_interface": {
                    "id": 302,
                    "name": "CORE-LINK-A"
                  },
                  "bps": 77066.66,
                  "pps": 183.33
                }
              ]
            }
          ]
        }
      ]
    }
  }
}
```

<br>

## What the Final Correlated Report Achieved

The final report was designed to be more than an alert relay.

It combined:
- anomaly metadata from GenieATM
- top source IP details
- burst volume indicators
- ASN and prefix distribution
- source reputation lookups
- duration and event context
- baseline-relevant metrics for analyst review

That meant the analyst no longer had to jump across multiple systems just to answer a basic question:

The enhancement stack shortened that decision cycle.\
And in DDoS operations, that matters.

<br>


![anomaly event decision - likely false positve](/assets/img/images/ddos/n8n/discord02.png){: width="1200" height="800" .shadow .rounded-3}

![anomaly event decision - likely false positve](/assets/img/images/ddos/n8n/discord04.png){: width="1200" height="800" .shadow .rounded-3}

![anomaly event decision - likely attack](/assets/img/images/ddos/n8n/discord03.png){: width="1200" height="800" .shadow .rounded-3}

This was the real value of the added stack.

- Not another dashboard for the sake of dashboards.
- Not another integration just to say there is automation.

But an operational layer that reduced friction between detection and decision-making.


# Closing Thoughts

One of the mistakes we often make in security architecture is assuming that deploying the core appliance completes the solution.

**However, it does not.**

Especially in DDoS operations, the problem is not only detection accuracy. It is also event usability, analyst fatigue, false positive handling, context correlation, and the speed at which the operations team can move from notification to conclusion.

In this case, the original DDoS platform was necessary, but not sufficient on its own for operational ease. The additional stack around it, built with Discord, n8n, OpenSearch, external IP enrichment, and controlled exposure through Cloudflare, helped bridge the gap between raw anomaly detection and usable security operations.

In the end, was the real enhancement of the existing DDoS appliance stack.
