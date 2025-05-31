---
title: Lightweight ELK Deployment on K3s for F5 Security Logs
description: >-
  This post walks through deploying the ELK stack on a lightweight, single-node K3s Kubernetes cluster to collect, parse, and store F5 BIG-IP security logs.
author: nishalrai
date: 2025-05-31 20:55:00 +0800
categories: [Kubernetes, ELK]
tags: [elk deployment]
---

F5 Networks' BIG-IP systems are widely used for application delivery and security, but their local logging capabilities are constrained. With a storage limit of **either 3 million requests or 2 GB**, logs are archived once either threshold is reached, retaining only the most recent entries. Each daemon and feature (e.g., security, bot, or DoS event logs) shares this quota, making local storage insufficient for high-traffic production environments. 
To overcome this issue, I planned to implement an external logging solution using the ELK Stack (Elasticsearch, Logstash, Kibana) on a single-node k3s cluster, leveraging F5’s High-Speed Logging (HSL) for scalable log ingestion. This setup provides robust storage, parsing, and rich visualization capabilities, with the flexibility to scale as needed.

Initially, I planned to deploy the ELK Stack on a full Kubernetes cluster in a production environment. However, due to resource constraints and shifting objectives, I opted for a lightweight k3s deployment on a single node. This blog details the deployment process, configurations, and considerations for integrating F5 HSL with the ELK Stack, ensuring efficient log management and visualization.

