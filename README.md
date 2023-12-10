## Introduction

If you want to look into the health of your Cribl Stream environment, you are usually pretty set with the default Monitoring functionality that Cribl offers out of the box.

But Cribl also lets you send Internal Metrics/Logs to external monitoring tools, so you can take advantage of the advanced searching, visualization and alerting capabilities of Elasticsearch.

## How It Works

We prepare Elasticsearch before sending the data with the appropriate mappings.

Metrics are saved in [Time Series Data Stream (TSDS)](https://www.elastic.co/guide/en/elasticsearch/reference/current/tsds.html).

Logs are saved in [Data Streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html).

We connect [the Cribl Internal Metrics and Logs sources](https://docs.cribl.io/stream/sources-cribl-internal/#configuring-cribl-internal-logsmetrics-as-a-datasource) to an [Elasticsearch destination](https://docs.cribl.io/stream/destinations-elastic/) using QuickConnect.

We select a custom pipeline as a pre-processing pipeline for logs before sending to Elasticsearch.

We have created some dashboards and alerts for you, from the perspective of a Cribl Support Engineer that we can import into Kibana.

## Prerequisites

- Elasticsearch version 8.7+
- Kibana version 8.8.0+

## Getting started

We provide 2 options to set you up quickly. 

Option 1: You can prepare Elasticsearch and Cribl Stream automatically using a `bootstrap.sh` script. This is ideal for larger environments, to prevent having to configure many worker groups at the same time. 

Option 2: You can quickly configure Elasticsearch and Cribl Stream this manually as well.

For both options you will then have to manually commit and deploy Cribl Stream's changes and import the Dashboards into Kibana.

You can then also choose to collect logs from the Leader separately as an optional step.

## Option 1 - Bootstrap Script

<details><summary>This script will run the manual steps for every Stream worker group, which could save some time.</summary>

Prerequisites for the script are: `Stream, Bash, jq, curl`

Files to run the script are in [the Github repository](https://github.com/criblio/elastic-cribl-monitoring).

| :information_source: The following variables will need to be updated in the `.env` file: |
|----------------------------------------------|

| Variable(s)  | Description |
| ------------- | ------------- |
|`ES_ELASTIC_URL` | The endpoint to your Elasticsearch deployment (e.g. `https://elasticsearch.domain.com:9200`) |
|`ES_KIBANA_URL` | The endpoint to your Kibana instance (e.g. `https://kibana.domain.com:5601`) |
|`ES_CRIBL_URL` | The endpoint to your Cribl Leader node or your Cribl.Cloud URL .  (e.g. `https://cribl.domain.com:9000` or `https://main-happy-margulis-topxery.cribl.cloud`) |
|`CRIBL_USER`, `CRIBL_PASSWORD` | **On-Premise specific options.** Your typical Username and Password combination for Authenticating through the Cribl UI. |
| `CRIBL_CLIENT_ID` , `CRIBL_CLIENT_SECRET` | **Cribl.Cloud specific options.** The Client ID and secret created by [following instructions here](https://docs.cribl.io/stream/api-tutorials/#criblcloud-free-tier). |
| `ES_ELASTIC_USER`, `ES_ELASTIC_PASSWORD` | Elasticsearch username and password to authenticate Elasticsearch API calls with. |
| `ES_ELASTIC_BEARER_TOKEN` | (optional) If you want to use [token-based authentication service](https://www.elastic.co/guide/en/elasticsearch/reference/current/token-authentication-services.html) to authenticate Elasticsearch API calls (limited to service-accounts and token-service) |
| `ES_ELASTIC_PASSWORD` | The password for your Cribl Writer user |
| `ES_CRIBL_WORKERGROUP_NAME` | The worker groups for which to apply the bootstrap script to (e.g. `("default" "defaultHybrid")` for multiple or `("defaultHybrid")` for a single group) |
| `ES_CRIBL_CUSTOM_IDENTIFIER` | (optional) Any value for a custom identifier that you want to add, such as a data centre name. Will be added as a field to the events. |
| `ES_CRIBL_ELASTIC_OUTPUT_ID` (optional) | The ID of the Elasticsearch output that will be created/updated. This id will be the same across all your worker groups. |

| :information_source: The bootstrap script has some debugging options which can be enabled by uncommenting lines at the top of the file. This will create a log file with very verbose information in the same working directory as the script. |
|----------------------------------------------|

#### Run the bootstrap script:

1. Make `bootstrap.sh` executable (`chmod +x bootstrap.sh`)
2. Run: `./bootstrap.sh`
3. Commit and deploy from Cribl to make sure the changes take effect
4. Continue to the Importing Saved Objects section.

| :information_source: Common issues while running the script: |
|----------------------------------------------|
| - Be sure to commit and deploy, for the changes in Stream to take effect! |
| - If the `ES_*_URL` endpoints require a TLS connection while the Certificate Authorities certificates are not in your local trust stores, you will have to adjust the curl commands to use the `-k` flag |

</details>

## Option 2 - Manual Configuration

<details><summary>Here are the manual steps to configure Elasticsearch first and then Cribl Stream or Edge. Keep in mind that the script only works for Stream at the moment.</summary>

#### 1. Elasticsearch Configuration

Please copy & paste and then execute the appropriate commands into your [Developer Console](https://www.elastic.co/guide/en/kibana/current/console-kibana.html) one by one in the order that they are displayed:

| :information_source: Please update the password for the user and/or the permissions of the corresponding user role. |
|----------------------------------------------|

<details><summary>1. Roles</summary>

```
PUT _security/role/cribl_writer
{
  "cluster": ["manage_index_templates", "monitor", "manage_ilm"],
  "indices": [
    {
      "names": [ "metrics-cribl-internal", "logs-cribl-internal" ],
      "privileges": ["write","create","create_index","manage","manage_ilm"]
    }
  ]
}
```

</details>

<details><summary>2. Users</summary>

```
PUT _security/user/cribl_internal
{
  "password" : "cribl-test-password",
  "roles" : [ "cribl_writer"],
  "full_name" : "Internal Cribl User"
}
```

</details>

<details><summary>3. Component Templates</summary>

**Metrics:**

```
PUT _component_template/metrics-cribl-internal
{
  "template": {
    "settings": {
      "index": {
        "look_ahead_time": "10m",
        "codec": "best_compression"
      }
    },
    "mappings": {
      "_data_stream_timestamp": {
        "enabled": true
      },
      "dynamic": "runtime",
      "properties": {
        "custom_id": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "@timestamp": {
          "type": "date"
        },
        "_metric_type": {
          "type": "keyword"
        },
        "host": {
          "properties": {
            "name": {
              "type": "keyword",
              "time_series_dimension": true
            }
          }
        },
        "_metric": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "_value": {
          "type": "double",
          "time_series_metric": "gauge"
        },
        "cribl_wp": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "group": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "input": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "output": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "name": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "pipeline": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "project": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "route": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "source": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "subscription": {
          "type": "keyword",
          "time_series_dimension": true
        },
        "sourcetype": {
          "type": "keyword",
          "time_series_dimension": true
        }
      }
    }
  }
}
```

**Logs:**

```
PUT _component_template/logs-cribl-internal
{
  "template": {
    "mappings": {
      "dynamic": false,
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "BRANCH": {
          "type": "keyword"
        },
        "TIMESTAMP": {
          "type": "date"
        },
        "VERSION": {
          "type": "keyword"
        },
        "_raw": {
          "type": "text"
        },
        "abortCxn": {
          "type": "long"
        },
        "ack": {
          "type": "long"
        },
        "action": {
          "type": "keyword"
        },
        "activeCxn": {
          "type": "long"
        },
        "activeEP": {
          "type": "long"
        },
        "activeId": {
          "type": "keyword"
        },
        "activelyClosing": {
          "type": "long"
        },
        "activityLogSampleRate": {
          "type": "long"
        },
        "actualReloadPeriodMs": {
          "type": "long"
        },
        "addIdToStagePath": {
          "type": "boolean"
        },
        "alive": {
          "type": "keyword"
        },
        "all": {
          "properties": {
            "values": {
              "type": "long"
            }
          }
        },
        "allCollectors": {
          "type": "text"
        },
        "apiKey": {
          "type": "keyword"
        },
        "apiName": {
          "type": "keyword"
        },
        "apiVersion": {
          "type": "long"
        },
        "appId": {
          "type": "keyword"
        },
        "arch": {
          "type": "keyword"
        },
        "args": {
          "type": "text"
        },
        "assumeRoleArn": {
          "type": "keyword"
        },
        "audience": {
          "type": "keyword"
        },
        "authMethod": {
          "type": "keyword"
        },
        "authToken": {
          "type": "keyword"
        },
        "authTokens": {
          "type": "keyword"
        },
        "authType": {
          "type": "keyword"
        },
        "authUrl": {
          "type": "keyword"
        },
        "authenticationTimeout": {
          "type": "long"
        },
        "avoidDuplicates": {
          "type": "boolean"
        },
        "awsAccountId": {
          "type": "keyword"
        },
        "awsApiKey": {
          "type": "keyword"
        },
        "b": {
          "type": "long"
        },
        "baseFileName": {
          "type": "keyword"
        },
        "batchSize": {
          "type": "long"
        },
        "bearerTokenLen": {
          "type": "long"
        },
        "blockedEP": {
          "type": "long"
        },
        "blockedSince": {
          "type": "long"
        },
        "breakerRulesets": {
          "type": "keyword"
        },
        "broker": {
          "type": "keyword"
        },
        "brokers": {
          "type": "text"
        },
        "bucket": {
          "type": "keyword"
        },
        "bufSize": {
          "type": "long"
        },
        "buffered": {
          "type": "long"
        },
        "build": {
          "type": "keyword"
        },
        "cacheCleanupNumEvents": {
          "type": "long"
        },
        "cacheExpirationMs": {
          "type": "long"
        },
        "caller": {
          "properties": {
            "groupId": {
              "type": "keyword"
            },
            "guid": {
              "type": "keyword"
            },
            "inputId": {
              "type": "keyword"
            },
            "workerId": {
              "type": "keyword"
            }
          }
        },
        "capabilities": {
          "properties": {
            "ack": {
              "type": "keyword"
            },
            "compression": {
              "type": "keyword"
            }
          }
        },
        "captureHeaders": {
          "type": "boolean"
        },
        "cfgWorkerCount": {
          "type": "long"
        },
        "channel": {
          "type": "keyword"
        },
        "checkFileModTime": {
          "type": "boolean"
        },
        "checksum": {
          "type": "keyword"
        },
        "cid": {
          "type": "keyword"
        },
        "cleanFields": {
          "type": "boolean"
        },
        "cleanupInterval": {
          "type": "long"
        },
        "client": {
          "type": "keyword"
        },
        "clientId": {
          "type": "keyword"
        },
        "closeCxn": {
          "type": "long"
        },
        "closingFiles": {
          "type": "long"
        },
        "collected": {
          "type": "boolean"
        },
        "collectible": {
          "properties": {
            "__pageNum": {
              "type": "long"
            },
            "collectedEvents": {
              "type": "long"
            },
            "collectorId": {
              "type": "keyword"
            },
            "contentCreated": {
              "type": "date"
            },
            "contentExpiration": {
              "type": "date"
            },
            "contentId": {
              "type": "keyword"
            },
            "contentTime": {
              "type": "float"
            },
            "contentType": {
              "type": "keyword"
            },
            "contentUri": {
              "type": "keyword"
            },
            "discoveredEvents": {
              "type": "long"
            },
            "earliest": {
              "type": "long"
            },
            "fakeDiscover": {
              "type": "boolean"
            },
            "filteredEvents": {
              "type": "long"
            },
            "flushedBuffers": {
              "type": "long"
            },
            "guid": {
              "type": "keyword"
            },
            "host": {
              "type": "keyword"
            },
            "id": {
              "type": "keyword"
            },
            "latest": {
              "type": "long"
            },
            "source": {
              "type": "keyword"
            },
            "taskId": {
              "type": "keyword"
            }
          }
        },
        "collectorId": {
          "type": "keyword"
        },
        "collectorIds": {
          "type": "text"
        },
        "collectors": {
          "properties": {
            "cronSchedule": {
              "type": "keyword"
            },
            "id": {
              "type": "keyword"
            }
          }
        },
        "commitFrequency": {
          "type": "long"
        },
        "compareExpression": {
          "type": "keyword"
        },
        "compress": {
          "type": "keyword"
        },
        "compression": {
          "type": "keyword"
        },
        "concurrency": {
          "type": "long"
        },
        "concurrentJobLimit": {
          "type": "long"
        },
        "concurrentScheduledJobLimit": {
          "type": "long"
        },
        "concurrentSystemJobLimit": {
          "type": "long"
        },
        "confReloadPeriodSec": {
          "type": "long"
        },
        "configHelper": {
          "type": "boolean"
        },
        "configHelperSocketDir": {
          "type": "keyword"
        },
        "conflictingFields": {
          "properties": {
            "time": {
              "type": "long"
            }
          }
        },
        "connected": {
          "type": "boolean"
        },
        "connectionTimeout": {
          "type": "long"
        },
        "connections": {
          "properties": {
            "output": {
              "type": "keyword"
            }
          }
        },
        "consecutiveSkips": {
          "type": "long"
        },
        "consumerOpts": {
          "properties": {
            "groupId": {
              "type": "keyword"
            },
            "heartbeatInterval": {
              "type": "long"
            },
            "maxBytes": {
              "type": "long"
            },
            "maxBytesPerPartition": {
              "type": "long"
            },
            "rebalanceTimeout": {
              "type": "long"
            },
            "sessionTimeout": {
              "type": "long"
            }
          }
        },
        "consumerRunOpts": {
          "properties": {
            "autoCommit": {
              "type": "boolean"
            }
          }
        },
        "consumerRunning": {
          "type": "boolean"
        },
        "containerName": {
          "type": "keyword"
        },
        "content": {
          "type": "text"
        },
        "contentCreated": {
          "type": "date"
        },
        "contentExpiration": {
          "type": "date"
        },
        "contentId": {
          "type": "keyword"
        },
        "contentTime": {
          "type": "float"
        },
        "contentType": {
          "type": "keyword"
        },
        "contentUri": {
          "type": "keyword"
        },
        "context": {
          "type": "keyword"
        },
        "copyStatsKey": {
          "type": "keyword"
        },
        "correlationId": {
          "type": "long"
        },
        "count": {
          "type": "long"
        },
        "cpuPerc": {
          "type": "float"
        },
        "cpuProfile": {
          "type": "boolean"
        },
        "cpus": {
          "type": "long"
        },
        "createContainer": {
          "type": "boolean"
        },
        "createQueue": {
          "type": "boolean"
        },
        "createdAt": {
          "type": "long"
        },
        "criblAPI": {
          "type": "keyword"
        },
        "cribl_pipe": {
          "type": "keyword"
        },
        "current": {
          "type": "long"
        },
        "currentReadables": {
          "type": "long"
        },
        "currentState": {
          "type": "keyword"
        },
        "customContentType": {
          "type": "keyword"
        },
        "customDropWhenNull": {
          "type": "boolean"
        },
        "customEventDelimiter": {
          "type": "keyword"
        },
        "custom_id": {
          "type": "keyword"
        },
        "customSourceExpression": {
          "type": "keyword"
        },
        "data": {
          "properties": {
            "__cloneCount": {
              "type": "long"
            },
            "__criblEventType": {
              "type": "keyword"
            },
            "__final": {
              "type": "boolean"
            },
            "__jsonFail": {
              "type": "boolean"
            },
            "__raw": {
              "type": "text"
            },
            "__socketAddr": {
              "type": "keyword"
            },
            "__srcIpPort": {
              "type": "keyword"
            },
            "_raw": {
              "type": "text"
            },
            "_time": {
              "type": "float"
            },
            "body": {
              "type": "text"
            },
            "opts": {
              "properties": {
                "_dst": {
                  "type": "keyword"
                },
                "_src": {
                  "type": "keyword"
                },
                "streamOptions": {
                  "properties": {
                    "highWaterMark": {
                      "type": "long"
                    }
                  }
                },
                "timeout": {
                  "type": "long"
                }
              }
            },
            "req": {
              "type": "text"
            },
            "reqId": {
              "type": "long"
            },
            "type": {
              "type": "keyword"
            },
            "workerId": {
              "type": "keyword"
            }
          }
        },
        "dead": {
          "type": "keyword"
        },
        "delimiterRegex": {
          "type": "keyword"
        },
        "destPath": {
          "type": "keyword"
        },
        "destinationDir": {
          "type": "keyword"
        },
        "diagfile": {
          "type": "keyword"
        },
        "diaghost": {
          "type": "keyword"
        },
        "dir": {
          "type": "keyword"
        },
        "directorties": {
          "type": "text"
        },
        "directory": {
          "type": "text"
        },
        "disabled": {
          "type": "boolean"
        },
        "diskUsage": {
          "properties": {
            "bytesAvailable": {
              "type": "long"
            },
            "bytesUsed": {
              "type": "long"
            },
            "diskPath": {
              "type": "text"
            },
            "totalDiskSize": {
              "type": "long"
            }
          }
        },
        "dnsResolvePeriodSec": {
          "type": "long"
        },
        "downloadUrl": {
          "type": "text"
        },
        "dropEventsMode": {
          "type": "boolean"
        },
        "dropped": {
          "type": "long"
        },
        "droppedEvents": {
          "type": "long"
        },
        "duration": {
          "type": "long"
        },
        "durationSeconds": {
          "type": "long"
        },
        "earliest": {
          "type": "long"
        },
        "elapsed": {
          "type": "long"
        },
        "elasticAPI": {
          "type": "keyword"
        },
        "eluPerc": {
          "type": "float"
        },
        "enableACK": {
          "type": "boolean"
        },
        "enableAssumeRole": {
          "type": "boolean"
        },
        "enableHeader": {
          "type": "boolean"
        },
        "enableMultiMetrics": {
          "type": "boolean"
        },
        "enableProxyHeader": {
          "type": "boolean"
        },
        "enableSQSAssumeRole": {
          "type": "boolean"
        },
        "enableUnixPath": {
          "type": "boolean"
        },
        "enabled": {
          "type": "long"
        },
        "endpoint": {
          "properties": {
            "host": {
              "type": "keyword"
            },
            "maxVersion": {
              "type": "keyword"
            },
            "minVersion": {
              "type": "keyword"
            },
            "port": {
              "type": "long"
            },
            "rejectUnauthorized": {
              "type": "boolean"
            },
            "servername": {
              "type": "keyword"
            },
            "tls": {
              "type": "boolean"
            }
          }
        },
        "endpoints": {
          "properties": {
            "family": {
              "type": "long"
            },
            "host": {
              "type": "keyword"
            },
            "key": {
              "type": "keyword"
            },
            "resolvedHost": {
              "type": "keyword"
            },
            "stats": {
              "properties": {
                "bytes": {
                  "type": "long"
                },
                "errors": {
                  "type": "long"
                },
                "events": {
                  "type": "long"
                },
                "health": {
                  "type": "long"
                },
                "lastFlushBytes": {
                  "type": "long"
                },
                "lastFlushTime": {
                  "type": "long"
                },
                "requests": {
                  "type": "long"
                },
                "totalBytes": {
                  "type": "long"
                },
                "totalErrors": {
                  "type": "long"
                },
                "totalEvents": {
                  "type": "long"
                },
                "totalRequests": {
                  "type": "long"
                }
              }
            },
            "url": {
              "type": "keyword"
            },
            "weight": {
              "type": "long"
            }
          }
        },
        "endtime": {
          "type": "date"
        },
        "env": {
          "type": "keyword"
        },
        "err": {
          "properties": {
            "code": {
              "type": "keyword"
            },
            "errno": {
              "type": "long"
            },
            "message": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "name": {
              "type": "keyword"
            },
            "originalError": {
              "properties": {
                "code": {
                  "type": "keyword"
                },
                "message": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                },
                "stack": {
                  "type": "text"
                }
              }
            },
            "path": {
              "type": "keyword"
            },
            "req": {
              "properties": {
                "body": {
                  "type": "text"
                },
                "opts": {
                  "properties": {
                    "_dst": {
                      "type": "keyword"
                    },
                    "_src": {
                      "type": "keyword"
                    },
                    "service": {
                      "type": "keyword"
                    },
                    "workerProcessFilter": {
                      "type": "keyword"
                    }
                  }
                },
                "req": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "stack": {
              "type": "text"
            },
            "syscall": {
              "type": "keyword"
            }
          }
        },
        "error": {
          "properties": {
            "__cloneCount": {
              "type": "long"
            },
            "__criblEventType": {
              "type": "keyword"
            },
            "__final": {
              "type": "boolean"
            },
            "__raw": {
              "type": "text"
            },
            "__socketAddr": {
              "type": "keyword"
            },
            "__srcIpPort": {
              "type": "keyword"
            },
            "details": {
              "properties": {
                "host": {
                  "type": "keyword"
                },
                "method": {
                  "type": "keyword"
                },
                "path": {
                  "type": "keyword"
                },
                "port": {
                  "type": "keyword"
                }
              }
            },
            "error": {
              "properties": {
                "message": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                },
                "stack": {
                  "type": "text"
                }
              }
            },
            "host": {
              "type": "keyword"
            },
            "message": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "method": {
              "type": "keyword"
            },
            "name": {
              "type": "keyword"
            },
            "path": {
              "type": "keyword"
            },
            "port": {
              "type": "keyword"
            },
            "reason": {
              "properties": {
                "caused_by": {
                  "properties": {
                    "reason": {
                      "type": "text"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "details": {
                  "properties": {
                    "host": {
                      "type": "keyword"
                    },
                    "method": {
                      "type": "keyword"
                    },
                    "path": {
                      "type": "keyword"
                    },
                    "port": {
                      "type": "keyword"
                    }
                  }
                },
                "message": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                },
                "name": {
                  "type": "keyword"
                },
                "stack": {
                  "type": "text"
                },
                "status": {
                  "type": "long"
                },
                "type": {
                  "type": "keyword"
                },
                "statusCode": {
                  "type": "long"
                }
              }
            },
            "req": {
              "type": "text"
            },
            "reqId": {
              "type": "long"
            },
            "rpc": {
              "type": "boolean"
            },
            "stack": {
              "type": "text"
            },
            "status": {
              "type": "long"
            },
            "statusCode": {
              "type": "long"
            },
            "type": {
              "type": "keyword"
            }
          }
        },
        "errors": {
          "type": "long"
        },
        "eventBreakerRegex": {
          "type": "keyword"
        },
        "excludeFields": {
          "properties": {
            "fieldname": {
              "type": "keyword"
            },
            "regexList": {
              "properties": {
                "negated": {
                  "type": "boolean"
                }
              }
            }
          }
        },
        "excludeSelf": {
          "type": "boolean"
        },
        "exclusive": {
          "type": "boolean"
        },
        "executor": {
          "properties": {
            "collectStep": {
              "type": "keyword"
            },
            "collectibles": {
              "properties": {
                "collectorId": {
                  "type": "keyword"
                },
                "contentCreated": {
                  "type": "date"
                },
                "contentExpiration": {
                  "type": "date"
                },
                "contentId": {
                  "type": "keyword"
                },
                "contentTime": {
                  "type": "float"
                },
                "contentType": {
                  "type": "keyword"
                },
                "contentUri": {
                  "type": "keyword"
                },
                "earliest": {
                  "type": "long"
                },
                "fakeDiscover": {
                  "type": "boolean"
                },
                "guid": {
                  "type": "keyword"
                },
                "host": {
                  "type": "keyword"
                },
                "latest": {
                  "type": "long"
                },
                "source": {
                  "type": "keyword"
                },
                "taskId": {
                  "type": "keyword"
                }
              }
            },
            "heartbeatPeriod": {
              "type": "long"
            },
            "input": {
              "properties": {
                "breakerRulesets": {
                  "type": "keyword"
                },
                "filter": {
                  "type": "keyword"
                },
                "id": {
                  "type": "keyword"
                },
                "metadata": {
                  "properties": {
                    "name": {
                      "type": "keyword"
                    },
                    "value": {
                      "type": "text",
                      "fields": {
                        "keyword": {
                          "type": "keyword",
                          "ignore_above": 256
                        }
                      }
                    }
                  }
                },
                "preprocess": {
                  "properties": {
                    "disabled": {
                      "type": "boolean"
                    }
                  }
                },
                "sendToRoutes": {
                  "type": "boolean"
                },
                "staleChannelFlushMs": {
                  "type": "long"
                },
                "throttleRatePerSec": {
                  "type": "keyword"
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "type": {
              "type": "keyword"
            }
          }
        },
        "executorId": {
          "type": "keyword"
        },
        "existing": {
          "properties": {
            "genId": {
              "type": "long"
            },
            "guid": {
              "type": "keyword"
            },
            "missedHBLimit": {
              "type": "long"
            },
            "period": {
              "type": "keyword"
            },
            "url": {
              "type": "keyword"
            }
          }
        },
        "existingOrNew": {
          "type": "keyword"
        },
        "existingRule": {
          "type": "keyword"
        },
        "exit": {
          "type": "long"
        },
        "exitCode": {
          "type": "long"
        },
        "expired": {
          "properties": {
            "guid": {
              "type": "keyword"
            },
            "lastHB": {
              "type": "long"
            },
            "task": {
              "properties": {
                "dir": {
                  "type": "keyword"
                },
                "executor": {
                  "properties": {
                    "collectStep": {
                      "type": "keyword"
                    },
                    "collectibles": {
                      "type": "keyword"
                    },
                    "heartbeatPeriod": {
                      "type": "long"
                    },
                    "input": {
                      "type": "keyword"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "jobId": {
                  "type": "keyword"
                },
                "logLevel": {
                  "type": "keyword"
                },
                "task": {
                  "properties": {
                    "conf": {
                      "type": "keyword"
                    },
                    "destructive": {
                      "type": "boolean"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "taskId": {
                  "type": "keyword"
                }
              }
            },
            "workerProcess": {
              "type": "keyword"
            }
          }
        },
        "expiryTime": {
          "type": "long"
        },
        "expression": {
          "type": "keyword"
        },
        "extractDir": {
          "type": "keyword"
        },
        "extractedSource": {
          "type": "keyword"
        },
        "failed": {
          "type": "long"
        },
        "failedRequestLoggingMode": {
          "type": "keyword"
        },
        "failover": {
          "properties": {
            "missedHBLimit": {
              "type": "long"
            },
            "period": {
              "type": "keyword"
            },
            "volume": {
              "type": "keyword"
            }
          }
        },
        "fakeDiscover": {
          "type": "boolean"
        },
        "fieldsLineRegex": {
          "type": "keyword"
        },
        "file": {
          "type": "keyword"
        },
        "fileFilter": {
          "type": "keyword"
        },
        "fileName": {
          "type": "keyword"
        },
        "fileNameSuffix": {
          "type": "keyword"
        },
        "filename": {
          "type": "keyword"
        },
        "filenames": {
          "type": "text"
        },
        "files": {
          "properties": {
            "created": {
              "type": "keyword"
            },
            "deleted": {
              "type": "keyword"
            },
            "modified": {
              "type": "keyword"
            }
          }
        },
        "first": {
          "properties": {
            "__offset": {
              "type": "long"
            }
          }
        },
        "firstOffset": {
          "type": "keyword"
        },
        "flushEventCount": {
          "type": "long"
        },
        "flushPeriodSec": {
          "type": "long"
        },
        "force": {
          "type": "boolean"
        },
        "format": {
          "type": "keyword"
        },
        "fromBeginning": {
          "type": "boolean"
        },
        "fullFidelity": {
          "type": "boolean"
        },
        "function": {
          "type": "keyword"
        },
        "functions": {
          "type": "keyword"
        },
        "fwd": {
          "type": "keyword"
        },
        "fwdType": {
          "type": "keyword"
        },
        "garbageCollectionPeriodSecs": {
          "type": "long"
        },
        "genId": {
          "type": "long"
        },
        "getRecordsLimit": {
          "type": "long"
        },
        "getRecordsLimitTotal": {
          "type": "long"
        },
        "group": {
          "type": "keyword"
        },
        "groupId": {
          "type": "keyword"
        },
        "groupProtocol": {
          "type": "keyword"
        },
        "guid": {
          "type": "keyword"
        },
        "guids": {
          "type": "text"
        },
        "hashFile": {
          "type": "keyword"
        },
        "hashUrl": {
          "type": "keyword"
        },
        "headerLineRegex": {
          "type": "keyword"
        },
        "heartbeatInterval": {
          "type": "long"
        },
        "highWatermark": {
          "type": "keyword"
        },
        "hint": {
          "type": "keyword"
        },
        "history": {
          "properties": {
            "args": {
              "properties": {
                "collector": {
                  "properties": {
                    "conf": {
                      "type": "text"
                    },
                    "destructive": {
                      "type": "boolean"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "groupId": {
                  "type": "keyword"
                },
                "id": {
                  "type": "keyword"
                },
                "input": {
                  "properties": {
                    "breakerRulesets": {
                      "type": "text"
                    },
                    "filter": {
                      "type": "keyword"
                    },
                    "metadata": {
                      "type": "keyword"
                    },
                    "output": {
                      "type": "keyword"
                    },
                    "pipeline": {
                      "type": "keyword"
                    },
                    "preprocess": {
                      "type": "keyword"
                    },
                    "sendToRoutes": {
                      "type": "boolean"
                    },
                    "staleChannelFlushMs": {
                      "type": "long"
                    },
                    "throttleRatePerSec": {
                      "type": "keyword"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "resumeOnBoot": {
                  "type": "boolean"
                },
                "run": {
                  "properties": {
                    "earliest": {
                      "type": "keyword"
                    },
                    "expression": {
                      "type": "keyword"
                    },
                    "jobTimeout": {
                      "type": "keyword"
                    },
                    "latest": {
                      "type": "keyword"
                    },
                    "logLevel": {
                      "type": "keyword"
                    },
                    "maxTaskReschedule": {
                      "type": "long"
                    },
                    "maxTaskSize": {
                      "type": "keyword"
                    },
                    "minTaskSize": {
                      "type": "keyword"
                    },
                    "mode": {
                      "type": "keyword"
                    },
                    "now": {
                      "type": "date"
                    },
                    "rescheduleDroppedTasks": {
                      "type": "boolean"
                    },
                    "taskHeartbeatPeriod": {
                      "type": "long"
                    },
                    "timeRangeType": {
                      "type": "keyword"
                    },
                    "timestampTimezone": {
                      "type": "keyword"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "schedule": {
                  "properties": {
                    "cronSchedule": {
                      "type": "keyword"
                    },
                    "enabled": {
                      "type": "boolean"
                    },
                    "maxConcurrentRuns": {
                      "type": "long"
                    },
                    "resumeMissed": {
                      "type": "boolean"
                    },
                    "run": {
                      "type": "keyword"
                    },
                    "skippable": {
                      "type": "boolean"
                    }
                  }
                },
                "ttl": {
                  "type": "keyword"
                },
                "type": {
                  "type": "keyword"
                },
                "workerAffinity": {
                  "type": "boolean"
                }
              }
            },
            "id": {
              "type": "keyword"
            },
            "stats": {
              "properties": {
                "collectedBytes": {
                  "type": "long"
                },
                "collectedEvents": {
                  "type": "long"
                },
                "discoveredEvents": {
                  "type": "long"
                },
                "discoveryComplete": {
                  "type": "long"
                },
                "filteredEvents": {
                  "type": "long"
                },
                "flushedBuffers": {
                  "type": "long"
                },
                "state": {
                  "properties": {
                    "cancelled": {
                      "type": "long"
                    },
                    "finished": {
                      "type": "long"
                    },
                    "initializing": {
                      "type": "long"
                    },
                    "pending": {
                      "type": "long"
                    },
                    "running": {
                      "type": "long"
                    }
                  }
                },
                "tasks": {
                  "properties": {
                    "cancelled": {
                      "type": "long"
                    },
                    "count": {
                      "type": "long"
                    },
                    "failed": {
                      "type": "long"
                    },
                    "finished": {
                      "type": "long"
                    },
                    "inFlight": {
                      "type": "long"
                    },
                    "maxExecutionTime": {
                      "type": "long"
                    },
                    "minExecutionTime": {
                      "type": "long"
                    },
                    "orphaned": {
                      "type": "long"
                    },
                    "totalExecutionTime": {
                      "type": "long"
                    }
                  }
                },
                "totalResults": {
                  "type": "long"
                }
              }
            },
            "status": {
              "properties": {
                "state": {
                  "type": "long"
                }
              }
            }
          }
        },
        "host": {
          "properties": {
            "name": {
              "type": "keyword"
            }
          }
        },
        "hostname": {
          "type": "keyword"
        },
        "hosts": {
          "properties": {
            "host": {
              "type": "keyword"
            },
            "port": {
              "type": "long"
            },
            "tls": {
              "type": "keyword"
            },
            "weight": {
              "type": "long"
            }
          }
        },
        "http": {
          "properties": {
            "response": {
              "properties": {
                "status_code": {
                  "type": "short"
                }
              }
            }
          }
        },
        "id": {
          "type": "keyword"
        },
        "idleTimeout": {
          "type": "long"
        },
        "inBytes": {
          "type": "long"
        },
        "inEvents": {
          "type": "long"
        },
        "includeUnidentifiableBinary": {
          "type": "boolean"
        },
        "index": {
          "type": "keyword"
        },
        "indexerDiscovery": {
          "type": "boolean"
        },
        "indexerDiscoveryConfigs": {
          "properties": {
            "authToken": {
              "type": "keyword"
            },
            "authType": {
              "type": "keyword"
            },
            "masterUri": {
              "type": "keyword"
            },
            "refreshIntervalSec": {
              "type": "long"
            },
            "site": {
              "type": "keyword"
            }
          }
        },
        "info": {
          "properties": {
            "architecture": {
              "type": "keyword"
            },
            "cpus": {
              "type": "long"
            },
            "cribl": {
              "properties": {
                "version": {
                  "type": "keyword"
                }
              }
            },
            "hostname": {
              "type": "keyword"
            },
            "node": {
              "type": "keyword"
            },
            "platform": {
              "type": "keyword"
            },
            "release": {
              "type": "keyword"
            },
            "totalmem": {
              "type": "long"
            }
          }
        },
        "infoFile": {
          "type": "keyword"
        },
        "input": {
          "type": "keyword"
        },
        "inputId": {
          "type": "keyword"
        },
        "instanceId": {
          "type": "keyword"
        },
        "interval": {
          "type": "long"
        },
        "intervalsSecs": {
          "type": "long"
        },
        "ip": {
          "type": "keyword"
        },
        "ipWhitelistRegex": {
          "type": "keyword"
        },
        "isConfigHelper": {
          "type": "boolean"
        },
        "isForkedProcess": {
          "type": "boolean"
        },
        "isLeader": {
          "type": "boolean"
        },
        "isManaged": {
          "type": "boolean"
        },
        "isRunning": {
          "type": "boolean"
        },
        "isSessionStale": {
          "type": "boolean"
        },
        "isStale": {
          "type": "boolean"
        },
        "isStandalone": {
          "type": "boolean"
        },
        "isV4": {
          "type": "boolean"
        },
        "job": {
          "properties": {
            "collector": {
              "properties": {
                "conf": {
                  "properties": {
                    "__scheduling": {
                      "properties": {
                        "stateTracking": {
                          "type": "keyword"
                        }
                      }
                    },
                    "app_id": {
                      "type": "keyword"
                    },
                    "authHeaderExpr": {
                      "type": "keyword"
                    },
                    "authHeaderKey": {
                      "type": "keyword"
                    },
                    "authentication": {
                      "type": "keyword"
                    },
                    "clientSecretParamName": {
                      "type": "keyword"
                    },
                    "client_secret": {
                      "type": "keyword"
                    },
                    "collectBody": {
                      "type": "keyword"
                    },
                    "collectMethod": {
                      "type": "keyword"
                    },
                    "collectRequestHeaders": {
                      "type": "keyword"
                    },
                    "collectRequestParams": {
                      "type": "keyword"
                    },
                    "collectScript": {
                      "type": "keyword"
                    },
                    "collectUrl": {
                      "type": "keyword"
                    },
                    "connectionId": {
                      "type": "keyword"
                    },
                    "content_type": {
                      "type": "keyword"
                    },
                    "disableTimeFilter": {
                      "type": "boolean"
                    },
                    "discoverScript": {
                      "type": "keyword"
                    },
                    "discovery": {
                      "properties": {
                        "discoverDataField": {
                          "type": "keyword"
                        },
                        "discoverMethod": {
                          "type": "keyword"
                        },
                        "discoverRequestParams": {
                          "type": "keyword"
                        },
                        "discoverType": {
                          "type": "keyword"
                        },
                        "discoverUrl": {
                          "type": "keyword"
                        }
                      }
                    },
                    "earliest": {
                      "type": "keyword"
                    },
                    "endpoint": {
                      "type": "keyword"
                    },
                    "ingestionLag": {
                      "type": "long"
                    },
                    "latest": {
                      "type": "keyword"
                    },
                    "loginBody": {
                      "type": "text"
                    },
                    "loginUrl": {
                      "type": "keyword"
                    },
                    "outputMode": {
                      "type": "keyword"
                    },
                    "pagination": {
                      "properties": {
                        "attribute": {
                          "type": "keyword"
                        },
                        "limit": {
                          "type": "long"
                        },
                        "limitField": {
                          "type": "keyword"
                        },
                        "maxPages": {
                          "type": "long"
                        },
                        "offset": {
                          "type": "long"
                        },
                        "offsetField": {
                          "type": "keyword"
                        },
                        "totalRecordField": {
                          "type": "keyword"
                        },
                        "type": {
                          "type": "keyword"
                        },
                        "zeroIndexed": {
                          "type": "boolean"
                        }
                      }
                    },
                    "password": {
                      "type": "keyword"
                    },
                    "plan_type": {
                      "type": "keyword"
                    },
                    "query": {
                      "type": "keyword"
                    },
                    "rejectUnauthorized": {
                      "type": "boolean"
                    },
                    "search": {
                      "type": "keyword"
                    },
                    "searchHead": {
                      "type": "keyword"
                    },
                    "shell": {
                      "type": "keyword"
                    },
                    "tenant_id": {
                      "type": "keyword"
                    },
                    "timeout": {
                      "type": "long"
                    },
                    "tokenRespAttribute": {
                      "type": "keyword"
                    },
                    "useRoundRobinDns": {
                      "type": "boolean"
                    },
                    "username": {
                      "type": "keyword"
                    }
                  }
                },
                "destructive": {
                  "type": "boolean"
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "groupId": {
              "type": "keyword"
            },
            "id": {
              "type": "keyword"
            },
            "input": {
              "properties": {
                "breakerRulesets": {
                  "type": "keyword"
                },
                "id": {
                  "type": "keyword"
                },
                "metadata": {
                  "properties": {
                    "name": {
                      "type": "keyword"
                    },
                    "value": {
                      "type": "keyword"
                    }
                  }
                },
                "output": {
                  "type": "keyword"
                },
                "pipeline": {
                  "type": "keyword"
                },
                "preprocess": {
                  "properties": {
                    "disabled": {
                      "type": "boolean"
                    }
                  }
                },
                "sendToRoutes": {
                  "type": "boolean"
                },
                "staleChannelFlushMs": {
                  "type": "long"
                },
                "throttleRatePerSec": {
                  "type": "keyword"
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "resumeOnBoot": {
              "type": "boolean"
            },
            "schedule": {
              "properties": {
                "cronSchedule": {
                  "type": "keyword"
                },
                "enabled": {
                  "type": "boolean"
                },
                "maxConcurrentRuns": {
                  "type": "long"
                },
                "resumeMissed": {
                  "type": "boolean"
                },
                "run": {
                  "properties": {
                    "discoverToRoutes": {
                      "type": "boolean"
                    },
                    "earliest": {
                      "type": "keyword"
                    },
                    "expression": {
                      "type": "keyword"
                    },
                    "filterTimeField": {
                      "type": "keyword"
                    },
                    "jobTimeout": {
                      "type": "keyword"
                    },
                    "latest": {
                      "type": "keyword"
                    },
                    "logLevel": {
                      "type": "keyword"
                    },
                    "maxTaskReschedule": {
                      "type": "long"
                    },
                    "maxTaskSize": {
                      "type": "keyword"
                    },
                    "minTaskSize": {
                      "type": "keyword"
                    },
                    "mode": {
                      "type": "keyword"
                    },
                    "rescheduleDroppedTasks": {
                      "type": "boolean"
                    },
                    "timeRangeType": {
                      "type": "keyword"
                    },
                    "timestampTimezone": {
                      "type": "keyword"
                    }
                  }
                },
                "skippable": {
                  "type": "boolean"
                }
              }
            },
            "ttl": {
              "type": "keyword"
            },
            "type": {
              "type": "keyword"
            },
            "workerAffinity": {
              "type": "boolean"
            }
          }
        },
        "jobId": {
          "type": "keyword"
        },
        "jobIds": {
          "type": "keyword"
        },
        "jobTimeout": {
          "type": "keyword"
        },
        "jobs": {
          "type": "text"
        },
        "jobsStore": {
          "type": "boolean"
        },
        "journals": {
          "type": "text"
        },
        "keep": {
          "type": "boolean"
        },
        "keepAliveTime": {
          "type": "long"
        },
        "keepAliveTimeout": {
          "type": "long"
        },
        "keepFor": {
          "type": "long"
        },
        "key": {
          "type": "keyword"
        },
        "keyExpr": {
          "type": "keyword"
        },
        "killed": {
          "type": "long"
        },
        "lastCursor": {
          "properties": {
            "offset": {
              "type": "long"
            },
            "sliceId": {
              "type": "long"
            }
          }
        },
        "lastIndexer": {
          "type": "keyword"
        },
        "lastModifiedTime": {
          "type": "long"
        },
        "latest": {
          "type": "long"
        },
        "leaderId": {
          "type": "keyword"
        },
        "lease": {
          "properties": {
            "guid": {
              "type": "keyword"
            },
            "missedHBLimit": {
              "type": "long"
            },
            "period": {
              "type": "keyword"
            },
            "url": {
              "type": "keyword"
            }
          }
        },
        "leaseFile": {
          "type": "keyword"
        },
        "level": {
          "type": "keyword"
        },
        "licenseLimit": {
          "type": "long"
        },
        "loadBalanceStatsPeriodSec": {
          "type": "long"
        },
        "loadBalanced": {
          "type": "boolean"
        },
        "loadBalancingAlgorithm": {
          "type": "keyword"
        },
        "logGroupName": {
          "type": "keyword"
        },
        "logLevel": {
          "type": "keyword"
        },
        "logNames": {
          "type": "text"
        },
        "logStreamEnv": {
          "type": "keyword"
        },
        "logStreamName": {
          "type": "keyword"
        },
        "lookups": {
          "type": "long"
        },
        "matchMode": {
          "type": "keyword"
        },
        "matchType": {
          "type": "keyword"
        },
        "maxActiveCxn": {
          "type": "long"
        },
        "maxActiveReq": {
          "type": "long"
        },
        "maxBufferSize": {
          "type": "long"
        },
        "maxBytes": {
          "type": "long"
        },
        "maxBytesPerPartition": {
          "type": "long"
        },
        "maxCacheSize": {
          "type": "long"
        },
        "maxConcurrentFileParts": {
          "type": "long"
        },
        "maxConcurrentSenders": {
          "type": "long"
        },
        "maxEventBytes": {
          "type": "long"
        },
        "maxFailedHealthChecks": {
          "type": "long"
        },
        "maxFileIdleTimeSec": {
          "type": "long"
        },
        "maxFileOpenTimeSec": {
          "type": "long"
        },
        "maxFileSize": {
          "type": "keyword"
        },
        "maxFileSizeMB": {
          "type": "long"
        },
        "maxJobs": {
          "type": "long"
        },
        "maxMessages": {
          "type": "long"
        },
        "maxMetrics": {
          "type": "long"
        },
        "maxMissedKeepAlives": {
          "type": "long"
        },
        "maxOpenFiles": {
          "type": "long"
        },
        "maxPayloadEvents": {
          "type": "long"
        },
        "maxPayloadSize": {
          "type": "long"
        },
        "maxPayloadSizeKB": {
          "type": "long"
        },
        "maxProcs": {
          "type": "long"
        },
        "maxQueueSize": {
          "type": "keyword"
        },
        "maxRecordSizeKB": {
          "type": "long"
        },
        "maxResults": {
          "type": "long"
        },
        "maxRetries": {
          "type": "long"
        },
        "maxS2Sversion": {
          "type": "keyword"
        },
        "maxSize": {
          "type": "keyword"
        },
        "maxVersion": {
          "type": "keyword"
        },
        "mem": {
          "properties": {
            "ext": {
              "type": "long"
            },
            "heap": {
              "type": "long"
            },
            "rss": {
              "type": "long"
            }
          }
        },
        "memberId": {
          "type": "keyword"
        },
        "memory": {
          "properties": {
            "free": {
              "type": "long"
            },
            "size": {
              "type": "long"
            },
            "total": {
              "type": "long"
            }
          }
        },
        "message": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "meta": {
          "properties": {
            "name": {
              "type": "keyword"
            }
          }
        },
        "metadata": {
          "properties": {
            "name": {
              "type": "keyword"
            },
            "value": {
              "type": "text"
            }
          }
        },
        "method": {
          "type": "keyword"
        },
        "metric": {
          "properties": {
            "dimensions": {
              "type": "keyword"
            },
            "name": {
              "type": "keyword"
            },
            "type": {
              "type": "keyword"
            }
          }
        },
        "metricsAddress": {
          "properties": {
            "path": {
              "type": "keyword"
            }
          }
        },
        "metricsFieldsBlacklist": {
          "type": "keyword"
        },
        "metricsGCPeriod": {
          "type": "keyword"
        },
        "metricsMaxCardinality": {
          "type": "long"
        },
        "metricsNeverDropList": {
          "type": "keyword"
        },
        "metricsWorkerIdBlacklist": {
          "type": "keyword"
        },
        "minEvents": {
          "type": "long"
        },
        "minVersion": {
          "type": "keyword"
        },
        "minimizeDuplicates": {
          "type": "boolean"
        },
        "missed": {
          "type": "long"
        },
        "mode": {
          "type": "keyword"
        },
        "modelName": {
          "type": "keyword"
        },
        "msg": {
          "properties": {
            "__cloneCount": {
              "type": "long"
            },
            "__criblEventType": {
              "type": "keyword"
            },
            "__final": {
              "type": "boolean"
            },
            "__raw": {
              "type": "text"
            },
            "__socketAddr": {
              "type": "keyword"
            },
            "__srcIpPort": {
              "type": "keyword"
            },
            "_time": {
              "type": "float"
            },
            "body": {
              "type": "text"
            },
            "opts": {
              "properties": {
                "_dst": {
                  "type": "keyword"
                },
                "_src": {
                  "type": "keyword"
                },
                "timeout": {
                  "type": "long"
                }
              }
            },
            "req": {
              "type": "keyword"
            },
            "reqId": {
              "type": "keyword"
            },
            "type": {
              "type": "keyword"
            },
            "workerId": {
              "type": "keyword"
            }
          }
        },
        "name": {
          "type": "keyword"
        },
        "nestedFields": {
          "type": "text"
        },
        "newCursor": {
          "properties": {
            "offset": {
              "type": "long"
            },
            "sliceId": {
              "type": "long"
            }
          }
        },
        "notifications": {
          "properties": {
            "condition": {
              "type": "keyword"
            },
            "conf": {
              "properties": {
                "name": {
                  "type": "keyword"
                },
                "timeWindow": {
                  "type": "keyword"
                },
                "usageThreshold": {
                  "type": "long"
                }
              }
            },
            "disabled": {
              "type": "boolean"
            },
            "group": {
              "type": "keyword"
            },
            "id": {
              "type": "keyword"
            },
            "targets": {
              "type": "text"
            }
          }
        },
        "nullFieldVal": {
          "type": "keyword"
        },
        "numFlushed": {
          "type": "long"
        },
        "numInFlight": {
          "type": "long"
        },
        "numMetrics": {
          "type": "long"
        },
        "numProcs": {
          "type": "long"
        },
        "numReceivers": {
          "type": "long"
        },
        "numRows": {
          "type": "long"
        },
        "numToAllow": {
          "type": "long"
        },
        "numToFlush": {
          "type": "long"
        },
        "objectACL": {
          "type": "text"
        },
        "octetCounting": {
          "type": "boolean"
        },
        "offset": {
          "type": "long"
        },
        "oldPid": {
          "type": "long"
        },
        "onBackpressure": {
          "type": "keyword"
        },
        "openCxn": {
          "type": "long"
        },
        "openFiles": {
          "type": "long"
        },
        "openFilesCount": {
          "type": "long"
        },
        "os": {
          "type": "keyword"
        },
        "outBytes": {
          "type": "long"
        },
        "outEvents": {
          "type": "long"
        },
        "output": {
          "type": "keyword"
        },
        "outputId": {
          "type": "keyword"
        },
        "overwrite": {
          "type": "boolean"
        },
        "pack": {
          "type": "keyword"
        },
        "packDir": {
          "type": "keyword"
        },
        "packNames": {
          "type": "text"
        },
        "packageFile": {
          "type": "keyword"
        },
        "packs": {
          "type": "text"
        },
        "parquetChunkDownloadTimeout": {
          "type": "long"
        },
        "parquetChunkSizeMB": {
          "type": "long"
        },
        "partition": {
          "type": "long"
        },
        "partitionExpr": {
          "type": "keyword"
        },
        "password": {
          "type": "keyword"
        },
        "path": {
          "type": "keyword"
        },
        "pathsToRemove": {
          "type": "text"
        },
        "payloadFormat": {
          "type": "keyword"
        },
        "pendingDuration": {
          "type": "long"
        },
        "period": {
          "type": "long"
        },
        "periodsToWait": {
          "type": "long"
        },
        "persistence": {
          "properties": {
            "enable": {
              "type": "boolean"
            },
            "maxDataSize": {
              "type": "keyword"
            }
          }
        },
        "pid": {
          "type": "long"
        },
        "pipe": {
          "type": "keyword"
        },
        "pipeId": {
          "type": "keyword"
        },
        "pipeline": {
          "type": "keyword"
        },
        "pipelines": {
          "type": "keyword"
        },
        "planType": {
          "type": "keyword"
        },
        "pointsPerInterval": {
          "type": "long"
        },
        "policies": {
          "type": "keyword"
        },
        "policy": {
          "properties": {
            "args": {
              "type": "keyword"
            },
            "description": {
              "type": "keyword"
            },
            "template": {
              "type": "text"
            },
            "title": {
              "type": "keyword"
            }
          }
        },
        "pollTimeout": {
          "type": "long"
        },
        "poolSize": {
          "properties": {
            "isAuto": {
              "type": "boolean"
            },
            "role": {
              "type": "keyword"
            },
            "value": {
              "type": "long"
            }
          }
        },
        "port": {
          "type": "long"
        },
        "pq": {
          "properties": {
            "commitFrequency": {
              "type": "long"
            },
            "compress": {
              "type": "keyword"
            },
            "maxBufferSize": {
              "type": "long"
            },
            "maxFileSize": {
              "type": "keyword"
            },
            "maxSize": {
              "type": "keyword"
            },
            "mode": {
              "type": "keyword"
            },
            "path": {
              "type": "keyword"
            }
          }
        },
        "pqBufferedEvents": {
          "type": "long"
        },
        "pqCompress": {
          "type": "keyword"
        },
        "pqEnabled": {
          "type": "boolean"
        },
        "pqInBytes": {
          "type": "long"
        },
        "pqInEvents": {
          "type": "long"
        },
        "pqIsEngaged": {
          "type": "boolean"
        },
        "pqMaxFileSize": {
          "type": "keyword"
        },
        "pqMaxSize": {
          "type": "keyword"
        },
        "pqOnBackpressure": {
          "type": "keyword"
        },
        "pqOutBytes": {
          "type": "long"
        },
        "pqOutEvents": {
          "type": "long"
        },
        "pqPath": {
          "type": "keyword"
        },
        "pqSize": {
          "type": "long"
        },
        "pqStrictOrdering": {
          "type": "boolean"
        },
        "pqTotalBytes": {
          "type": "long"
        },
        "pqTotalEvents": {
          "type": "long"
        },
        "prefix": {
          "type": "keyword"
        },
        "preprocess": {
          "properties": {
            "disabled": {
              "type": "boolean"
            }
          }
        },
        "previousState": {
          "type": "keyword"
        },
        "processorId": {
          "type": "keyword"
        },
        "procs": {
          "type": "long"
        },
        "query": {
          "type": "keyword"
        },
        "queueName": {
          "type": "keyword"
        },
        "queueSize": {
          "type": "long"
        },
        "queueType": {
          "type": "keyword"
        },
        "r": {
          "type": "long"
        },
        "readMode": {
          "type": "keyword"
        },
        "reapableBackups": {
          "properties": {
            "createdDate": {
              "type": "date"
            },
            "path": {
              "type": "keyword"
            }
          }
        },
        "reapedJobs": {
          "type": "long"
        },
        "reapedTasks": {
          "type": "long"
        },
        "reason": {
          "properties": {
            "code": {
              "type": "keyword"
            },
            "errno": {
              "type": "long"
            },
            "host": {
              "type": "keyword"
            },
            "message": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "method": {
              "type": "keyword"
            },
            "name": {
              "type": "keyword"
            },
            "path": {
              "type": "keyword"
            },
            "port": {
              "type": "keyword"
            },
            "reason": {
              "properties": {
                "message": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                },
                "name": {
                  "type": "keyword"
                },
                "reason": {
                  "properties": {
                    "code": {
                      "type": "keyword"
                    },
                    "message": {
                      "type": "text",
                      "fields": {
                        "keyword": {
                          "type": "keyword",
                          "ignore_above": 256
                        }
                      }
                    },
                    "stack": {
                      "type": "text"
                    }
                  }
                },
                "stack": {
                  "type": "keyword"
                }
              }
            },
            "req": {
              "properties": {
                "body": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                },
                "opts": {
                  "properties": {
                    "_dst": {
                      "type": "keyword"
                    },
                    "_src": {
                      "type": "keyword"
                    },
                    "service": {
                      "type": "keyword"
                    },
                    "workerProcessFilter": {
                      "type": "keyword"
                    }
                  }
                },
                "req": {
                  "type": "text"
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "stack": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "syscall": {
              "type": "keyword"
            }
          }
        },
        "reauthenticationThreshold": {
          "type": "long"
        },
        "rebalanceTimeout": {
          "type": "long"
        },
        "received": {
          "type": "long"
        },
        "reconnectInMs": {
          "type": "long"
        },
        "reconnectTime": {
          "type": "long"
        },
        "redirectUri": {
          "type": "keyword"
        },
        "redirect_uri": {
          "type": "keyword"
        },
        "refCount": {
          "type": "long"
        },
        "region": {
          "type": "keyword"
        },
        "rejectCxn": {
          "type": "long"
        },
        "rejectUnauthorized": {
          "type": "boolean"
        },
        "remainingRefs": {
          "type": "long"
        },
        "removeEmptyDirs": {
          "type": "boolean"
        },
        "removed": {
          "type": "long"
        },
        "reqId": {
          "type": "long"
        },
        "requestId": {
          "type": "keyword"
        },
        "requestTimeout": {
          "type": "long"
        },
        "requested": {
          "type": "long"
        },
        "res": {
          "properties": {
            "__cloneCount": {
              "type": "long"
            },
            "__criblEventType": {
              "type": "keyword"
            },
            "__final": {
              "type": "boolean"
            },
            "__raw": {
              "type": "text"
            },
            "__socketAddr": {
              "type": "keyword"
            },
            "__srcIpPort": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "body": {
              "type": "text"
            },
            "req": {
              "type": "text"
            },
            "reqId": {
              "type": "long"
            },
            "status": {
              "type": "long"
            },
            "type": {
              "type": "keyword"
            }
          }
        },
        "response_time": {
          "type": "long"
        },
        "resiliency": {
          "type": "keyword"
        },
        "restartDelay": {
          "type": "long"
        },
        "result": {
          "properties": {
            "availableVersions": {
              "properties": {
                "architecture": {
                  "type": "keyword"
                },
                "build": {
                  "type": "keyword"
                },
                "downloadUrl": {
                  "type": "keyword"
                },
                "fullVersion": {
                  "type": "keyword"
                },
                "major": {
                  "type": "long"
                },
                "minor": {
                  "type": "long"
                },
                "platform": {
                  "type": "keyword"
                },
                "point": {
                  "type": "long"
                },
                "preRelease": {
                  "type": "keyword"
                }
              }
            },
            "canUpgrade": {
              "type": "boolean"
            },
            "installedVersion": {
              "properties": {
                "architecture": {
                  "type": "keyword"
                },
                "build": {
                  "type": "keyword"
                },
                "fullVersion": {
                  "type": "keyword"
                },
                "major": {
                  "type": "long"
                },
                "minor": {
                  "type": "long"
                },
                "platform": {
                  "type": "keyword"
                },
                "point": {
                  "type": "long"
                },
                "preRelease": {
                  "type": "keyword"
                }
              }
            },
            "isSuccess": {
              "type": "boolean"
            },
            "message": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "result": {
              "properties": {
                "author": {
                  "type": "keyword"
                },
                "description": {
                  "type": "text"
                },
                "displayName": {
                  "type": "keyword"
                },
                "id": {
                  "type": "keyword"
                },
                "source": {
                  "type": "keyword"
                },
                "tags": {
                  "properties": {
                    "streamtags": {
                      "type": "keyword"
                    }
                  }
                },
                "version": {
                  "type": "keyword"
                }
              }
            },
            "upgradedToVersion": {
              "properties": {
                "architecture": {
                  "type": "keyword"
                },
                "build": {
                  "type": "keyword"
                },
                "fullVersion": {
                  "type": "keyword"
                },
                "major": {
                  "type": "long"
                },
                "minor": {
                  "type": "long"
                },
                "platform": {
                  "type": "keyword"
                },
                "point": {
                  "type": "long"
                },
                "preRelease": {
                  "type": "keyword"
                }
              }
            }
          }
        },
        "retryCount": {
          "type": "long"
        },
        "retryMs": {
          "type": "long"
        },
        "retryTime": {
          "type": "long"
        },
        "reuseConnections": {
          "type": "boolean"
        },
        "reverseResults": {
          "type": "boolean"
        },
        "role": {
          "type": "keyword"
        },
        "rowCount": {
          "type": "long"
        },
        "ruleType": {
          "type": "keyword"
        },
        "rules": {
          "properties": {
            "description": {
              "type": "text"
            },
            "filter": {
              "type": "keyword"
            },
            "final": {
              "type": "boolean"
            },
            "output": {
              "type": "keyword"
            },
            "rate": {
              "type": "long"
            }
          }
        },
        "running": {
          "type": "long"
        },
        "s2sVersion": {
          "type": "keyword"
        },
        "sampleId": {
          "type": "keyword"
        },
        "samplePeriodMS": {
          "type": "long"
        },
        "samples": {
          "properties": {
            "eventsPerSec": {
              "type": "long"
            },
            "sample": {
              "type": "keyword"
            }
          }
        },
        "scheduled": {
          "type": "keyword"
        },
        "scheduledJob": {
          "type": "keyword"
        },
        "schedulingPolicy": {
          "type": "keyword"
        },
        "sendToRoutes": {
          "type": "boolean"
        },
        "senderUnhealthyTimeAllowance": {
          "type": "long"
        },
        "sent": {
          "type": "long"
        },
        "sentAt": {
          "type": "long"
        },
        "servername": {
          "type": "keyword"
        },
        "service": {
          "type": "keyword"
        },
        "serviceId": {
          "type": "long"
        },
        "serviceInterval": {
          "type": "long"
        },
        "serviceName": {
          "type": "keyword"
        },
        "servicePeriodMS": {
          "type": "long"
        },
        "servicePeriodMin": {
          "type": "float"
        },
        "servicePeriodSecs": {
          "type": "long"
        },
        "sessionTimeout": {
          "type": "long"
        },
        "severity": {
          "type": "keyword"
        },
        "sfdcAccountName": {
          "type": "keyword"
        },
        "shardExpr": {
          "type": "keyword"
        },
        "shardIteratorType": {
          "type": "keyword"
        },
        "shouldMarkCriblBreaker": {
          "type": "boolean"
        },
        "shouldStartConsumer": {
          "type": "keyword"
        },
        "signal": {
          "type": "keyword"
        },
        "signature": {
          "type": "keyword"
        },
        "signatureVersion": {
          "type": "keyword"
        },
        "since": {
          "type": "long"
        },
        "singleMsgUdpPackets": {
          "type": "boolean"
        },
        "size": {
          "type": "long"
        },
        "skipOnError": {
          "type": "boolean"
        },
        "socketTimeout": {
          "type": "long"
        },
        "source": {
          "type": "keyword"
        },
        "sourceDir": {
          "type": "keyword"
        },
        "sourcetype": {
          "type": "keyword"
        },
        "splunkHecAPI": {
          "type": "keyword"
        },
        "splunkHecAcks": {
          "type": "boolean"
        },
        "src": {
          "type": "keyword"
        },
        "srcId": {
          "type": "keyword"
        },
        "ssl": {
          "type": "keyword"
        },
        "stack": {
          "type": "keyword"
        },
        "stagePath": {
          "type": "keyword"
        },
        "staleChannelFlushMs": {
          "type": "long"
        },
        "started": {
          "type": "long"
        },
        "starting": {
          "type": "long"
        },
        "starttime": {
          "type": "date"
        },
        "state": {
          "type": "keyword"
        },
        "status": {
          "properties": {
            "code": {
              "type": "long"
            },
            "error": {
              "properties": {
                "message": {
                  "type": "text",
                  "fields": {
                    "keyword": {
                      "type": "keyword",
                      "ignore_above": 256
                    }
                  }
                }
              }
            },
            "health": {
              "type": "keyword"
            },
            "timestamp": {
              "type": "long"
            }
          }
        },
        "stopped": {
          "type": "long"
        },
        "streamName": {
          "type": "keyword"
        },
        "subscriptions": {
          "properties": {
            "batchTimeout": {
              "type": "long"
            },
            "compress": {
              "type": "boolean"
            },
            "contentFormat": {
              "type": "keyword"
            },
            "heartbeatInterval": {
              "type": "long"
            },
            "querySelector": {
              "type": "keyword"
            },
            "readExistingEvents": {
              "type": "boolean"
            },
            "sendBookmarks": {
              "type": "boolean"
            },
            "subscriptionName": {
              "type": "keyword"
            },
            "targets": {
              "type": "text"
            },
            "version": {
              "type": "keyword"
            },
            "xmlQuery": {
              "type": "keyword"
            }
          }
        },
        "supressionMs": {
          "type": "long"
        },
        "sync": {
          "type": "boolean"
        },
        "systemFields": {
          "type": "keyword"
        },
        "tableIsNull": {
          "type": "boolean"
        },
        "tags": {
          "properties": {
            "system": {
              "type": "boolean"
            }
          }
        },
        "tailOnly": {
          "type": "boolean"
        },
        "task": {
          "properties": {
            "collectibles": {
              "properties": {
                "collectorId": {
                  "type": "keyword"
                },
                "compression": {
                  "type": "keyword"
                },
                "contentCreated": {
                  "type": "date"
                },
                "contentExpiration": {
                  "type": "date"
                },
                "contentId": {
                  "type": "keyword"
                },
                "contentUri": {
                  "type": "keyword"
                },
                "earliest": {
                  "type": "long"
                },
                "host": {
                  "type": "keyword"
                },
                "id": {
                  "type": "keyword"
                },
                "latest": {
                  "type": "long"
                },
                "offset": {
                  "type": "long"
                },
                "source": {
                  "type": "keyword"
                },
                "taskId": {
                  "type": "keyword"
                }
              }
            },
            "conf": {
              "properties": {
                "app_id": {
                  "type": "keyword"
                },
                "authHeaderExpr": {
                  "type": "keyword"
                },
                "authHeaderKey": {
                  "type": "keyword"
                },
                "authentication": {
                  "type": "keyword"
                },
                "client_secret": {
                  "type": "keyword"
                },
                "collectMethod": {
                  "type": "keyword"
                },
                "collectRequestHeaders": {
                  "properties": {
                    "name": {
                      "type": "keyword"
                    },
                    "value": {
                      "type": "keyword"
                    }
                  }
                },
                "collectRequestParams": {
                  "properties": {
                    "name": {
                      "type": "keyword"
                    },
                    "value": {
                      "type": "keyword"
                    }
                  }
                },
                "collectUrl": {
                  "type": "keyword"
                },
                "collectorId": {
                  "type": "keyword"
                },
                "content_type": {
                  "type": "keyword"
                },
                "disableTimeFilter": {
                  "type": "boolean"
                },
                "discoverToRoutes": {
                  "type": "boolean"
                },
                "discovery": {
                  "properties": {
                    "discoverType": {
                      "type": "keyword"
                    }
                  }
                },
                "earliest": {
                  "type": "keyword"
                },
                "filter": {
                  "type": "keyword"
                },
                "ingestionLag": {
                  "type": "long"
                },
                "latest": {
                  "type": "keyword"
                },
                "loginBody": {
                  "type": "text"
                },
                "loginUrl": {
                  "type": "keyword"
                },
                "metadata": {
                  "properties": {
                    "name": {
                      "type": "keyword"
                    },
                    "value": {
                      "type": "keyword"
                    }
                  }
                },
                "pagination": {
                  "properties": {
                    "attribute": {
                      "type": "keyword"
                    },
                    "maxPages": {
                      "type": "long"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "password": {
                  "type": "keyword"
                },
                "plan_type": {
                  "type": "keyword"
                },
                "tenant_id": {
                  "type": "keyword"
                },
                "timeout": {
                  "type": "long"
                },
                "tokenRespAttribute": {
                  "type": "keyword"
                },
                "useRoundRobinDns": {
                  "type": "boolean"
                },
                "username": {
                  "type": "keyword"
                }
              }
            },
            "destructive": {
              "type": "boolean"
            },
            "dir": {
              "type": "keyword"
            },
            "executor": {
              "properties": {
                "collectStep": {
                  "type": "keyword"
                },
                "collectibles": {
                  "properties": {
                    "collectorId": {
                      "type": "keyword"
                    },
                    "compression": {
                      "type": "keyword"
                    },
                    "contentCreated": {
                      "type": "date"
                    },
                    "contentExpiration": {
                      "type": "date"
                    },
                    "contentId": {
                      "type": "keyword"
                    },
                    "contentTime": {
                      "type": "float"
                    },
                    "contentType": {
                      "type": "keyword"
                    },
                    "contentUri": {
                      "type": "keyword"
                    },
                    "earliest": {
                      "type": "long"
                    },
                    "guid": {
                      "type": "keyword"
                    },
                    "host": {
                      "type": "keyword"
                    },
                    "id": {
                      "type": "keyword"
                    },
                    "latest": {
                      "type": "long"
                    },
                    "source": {
                      "type": "keyword"
                    },
                    "taskId": {
                      "type": "keyword"
                    }
                  }
                },
                "heartbeatPeriod": {
                  "type": "long"
                },
                "input": {
                  "properties": {
                    "breakerRulesets": {
                      "type": "keyword"
                    },
                    "filter": {
                      "type": "keyword"
                    },
                    "id": {
                      "type": "keyword"
                    },
                    "sendToRoutes": {
                      "type": "boolean"
                    },
                    "staleChannelFlushMs": {
                      "type": "long"
                    },
                    "type": {
                      "type": "keyword"
                    }
                  }
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "jobId": {
              "type": "keyword"
            },
            "keep": {
              "type": "boolean"
            },
            "logLevel": {
              "type": "keyword"
            },
            "reschedule": {
              "type": "long"
            },
            "task": {
              "properties": {
                "conf": {
                  "properties": {
                    "app_id": {
                      "type": "keyword"
                    },
                    "client_secret": {
                      "type": "keyword"
                    },
                    "collectorId": {
                      "type": "keyword"
                    },
                    "content_type": {
                      "type": "keyword"
                    },
                    "discoverToRoutes": {
                      "type": "boolean"
                    },
                    "earliest": {
                      "type": "keyword"
                    },
                    "filter": {
                      "type": "keyword"
                    },
                    "ingestionLag": {
                      "type": "long"
                    },
                    "latest": {
                      "type": "keyword"
                    },
                    "plan_type": {
                      "type": "keyword"
                    },
                    "tenant_id": {
                      "type": "keyword"
                    },
                    "timeout": {
                      "type": "long"
                    }
                  }
                },
                "destructive": {
                  "type": "boolean"
                },
                "type": {
                  "type": "keyword"
                }
              }
            },
            "taskId": {
              "type": "keyword"
            },
            "type": {
              "type": "keyword"
            }
          }
        },
        "taskId": {
          "type": "keyword"
        },
        "tasksCompleted": {
          "type": "long"
        },
        "tasksStarted": {
          "type": "long"
        },
        "tcpPort": {
          "type": "long"
        },
        "tcpRouting": {
          "type": "keyword"
        },
        "tenantId": {
          "type": "keyword"
        },
        "text": {
          "type": "keyword"
        },
        "textSecret": {
          "type": "keyword"
        },
        "throttleRatePerSec": {
          "type": "keyword"
        },
        "time": {
          "type": "date"
        },
        "timeout": {
          "type": "long"
        },
        "timeoutSec": {
          "type": "long"
        },
        "timestamp": {
          "type": "long"
        },
        "timestampAnchorRegex": {
          "type": "keyword"
        },
        "timestampEarliest": {
          "type": "keyword"
        },
        "title": {
          "type": "keyword"
        },
        "tls": {
          "properties": {
            "caPath": {
              "type": "keyword"
            },
            "certPath": {
              "type": "keyword"
            },
            "certificateName": {
              "type": "keyword"
            },
            "commonNameRegex": {
              "type": "keyword"
            },
            "disabled": {
              "type": "boolean"
            },
            "enabled": {
              "type": "boolean"
            },
            "maxVersion": {
              "type": "keyword"
            },
            "minVersion": {
              "type": "keyword"
            },
            "ocspCheck": {
              "type": "boolean"
            },
            "passphrase": {
              "type": "keyword"
            },
            "privKeyPath": {
              "type": "keyword"
            },
            "rejectUnauthorized": {
              "type": "boolean"
            },
            "requestCert": {
              "type": "boolean"
            }
          }
        },
        "toCommit": {
          "properties": {
            "offset": {
              "type": "keyword"
            },
            "partition": {
              "type": "keyword"
            },
            "topic": {
              "type": "keyword"
            }
          }
        },
        "toDelete": {
          "type": "long"
        },
        "toDo": {
          "properties": {
            "author": {
              "type": "keyword"
            },
            "description": {
              "type": "text"
            },
            "displayName": {
              "type": "keyword"
            },
            "id": {
              "type": "keyword"
            },
            "tags": {
              "properties": {
                "streamtags": {
                  "type": "keyword"
                }
              }
            },
            "version": {
              "type": "keyword"
            }
          }
        },
        "token": {
          "type": "keyword"
        },
        "tokenTTLMinutes": {
          "type": "long"
        },
        "tokenUrl": {
          "type": "keyword"
        },
        "topic": {
          "type": "keyword"
        },
        "topicPartitions": {
          "properties": {
            "numPartitions": {
              "type": "long"
            },
            "topic": {
              "type": "keyword"
            }
          }
        },
        "topics": {
          "type": "text"
        },
        "total": {
          "type": "long"
        },
        "totalBytes": {
          "type": "long"
        },
        "totalEvents": {
          "type": "long"
        },
        "totalGiB": {
          "type": "keyword"
        },
        "totalMessages": {
          "type": "long"
        },
        "totalSize": {
          "type": "long"
        },
        "totalSkips": {
          "type": "long"
        },
        "trigger": {
          "type": "keyword"
        },
        "type": {
          "type": "keyword"
        },
        "udpPort": {
          "type": "long"
        },
        "updated": {
          "type": "long"
        },
        "url": {
          "type": "keyword"
        },
        "urls": {
          "properties": {
            "url": {
              "type": "keyword"
            },
            "weight": {
              "type": "long"
            }
          }
        },
        "useAck": {
          "type": "boolean"
        },
        "useFwdTimezone": {
          "type": "boolean"
        },
        "useRoundRobinDns": {
          "type": "boolean"
        },
        "useToken": {
          "type": "boolean"
        },
        "user": {
          "type": "keyword"
        },
        "user_agent": {
          "properties": {
            "name": {
              "type": "keyword"
            }
          }
        },
        "username": {
          "type": "keyword"
        },
        "version": {
          "type": "keyword"
        },
        "waitPeriod": {
          "type": "long"
        },
        "warning": {
          "properties": {
            "count": {
              "type": "long"
            },
            "message": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            },
            "name": {
              "type": "keyword"
            },
            "stack": {
              "type": "text"
            },
            "type": {
              "type": "keyword"
            }
          }
        },
        "weight": {
          "type": "long"
        },
        "worker": {
          "type": "keyword"
        },
        "workerCount": {
          "type": "long"
        },
        "workerId": {
          "type": "keyword"
        },
        "writeTimeout": {
          "type": "long"
        }
      }
    },
    "settings": {
      "index": {
        "mapping": {
          "total_fields": {
            "limit": 1300
          }
        }
      }
    }
  }
}
```

</details>

<details><summary>4. Index Templates</summary>

**Metrics:**

```
PUT _index_template/metrics-cribl-internal
{
  "index_patterns": [
    "metrics-cribl-internal"
  ],
  "composed_of": [
    "metrics-cribl-internal"
  ]
  "data_stream": { },
  "template": {
    "settings": {
      "index.mode": "time_series",
      "index.routing_path": ["host.name", "cribl_wp", "_metric"]
    }
  }
}
```

**Logs:**

```
PUT _index_template/logs-cribl-internal
{
  "index_patterns": [
    "logs-cribl-internal"
  ],
  "composed_of": [
    "logs-cribl-internal"
  ],
  "data_stream": {}
}
```

</details>

#### 2. Cribl Stream Configuration

Repeat the following instructions for every Worker Group in your distributed environment:

1. Import the pipeline to remove mapping conflicts:
  - Create a new pipeline with the name `cribl-internal_rm_conflicts` and wait to save it
  - Open the Advanced JSON editor.
  - Copy and paste the following JSON and then save the pipeline:

<details><summary>Pipeline cribl-internal_rm_conflicts</summary>

```

```
</details>

2. Create a new [Elasticsearch Destination](https://docs.cribl.io/stream/destinations-elastic/#configuring-product-to-output-to-elasticsearch) named `cribl_elasticsearch`:
  - Set the appropriate Bulk API URL or Cloud ID
  - Set the appropriate authentication settings
  - Set the DataStream to ``metrics-cribl-internal``
  - Configure backpressure behaviour to be persistent queueing

3. Configure the [Cribl Internal Metrics](https://docs.cribl.io/stream/sources-cribl-internal/#configuring) Source and from the source configuration page:
  - Navigate to **Processing Settings > Fields**
    - Set an `__index` field with value ``metrics-cribl-internal``
    - (Optional) Add a `custom_id` field with a custom value, for example a data centre name
  - Navigate to **Connected Destinations**
    - Set the *Pipeline/Pack* to `cribl_metrics_rollup`
    - Set the *Destination* to `elastic:cribl_elasticsearch`
  - Enable the source

4. Configure the [Cribl Internal Logs](https://docs.cribl.io/stream/internal-logs/) Source and from the source configuration page:
  - Navigate to **Processing Settings > Fields**
    - Set an `__index` field with value ``logs-cribl-internal``
    - (Optional) Add a `custom_id` field with a custom value, for example a data centre name
  - Navigate to **Connected Destinations**
    - Set the *Pipeline/Pack* to `cribl-internal_rm_conflicts`
    - Set the *Destination* to `elastic:cribl_elasticsearch`
  - Enable the source

5. Commit & deploy if your Stream is in a distributed environment

</details>

## Importing Saved Objects

Dashboards as well as Rules can be imported by uploading the `ECM_saved_objects.ndjson` file to Kibana.

They can be imported through the [Managed Saved Objects Interface](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html#_import).

Note that the setting [`xpack.encryptedSavedObjects.encryptionKey`](https://www.elastic.co/guide/en/kibana/current/xpack-security-secure-saved-objects.html) may need to be set.

## Optional Configuration

#### Leader logs:

Leader logs are currently NOT sent via the Internal Logs source, so for the leader dashboard to work, additional steps need to be performed so leader logs are forwarded to the Elasticsearch destination as well:

1. Install a *Cribl Edge* node on your *Leader* node and configure local log collection via a File Monitor source. An easy way to do this would be by using [the bootstrap method described here](https://docs.cribl.io/stream/deploy-workers/#add-bootstrap).

| :information_source: When deploying the edge node to your leader node, we recommend having a separate fleet just for this node. Be sure to disable all other inputs on that edge node except for file monitor inputs. |
|----------------------------------------------|

2. Configure the File Monitor source to collect logs by setting the Filename Allowlist modal to `/opt/cribl/log/*.log`
3. Then you’ll also need to import the `cribl-internal_rm_conflicts` pipeline from step 2.1 above into the edge environment.
4. And configure the File Monitor Source like the Cribl Internal Logs source in step 2.4.

## Additional Considerations

#### Mappings:

The existing field mapping for logs is set to static and this has been deliberately the chosen for the following reasons:

- prevent excessive mapping conflicts and explosions
- to keep in line with shard sizing across versions
- to have a solid set of known fields as a starting point to choose from

The mappings specified in the component templates may need further adjustments going forward. This is because the list of existing fields will continue to grow as more features are either enabled or added.

#### General:

1. Due to the static mapping on the logs data streams, the size of the created Logs indices can grow quickly.
    - Consider adjusting the index templates and add an Index Lifecycle Management (ILM) policy with a rollover action
2. Some mapping conflicts could still arise for specific unaccounted for logs
    - The index mapping or pre-processing pipeline attached to the Elasticsearch destination may have to be continuously updated
3. If some interesting fields cannot be searched or aggregated
    - You can add the field to the component template in the index template. You may have to reindex the existing index and/or do a rollover to have new data come in with the updated mapping.

4. The metrics index uses TSDS under the hood, so you can use Elasticsearch’s [Downsample](https://www.elastic.co/guide/en/elasticsearch/reference/current/downsampling.html) ILM action, to reduce storage over time as metrics become less relevant.

#### Available Dashboards

| Name | Description |
| ------------- | ------------- |
| Cribl Logs - Home  | Starting point for troubleshooting issues in a Cribl environment  |
| Cribl Logs - Stats | Minutely Stats created from Internal Logs |
| Cribl Logs - Stats by Worker Process | Minutely Stats created from Internal Logs but split by Worker Process |
| Cribl Leader - Overview | Overview of important leader related Statistics and metrics created from the logs of the leader node |
| Cribl Metrics - Overview | Dashboard based on general metrics on a host level |
| Cribl Metrics - Worker Processes | The overview of workers broken down by worker process |
| Cribl Metrics - Source & Destination | The overview of incoming and outgoing data broken down by source and destination |

#### Available Rules

| Name | Description |
| ------------- | ------------- |
| Thruput Threshold | 200GB Total Thruput per worker process per day exceeded |
| Memory Threshold | 2GB Memory Usage per worker process exceeded in the last 5 minutes |
| CPU Threshold | CPU Utilization by Worker Process averages above 95% in the last 5 minutes |
| PQ Threshold | Persistent Queue size is engaged or exceeded threshold in the last 5 minutes |
| Source Unhealthy | Destination has been marked as unhealthy in the last 5 minutes |
| Destination Unhealthy | Destination has been marked as unhealthy in the last 5 minutes |

## Searching Leader Logs

You can use the Discover App to find leader logs. The included pipeline splits the logs by `sourcetype` field, allowing you to quickly search for a specific type of log:

| `sourcetype` value  | Description |
| ------------- | ------------- |
| cribl:access    | API calls |
| cribl:audit     | Actions pertaining to files |
| cribl:log       | Principal logs |
| cribl:notifications | Messages that appear in the notification list |
| cribl:stderr    | Garbage collections and OOM killers |
| cribl:ui-access | Interactions with different UI components |

## Navigating the available Dashboards

Start by applying the available controls. Every Dashboard can have its own set of Controls. Controls are filters to help narrow down the searches that are sent to the Elasticsearch infrastructure. These are filters to help let you set context on the dashboards. That context can then be carried over to the other dashboards as well by pinning filters.

#### Controls

| :information_source: Narrowing down the time range can help prevent infrastructure overhead. |
|----------------------------------------------|

- The available values in the controls depend on the data which has been found in the time range.
- Every dashboard will have different controls based on the underlying searched data
- You can easily switch dashboards while keeping context with pinned filters

#### Drilldowns

Most dashboards included here contain drill downs, allowing you to go into more details.

For example: 

Visualizations on the `Cribl Metrics - Overview` dashboard, allows you to drilldown into the `Cribl Metrics - Worker Processes` dashboard after observing that one of the workers shows signs of high CPU utilization.
