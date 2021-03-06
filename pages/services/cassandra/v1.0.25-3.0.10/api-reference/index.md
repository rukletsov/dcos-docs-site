---
layout: layout.pug
navigationTitle:  API Reference
title: API Reference
menuWeight: 80
excerpt:
featureMaturity:
enterprise: false
---

<!-- This source repo for this topic is https://github.com/dcos-cassandra-service -->


The DC/OS Apache Cassandra Service provides a REST API that can be accessed from outside the cluster. The <dcos_url> parameter referenced below indicates the base URL of the DC/OS cluster on which the Cassandra Service is deployed. Depending on the transport layer security configuration of your deployment, this may be a HTTP or a HTTPS URL.

<a name="#rest-auth"></a>
# REST API Authentication

REST API requests must be authenticated. This authentication is only applicable for interacting with the Cassandra REST API directly. You do not need the token to access the Cassandra nodes themselves.

If you are using Enterprise DC/OS, follow these instructions to [create a service account and an authentication token](/1.10/security/ent/service-auth/custom-service-auth/). You can then configure your service to automatically refresh the authentication token when it expires. To get started more quickly, you can also [get the authentication token without a service account](/1.10/security/ent/iam-api/), but you will need to manually refresh the token.

If you are using open source DC/OS, follow these instructions to [pass your HTTP API token to the DC/OS endpoint](/1.10/security/oss/iam-api/).

Once you have the authentication token, you can store it in an environment variable and reference it in your REST API calls:

```
export auth_token=uSeR_t0k3n
```

The `curl` examples in this document assume that an auth token has been stored in an environment variable named `auth_token`.

If your DC/OS Enterprise installation requires encryption, you must also use the `ca-cert` flag when making REST calls. Refer to [Obtaining and passing the DC/OS certificate in cURL requests](/1.9/networking/tls-ssl/get-cert) for information on how to use the `--cacert` flag. [If encryption is not required](/1.9/networking/tls-ssl/), you can omit the --cacert flags.

# Configuration

## View the Installation Plan

```
curl -H "Authorization: token=$auth_token" <dcos_url>/service/cassandra/v1/plan
```

## Retrieve Connection Info

```
curl -H "Authorization: token=$auth_token" <dcos_url>/cassandra/v1/connection
```

You will see a response similar to the following:

```
{
    "nodes": [
        "10.0.0.47:9042",
        "10.0.0.50:9042",
        "10.0.0.49:9042"
    ]
}
```

This JSON array contains a list of valid nodes that the client can use to connect to the Cassandra cluster. For availability reasons, it is best to specify multiple nodes in the CQL Driver configuration used by the application.

## Pause Installation

The installation will pause after completing installation of the current node and wait for user input.

```
curl -X POST -H "Authorization: token=$auth_token" <dcos_url>/service/cassandra/v1/plan?cmd=interrupt
```

## Resume Installation

The REST API request below will resume installation at the next pending node.

```
curl -X PUT -H "Authorization: token=$auth_token" <dcos_url>/service/cassandra/v1/plan?cmd=proceed
```

# Managing

## Node Status
Retrieve the status of a node by sending a GET request to `/v1/nodes/<node-#>/status`:

```
curl -H "Authorization: token=$auth_token" <dcos_url>/service/cassandra/v1/nodes/<node-#>/status
```

## Node Info
Retrieve node information by sending a GET request to `/v1/nodes/<node-#>/info`:

```
curl -H "Authorization: token=$auth_token" <dcos_url>/service/cassandra/v1/nodes/</node-#>/info
```

## Service Status Info
Send a GET request to the `/v1/state/properties/suppressed` endpoint to learn if Cassandra is in a `suppressed` state and not receiving offers. If a service does not need offers, Mesos can "suppress" it so that other services are not starved for resources.

You can use this request to troubleshoot: if you think Cassandra should be receiving resource offers, but is not, you can use this API call to see if Cassandra is suppressed.

```
curl -H "Authorization: token=$auth_token" "<dcos_url>/service/cassandra/v1/state/properties/suppressed"
```

## Cleanup

First, create the request payload, for example, in a file `cleanup.json`:

```
{
    "nodes":["*"],
    "key_spaces":["my_keyspace"],
    "column_families":["my_cf_1", "my_cf_w"]
}
```

In the above, the nodes list indicates the nodes on which cleanup will be performed. The value [*], indicates to perform the cleanup cluster wide. key_spaces and column_families indicate the key spaces and column families on which cleanup will be performed. These may be ommitted if all key spaces and/or all column families should be targeted. The JSON below shows the request payload for a cluster wide cleanup operation of all key spaces and column families.

```
{
    "nodes":["*"]
}
```

```
curl -X PUT -H "Authorization: token=$auth_token" -H "Content-Type:application/json" <dcos_url>/service/cassandra/v1/cleanup/start --data @cleanup.json
```

## Repair

First, create the request payload, for example, in a file `repair.json`:

```
{
    "nodes":["*"],
    "key_spaces":["my_keyspace"],
    "column_families":["my_cf_1", "my_cf_w"]
}
```
In the above, the nodes list indicates the nodes on which the repair will be performed. The value [*], indicates to perform the repair cluster wide. key_spaces and column_families indicate the key spaces and column families on which repair will be performed. These may be ommitted if all key spaces and/or all column families should be targeted. The JSON below shows the request payload for a cluster wide repair operation of all key spaces and column families.

```
{
    "nodes":["*"]
}
```

```
curl -X PUT -H "Authorization: token=$auth_token" -H "Content-Type:application/json" <dcos_url>/service/cassandra/v1/repair/start --data @repair.json
```

## Backup

First, create the request payload, for example, in a file `backup.json`:

```
{
    "backup_name":"<backup-name>",
    "external_location":"s3://<bucket-name>",
    "s3_access_key":"<s3-access-key>",
    "s3_secret_key":"<s3-secret-key>"
}
```

Then, submit the request payload via `PUT` request to `/v1/backup/start`

```
curl -X PUT -H "Authorization: token=$auth_token" -H "Content-Type: application/json" -d @backup.json <dcos_url>/service/cassandra/v1/backup/start
```

## Restore

First, bring up a new instance of your Cassandra cluster with the same number of nodes as the cluster whose snapshot backup you want to restore.

Next, create the request payload, for example, in a file `restore.json`:

```
{
    "backup_name":"<backup-name-to-restore>",
    "external_location":"s3://<bucket-name-where-backups-are-stored>",
    "s3_access_key":"<s3-access-key>",
    "s3_secret_key":"<s3-secret-key>"
}
```

Next, submit the request payload via `PUT` request to `/v1/restore/start`

```
curl -X PUT -H "Authorization: token=$auth_token" -H "Content-Type: application/json" -d @restore.json <dcos_url>/service/cassandra/v1/restore/start
{"status":"started", message:""}
```

Check status of the restore:

```
curl -X -H "Authorization: token=$auth_token" <dcos_url>/service/cassandra/v1/restore/status
```