The complete set of manifest files is available in my GitHub repository for reference.<br>
[F5 Security Logs ELK (k3s/k8s)](https://github.com/nishalrai/f5-security-logs-elk-k3s-k8s.git)

<br>

# Deployment Overivew
The ELK Stack deployment on k3s involves the following components and steps:
- **Hardware**: A RHEL 9.5 server with 8 GB RAM, 100 GB storage, and 6 vCPUs.
- **k3s Installation**: Lightweight Kubernetes distribution for single-node deployment.
- **Permissions Setup**: Configuring user access to k3s.
- **Elasticsearch Storage**: Custom directory for indices and persistent volume configuration.
- **Elasticsearch Deployment**: Core storage for logs with resource limits and custom JVM settings.
- **Kibana Deployment**: Web UI for log visualization.
- **Logstash Deployment**: Log parsing and ingestion pipeline with F5 HSL integration.
- **F5 HSL Configuration**: Streaming logs to Logstash.
- **Testing and Visualization**: Validating Logstash functionality and visualizing logs in Kibana

<br>

## Server Setup
The deployment uses a RHEL 9.5 server. While other Linux distributions can be used, minor configuration adjustments may be necessary. Verify the OS with:
`cat /etc/os-release`


To simplify the setup, disable the firewall (ensure this aligns with your security policies):
```
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

Update the system and install k3s using the official script:
```
sudo dnf update -y
curl -sfL https://get.k3s.io | sh -
```

For detailed k3s installation instructions, refer to the official k3s documentation.
[Quickstart k3s](https://docs.k3s.io/quick-start)

## Configuring k3s User Permissions
To manage the k3s cluster, configure user access to the k3s configuration file:
```
sudo ls -l /etc/rancher/k3s/k3s.yaml
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

Create a k3s group and adjust permissions:
```
sudo groupadd k3s
sudo chown root:k3s /etc/rancher/k3s/k3s.yaml
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
sudo usermod -aG k3s $USER
newgrp k3s
```

> This ensures non-root users can interact with the k3s cluster securely.

<br>

## Preparing Elasticsearch Storage
Elasticsearch is a distributed search engine built on **Lucene**, designed for horizontal scalability and capable of handling both structured and unstructured data. It processes JSON requests and returns JSON responses, making it ideal for log storage and analytics. Each Elasticsearch shard is an inverted index, optimized for fast search and aggregation, often outperforming solutions like Hadoop or Spark for log analysis.


To store Elasticsearch indices, create a custom directory on the k3s node:
```
sudo mkdir -p /elasticsearch-data
sudo chown -R 1000:1000 /elasticsearch-data
sudo ls -ld /elasticsearch-data
```

> The 1000:1000 ownership aligns with the default UID/GID of the Elasticsearch container, ensuring proper access.


# Deploying Elasticsearch

Prerequisites
- A running k3s cluster.
- kubectl configured to interact with the cluster.
- Lens for cluster visualization (optional but recommended).
- Elasticsearch version compatibility with Kibana and Logstash (use version 8.15.0 for consistency).

## Installing the Elastic Operator
Install the [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-yaml-manifest-quickstart) operator to manage Elasticsearch, Kibana, and Logstash resources:
```
kubectl create -f https://download.elastic.co/downloads/eck/3.0.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/3.0.0/operator.yaml
```

Reference: [Deploy an orchestrator](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/deploy-an-orchestrator)

<br>

## Persistent Volume and Claim
Create a [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PV) and persistent volume claim (PVC) to store Elasticsearch data. Below is an example elasticsearch-pv=pvc.yaml manifest:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-local-pv
spec:
  capacity:
    storage: 40Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path           # Match your StorageClass (K3s default local storage)
  local:
    path: /elasticsearch-data             # Local directory on the node to store data
  nodeAffinity:                           # Bind PV to a specific node by hostname
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - appsec-k3s               # Replace with your actual node hostname (`kubectl get nodes -o wide`)
  persistentVolumeReclaimPolicy: Retain  # Retain data even if PVC is deleted
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path            # Must match PV’s storageClassName
  resources:
    requests:
      storage: 40Gi
  volumeName: elasticsearch-local-pv     # Explicitly bind to the PV above
```

Then apply the manifest file:
`kubectl apply -f elasticsearch-pv-pvc.yaml`

After create an elasticsearch pod, with the below `elasticsearch.yaml` manifest file provided below:
```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 8.18.1
  nodeSets:
    - name: default
      count: 1
      config:
        node.store.allow_mmap: false
        network.host: "0.0.0.0"
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 2Gi
                  cpu: "2"
                limits:
                  memory: 4Gi
                  cpu: "3"
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms2g -Xmx2g"
              volumeMounts:
                - name: elasticsearch-data
                  mountPath: /usr/share/elasticsearch/data
          # DO NOT manually define the readinessProbe
          # DO NOT manually mount the elasticsearch-es-elastic-user secret
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            storageClassName: local-path
            resources:
              requests:
                storage: 40Gi

```

Apply the manifest:<br>
`kubectl apply -f elasticsearch.yaml`


And to monitor the cluster health:<br>
`kubectl get elasticsearch`

<br>

## Accessing Elasticsearch
A ClusterIP service (quickstart-es-http) is automatically created. Retrieve the default elastic user password:<br>
```
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```

Test the Elasticsearch endpoint from within the cluster:<br>
```
kubectl exec -it quickstart-es-default-0 -- curl -k -u "elastic:$PASSWORD" https://quickstart-es-http:9200
```

For local access, port-forward the service:
```
kubectl port-forward service/quickstart-es-http 9200
curl -u "elastic:$PASSWORD" -k https://localhost:9200
```

> Accessing https://localhost:9200 in a browser will trigger a certificate warning due to the self-signed certificate. For production, configure valid certificate.

<br>

# Deploying Kibana
Kibana provides a web-based interface for searching and visualizing Elasticsearch data, offering charts, graphs, and dashboards for log analysis. Deploy Kibana with the following kibana.yaml:

```
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: default
spec:
  version: 8.18.1
  count: 1
  elasticsearchRef:
    name: elasticsearch
    serviceName: elasticsearch-es-http
  http:
    service:
      spec:
        type: NodePort
        ports:
        - port: 5601
          targetPort: 5601
          nodePort: 30080
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: "0.5"
          limits:
            memory: 2Gi
            cpu: "1"
```

Apply the manifest:<br>
`kubectl apply -f kibana.yaml`

Monitor Kibana status:<br>
`kubectl get kibana`


Access Kibana at http://<k3s-node-ip>:30080. Log in using the elastic user and the password retrieved earlier.<br>
```
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```


Once Elastcisearch and Kibana setup is complete, create an Index Lifecycle Management (ILM) policy, where one can use either:
- curl (direct API calls to Elasticsearch)
- Kibana Dev Tools (Console tab)

Below is a step-by-step guide using both methods, so you can choose the one that best suits your workflow.

<br>

## Goal
Set up an ILM policy to:

- Hot phase: Retain active data for 7 days
- Delete phase: Automatically delete indices older than 15 days

Apply this policy to the `logstash-*` index pattern.


**Define the ILM Policy**
- Method A: Using `curl` command.
```
curl -k -u "elastic:h34z3Ti8BA2M1Bh47Hf8pq8A" -X PUT "https://localhost:30920/_ilm/policy/waf-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_age": "7d",
              "max_size": "10kb"
            }
          }
        },
        "delete": {
          "min_age": "15d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```
> Ensure https://localhost:9200 is reachable and trusted. Use your own IP or domain if applicable.


- Method B: Using Kibana Dev Tools (preferred for visual confirmation)
Open Kibana → Dev Tools, then paste:
```
PUT _ilm/policy/waf-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "10Gb"
          }
        }
      },
      "delete": {
        "min_age": "15d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

```
PUT _index_template/waf-logs-template
{
  "index_patterns": ["waf-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "waf-policy",
      "index.lifecycle.rollover_alias": "waf-logs"
    }
  }
}

