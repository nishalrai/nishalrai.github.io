---
title: Integration of AS3 and TS extension of F5 BIG-IP with Azure Sentinel
author: nishalrai
date: 2023-06-25 20:55:00 +0800
categories: [F5 Networks, Azure]
tags: [integration, azure]
---

One can leverage the usage of Azure Sentinel to collect and display the data using the Telemetry streaming extension on the F5 BIG-IP device. Azure Sentinel is able to collect the logs from the F5 BIG-IP via Telemetry Streaming regardless of its deployed location – F5 BIG-IP does not need to be on Azure to fetch those logs.

A little background about the F5 BIG-IP Application Services 3 and Telemetry Streaming.

BIG-IP AS3, the F5 BIG-IP Application Services 3 is an extension that uses a declarative model – JSON declaration instead of a set of imperative commands to create resources on a BIG-IP system. The system’s API endpoint – `(https://<BIG-IP>/mgmt/shared/appsvcs/declare)`

Telemetry streaming (TS) is an iControl LX extension delivered as a TMOS-independent RPM file with the ability to declaratively aggregate, normalize and forward statistics and events from the BIG-IP to a consumer application by posting a single TS JSON declaration to TS’s declarative REST API endpoint. The Telemetry Streaming’s API endpoint – `(https://<BIG-IP>>/mgmt/shared/telemetry/declare)`

<br>

# Setup of TS and AS3 on F5 BIG-IP to integrate with Azure Sentinel
The whole configuration is summarized in the following points:

Verify the required modules are enabled
Install the TS and AS3 extension on the F5 BIG-IP device
Create the required configuration object on F5 BIG-IP
Configure the Data connector of Azure with F5 BIG-IP device
Verify all the required data types are available on Azure Sentinel
The configuration involves both TS and AS3 extensions for different purposes – TS for establishing a connection with Azure Sentinel Data connector and AS3 for creating configuration object in the F5 BIG-IP like Virtual Server, Request Logging profile, log profile, iRule, and others.

On the F5 BIG-IP device, the required modules to be enabled are ASM, AVR and iRulesLX.

![](/assets/img/images/f5/f5sentinel.png){: width="600" height="400"}
> NOTE:
The version on which the configuration is carried out is F5 BIG-IP v16.3.3 and v17.0.1

![](/assets/img/images/f5/f5sentinel2.png){: width="600" height="400"}

## Install the TS and AS3 extension on the F5 BIG-IP device
You need to download TS and AS3 extension and upload on your F5 BIG-IP device.

```
Download link of Telemetry Streaming:
https://github.com/F5Networks/f5-appsvcs-extension/releases

Download link of Application Streaming 3 extension:
https://github.com/F5Networks/f5-telemetry-streaming/releases
```

To upload on F5 BIG-IP device:
- Go to Main Dashboard > iApps > Package Management LX
- Click on Import and select the file

*f5-appsvcs v3.45.0 and f5-telemetry v1.33.0 is being used (the latest version available).*

## Create the required configuration object on F5 BIG-IP
AS3 and TS extension is used to configure F5 BIG-IP with the necessary resources with a single JSON declaration. In this configuration, Postman is used to configure event listeners for the various deployed modules.

The JSON declaration to configure to the following configuration object – Virtual Server, Pool, Node, iRule, Request Logging and Request log.

```
{
	"class": "ADC",
	"schemaVersion": "3.45.0",
	"remark": "Example depicting creation of BIG-IP module log profiles",
	"Common": {
		"class": "Tenant",
		"Shared": {
			"class": "Application",
			"template": "shared",
			"telemetry_local_rule": {
				"remark": "Only required when TS is a local listener",
				"class": "iRule",
				"iRule": "when CLIENT_ACCEPTED {\n  node 127.0.0.1 6514\n}"
			},
			"telemetry_local": {
				"remark": "Only required when TS is a local listener",
				"class": "Service_TCP",
				"virtualAddresses": [
					"255.255.255.254"
				],
				"virtualPort": 6514,
				"iRules": [
					"telemetry_local_rule"
				]
			},
			"telemetry": {
				"class": "Pool",
				"members": [{
					"enable": true,
					"serverAddresses": [
						"255.255.255.254"
					],
					"servicePort": 6514
				}],
				"monitors": [{
					"bigip": "/Common/tcp"
				}]
			},
			"telemetry_hsl": {
				"class": "Log_Destination",
				"type": "remote-high-speed-log",
				"protocol": "tcp",
				"pool": {
					"use": "telemetry"
				}
			},
			"telemetry_formatted": {
				"class": "Log_Destination",
				"type": "splunk",
				"forwardTo": {
					"use": "telemetry_hsl"
				}
			},
			"telemetry_publisher": {
				"class": "Log_Publisher",
				"destinations": [{
					"use": "telemetry_formatted"
				}]
			},
			"telemetry_traffic_log_profile": {
				"class": "Traffic_Log_Profile",
				"requestSettings": {
					"requestEnabled": true,
					"requestProtocol": "mds-tcp",
					"requestPool": {
						"use": "telemetry"
					},
					"requestTemplate": "event_source=\"request_logging\",hostname=\"$BIGIP_HOSTNAME\",client_ip=\"$CLIENT_IP\",server_ip=\"$SERVER_IP\",http_method=\"$HTTP_METHOD\",http_uri=\"$HTTP_URI\",virtual_name=\"$VIRTUAL_NAME\",event_timestamp=\"$DATE_HTTP\""
				},
                "responseSettings": {
                    "responseEnabled": true,
                    "responseProtocol": "mds-tcp",
                    "responsePool": {
                        "use": "telemetry"
                    },
                    "responseTemplate": "event_source=\"response_logging\",hostname=\"$BIGIP_HOSTNAME\",client_ip=\"$CLIENT_IP\",server_ip=\"$SERVER_IP\",http_method=\"$HTTP_METHOD\",http_uri=\"$HTTP_URI\",virtual_name=\"$VIRTUAL_NAME\",event_timestamp=\"$DATE_HTTP\",http_statcode=\"$HTTP_STATCODE\",http_status=\"$HTTP_STATUS\",response_ms=\"$RESPONSE_MSECS\""
                }
			},
			"telemetry_asm_security_log_profile": {
				"class": "Security_Log_Profile",
				"application": {
					"localStorage": false,
					"remoteStorage": "splunk",
					"servers": [{
						"address": "255.255.255.254",
						"port": "6514"
					}],
					"storageFilter": {
						"requestType": "all"
					}
				}
			}
		}
	}
}
```