PUT waf-logs-000001
{
  "aliases": {
    "waf-logs": {
      "is_write_index": true
    }
  }
}
```


# Deploying Logstash
Logstash processes and forwards logs from F5 HSL to Elasticsearch. Since F5 does not support external agents, we rely on its HSL capability to stream logs directly to Logstash. Create directories for Logstash pipelines and certificates:

```
sudo mkdir -p /etc/logstash/{pipelines,certificate,geoip}
sudo chown -R 1000:1000 /etc/logstash/pipelines
sudo chown -R 1000:1000 /etc/logstash/certificate
sudo chown -R 1000:1000 /etc/logstash/geoip
```

<br>

### Store the Elasticsearch CA certificate:
`kubectl get secret quickstart-es-http-certs-public -o jsonpath="{.data.ca\.crt}" | base64 -d > /etc/logstash/certificate/ca.crt`



Create a Logstash pipeline configuration in `/etc/logstash/piepleines` (f5-asm.conf):

```
input {
    syslog {
      port => 5244
    }
   }
   filter {
    grok {
      match => {
        "message" => [
          "attack_type=\"%{DATA:attack_type}\"",
          "management_ip_address=\"%{IP:management_ip_address}\"",
          ",blocking_exception_reason=\"%{DATA:blocking_exception_reason}\"",
          ",date_time=\"%{DATA:date_time}\"",
          ",dest_port=\"%{DATA:dest_port}\"",
          ",ip_client=\"%{DATA:ip_client}\"",
          ",is_truncated=\"%{DATA:is_truncated}\"",
          ",method=\"%{DATA:method}\"",
          ",policy_name=\"%{DATA:policy_name}\"",
          ",protocol=\"%{DATA:protocol}\"",
          ",request_status=\"%{DATA:request_status}\"",
          ",response_code=\"%{DATA:response_code}\"",
          ",severity=\"%{DATA:severity}\"",
          ",sig_cves=\"%{DATA:sig_cves}\"",
          ",sig_ids=\"%{DATA:sig_ids}\"",
          ",sig_names=\"%{DATA:sig_names}\"",
          ",sig_set_names=\"%{DATA:sig_set_names}\"",
          ",src_port=\"%{DATA:src_port}\"",
          ",sub_violations=\"%{DATA:sub_violations}\"",
          ",support_id=\"%{DATA:support_id}\"",
          "unit_hostname=\"%{DATA:unit_hostname}\"",
          ",uri=\"%{DATA:uri}\"",
          ",violation_rating=\"%{DATA:violation_rating}\"",
          ",vs_name=\"%{DATA:vs_name}\"",
          ",x_forwarded_for_header_value=\"%{DATA:x_forwarded_for_header_value}\"",
          ",outcome=\"%{DATA:outcome}\"",
          ",outcome_reason=\"%{DATA:outcome_reason}\"",
          ",violations=\"%{DATA:violations}\"",
          ",violation_details=\"%{DATA:violation_details}\"",
          ",request=\"%{DATA:request}\""
        ]
      }
      break_on_match => false
    }
    mutate {
      remove_field => ["message", "event"]
      split => { "attack_type" => "," }
      split => { "sig_ids" => "," }
      split => { "sig_names" => "," }
      split => { "sig_cves" => "," }
      split => { "staged_sig_ids" => "," }
      split => { "staged_sig_names" => "," }
      split => { "staged_sig_cves" => "," }
      split => { "sig_set_names" => "," }
      split => { "threat_campaign_names" => "," }
      split => { "staged_threat_campaign_names" => "," }
      split => { "violations" => "," }
      split => { "sub_violations" => "," }
    }
    if [x_forwarded_for_header_value] != "N/A" {
      mutate { add_field => { "source_host" => "%{x_forwarded_for_header_value}"}}
    } else {
      mutate { add_field => { "source_host" => "%{ip_client}"}}
    }
    geoip {
      source => "source_host"
      target => "geoip"
    }
   }

output {
  elasticsearch {
    hosts => ["https://elasticsearch-es-http:9200"]
    user => "elastic"
    password => "<elasticsearch-password"
    #ssl_certificate_authorities => ["/etc/logstash/certificate/ca.crt"]
    ssl_verification_mode => "none"
    index => "waf-logs-write"
    ilm_enabled => true
    ilm_policy => "waf-policy"
    ilm_rollover_alias => "waf-logs"
    ecs_compatibility => "disabled"
  }
}
```

Then deploy, logstash with the following `logstash.yaml` manifest file.
```
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: logstash
  namespace: default