## Tips to mitigate configuration issues
Use the visual studio code and add JSON formatter extension to format the JSON code and avoid any indentation error on the code.

On the JSON declaration, be careful with the schemaVersion, the version should match with the install The F5 Application Streaming v3 extension, in my case it’s 3.45.0


Launch the postman, enter the API endpoint:

`https://<BIG-IP>/mgmt/shared/appsvcs/declare`

*Output of the successful deployment:*


### Verify whether the object has been created on F5 BIG-IP
Browse to the F5 BIG-IP dashboard and verify whether all the required objects has been created or not.

Once all the object has been created, you need to execute the following command on the F5 BIG-IP CLI. This seems to be a bug on the TS listener with F5 BIG-IP device.

```
The issue was caused by a new db key which by default prohibits loopback addresses in irules.  

If you have configured a local listener, with an irule such as “when CLIENT_ACCEPTED {\n node 127.0.0.1 6514\n}”  

Then you need to run the following tmsh command.
tmsh modify sys db tmm.tcl.rule.node.allow_loopback_addresses value true  
```

For more info:

[Telemetry Streaming GitHub](https://github.com/F5Networks/f5-telemetry-streaming/issues/238)


## Configure the Data connector of Azure Sentinel with F5 BIG-IP device
Once all the above configuration has been completed, it’s time to integrate F5 BIG-IP device with Azure Sentinel. Telemetry Streaming extension will be used to establish the connection between the F5 BIG-IP device and data connector of Azure sentinel.
The JSON declaration used to establish the connection between the Azure Sentinel – Data Connector and F5 BIG-IP device.

```JSON
{
    "class": "Telemetry",
    "controls": {
        "class": "Controls",
        "logLevel": "info",
        "debug": true
    },
    "My_System": {
        "class": "Telemetry_System",
        "trace": "/var/tmp/telemetry_trace.log",
        "systemPoller": {
            "interval": 60
        }
    },
    "My_Listener": {
        "class": "Telemetry_Listener",
        "port": 6514
    },
    "My_Consumer": {
        "class": "Telemetry_Consumer",
        "type": "Azure_Log_Analytics",
        "workspaceId": "<workspace-id>",
        "passphrase": {
            "cipherText": "<cipher-text>"
        },
        "useManagedIdentity": false,
        "region": "<region>"
    }
}
```

You can find the required credentials of the Azure Sentinel on the workspace of the F5 BIG-IP connector page.

Once you’ve got all the required credentials then you can carry out the configuration.

I will be using Postman to declare the configuration in JSON format on system’s endpoint: `https://<BIG-IP>>/mgmt/shared/telemetry/declare`

Once you’ve got all the required credentials then you can carry out the configuration.

I will be using Postman to declare the configuration in JSON format on system’s endpoint:
https://<BIG-IP>>/mgmt/shared/telemetry/declare
<br>


## Verify all the required data types are available on Azure Sentinel
After all the configuration has been completed, you need to login into the Azure Portal. Browse to the Microsoft Sentinel then select the workspace.

Search for F5 BIG-IP and open the connector page then you can see the data type available.

On the Workspace of the Azure Sentinel, you can browse to the Workbook – F5 BIG-IP ASM, where all the collected logs of ASM (only Application Security logs) are visualized.


This is the visualization of the ASM logs on the Azure Sentinel.