spec:
  count: 1
  version: 8.18.1
  elasticsearchRefs:
  - name: elasticsearch
    serviceName: elasticsearch-es-http
    clusterName: elasticsearch
  config:
    xpack.monitoring.enabled: false  # Disable X-Pack monitoring
  #  log.level: debug  # Keep debug logging
    #xpack.monitoring.enabled: true
    #xpack.monitoring.elasticsearch.hosts: ["https://elasticsearch-es-http:9200"]
    # Removed ssl.certificate_authority since we're disabling cert verification
  pipelines:
  - pipeline.id: main
    path.config: /usr/share/logstash/pipeline/asm.conf
  podTemplate:
    spec:
      containers:
      - name: logstash
        env:
      #  - name: LOG_LEVEL
      #    value: "debug"
        - name: ELASTICSEARCH_HOSTS
          value: "https://elasticsearch-es-http:9200"
        - name: ELASTIC_USERNAME
          value: "elastic"
        - name: ELASTIC_PASSWORD
          value: "<elasticsearch-password>"
        resources:
          requests:
            memory: 1Gi
            cpu: "0.5"
          limits:
            memory: 2Gi
            cpu: "1"
        volumeMounts:
        - name: pipeline
          mountPath: /usr/share/logstash/pipeline
        - name: geoip
          mountPath: /usr/share/logstash/geoip
        - name: ca-cert
          mountPath: /etc/logstash/certificate
        # Removed elasticsearch-ca volumeMount since cert verification is disabled
      volumes:
      - name: pipeline
        hostPath:
          path: /etc/logstash/pipelines
          type: Directory
      - name: geoip
        hostPath:
          path: /etc/logstash/geoip
          type: Directory
      - name: api-credentials
        secret:
          secretName: logstash-api-credentials
      - name: ca-cert
        hostPath:
          path: /etc/logstash/certificate

      # Removed elasticsearch-ca volume since cert verification is disabled
  services:
  - name: syslog
    service:
      spec:
        type: NodePort
        ports:
        - port: 5244
          name: syslog
          protocol: TCP
          targetPort: 5244
          nodePort: 30044  # Adjust if needed to avoid conflicts
```

**Apply the manifest**:<br>
`kubectl apply -f logstash.yaml`


**Check Logstash status**:<br>
`kubectl get logstash`

<br>

# Configuring F5 HSL
Configure F5 BIG-IP to send HSL logs to the Logstash service at <k3s-node-ip>:30514. Refer to the F5 HSL documentation for specific configuration steps, which typically involve setting up a remote logging pool and destination. [HSL Configuration in F5 BIG-IP](https://nishalrai.com.np/posts/remote-hsl-f5bigip/)

# Testing Logstash
Verify Logstash connectivity to Elasticsearch:
`kubectl exec -it logstash-ls-0 -- /bin/sh -c "curl --cacert /etc/logstash/certificate/ca.crt -u elastic:$PASSWORD https://quickstart-es-http:9200"`


Check Logstash logs for errors:
`kubectl logs -n default logstash-ls-0 -f`


## Visualizing Logs in Kibana
Access Kibana at http://<k3s-node-ip>:30080. Navigate to the Discover tab, create an index pattern for f5-logs-*, and explore the logs. Use Kibana’s visualization tools to create dashboards for security events, bot activity, or DoS incidents.

# Additional Tips
- **Credential extraction**
`kubectl get secret quickstart-es-elastic-user -o jsonpath='{.data.elastic}' | base64 -d`

- **The Pod debugging**
```
kubectl describe pod <pod-name>
kubectl logs -n default <pod-name> -f
```

- **Certificate Validation**
`kubectl get secret quickstart-es-http-certs-public -o jsonpath="{.data.tls\.crt}" | base64 -d | openssl x509 -noout -text`

<br>

# Conclusion
Deploying the ELK Stack on a single-node k3s cluster provides a scalable, lightweight solution for managing F5 HSL logs. Elasticsearch stores and indexes logs, Logstash parses and forwards them, and Kibana offers powerful visualization capabilities. The future enhancements could include enabling xpack security for encrypted communication, optimizing resource usage with ConfigMaps, and scaling to a multi-node cluster for high availability.

This setup ensures that F5’s limited local logging is no longer a bottleneck, enabling robust log analysis and monitoring for production environments.
