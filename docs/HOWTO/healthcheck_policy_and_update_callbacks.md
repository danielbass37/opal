# How to use data update callbacks and OPA healthcheck policy
This document explains how to use two features that are separate yet closely related: [OPA healthcheck policy](#healthcheck) and [Data update callbacks](#callbacks).

## Working example configuration:
You can run the example docker compose configuration [found here](https://github.com/permitio/opal/blob/master/docker/docker-compose-with-callbacks.yml) to run OPAL with callbacks and healthcheck policy already configured correctly.

Run this one command on your machine:
```
curl -L https://raw.githubusercontent.com/authorizon/opal/master/docker/docker-compose-with-callbacks.yml \
> docker-compose.yml && docker-compose up
```

## <a name="healthcheck"></a> OPA healthcheck policy

#### What is the healthcheck policy?
A special OPA policy that (if activated) is loaded into OPA as the `system.opal` rego package.

This special policy can be used to make sure that OPA is ready to accept authorization queries, and than its state is not out of sync due to failed data updates.

#### What is the healthcheck policy good for?
You can use this policy as a **healthcheck for kubernetes** or any similar deployment, before shifting traffic into the new version of the opal-client container (i.e: this container contains the OPA agent) from an older deployment.

#### How can i activate the OPA healthcheck feature?
Set the following env var:
```
OPAL_OPA_HEALTH_CHECK_POLICY_ENABLED=True
```
You can check out a complete docker-compose configuration that uses this feature [here](https://github.com/permitio/opal/blob/master/docker/docker-compose-with-callbacks.yml).

#### How to query the healthcheck policy?
The healthcheck policy defines two main OPA rules:
* `ready`, checks that:
  * policy was synced correctly at least once
  * **and** all the initial data sources (defined by `OPAL_DATA_CONFIG_SOURCES`) were synced correctly as well.
* `healthy`, checks that:
  * OPA is `ready` (as defined above)
  * **and** latest policy update bundle synced correctly
  * **and** last published data update was fetched and synced correctly

You can query the `ready` rule like so:
```
curl --request POST 'http://localhost:8181/v1/data/system/opal/ready'
```

expected output format:
```json
{
    "result": true
}
```

You can query the `healthy` rule like so (same output format):
```
curl --request POST 'http://localhost:8181/v1/data/system/opal/healthy'
```

You can also query the entire document (contains latest policy git hash and last successful data update id):

```
curl --request GET http://localhost:8181/v1/data/system/opal
```

You'll get something like this as output:
```json
{
    "result": {
        "healthy": true,
        "last_data_transaction": {
            "actions": [
                "set_policy_data"
            ],
            "error": "",
            "id": "476f290d23964099b5ddd8f10e46873d",
            "success": true
        },
        "last_policy_transaction": {
            "actions": [
                "set_policies"
            ],
            "error": "",
            "id": "0e45f42f3d7da9b343f5c199934b4bf89a9cacbd",
            "success": true
        },
        "ready": true
    }
}
```

#### Advanced: How does the healthcheck feature work?
Please note: you don't need to understand this section to use the healthcheck policy. It goes into internal implementation of the feature, to the benefit of the interested reader.

OPAL has an internal OpaClient class (code [here](https://github.com/permitio/opal/blob/master/opal_client/policy_store/opa_client.py#L140)) that is used to communicate with the OPA agent via its [REST API](https://www.openpolicyagent.org/docs/latest/rest-api/). The `OpaClient` class holds a `OpaTransactionLogState` object (code [here](https://github.com/permitio/opal/blob/master/opal_client/policy_store/opa_client.py#L58)) that represents (a **very** simplified version of) the state of synchronization between OPAL client and OPA.

A transaction is initialized in the code using context managers:
```python
async with policy_store.transaction_context(update.id) as store_transaction:
  # do whatever with policy_store, example below:
  await store_transaction.set_policy_data(policy_data, path=policy_store_path)
```

Every time a transaction [is ended](https://github.com/permitio/opal/blob/master/opal_client/policy_store/base_policy_store_client.py#L116) it is saved into OPA, by rendering the state of `OpaTransactionLogState` using the [healthcheck policy template](https://github.com/permitio/opal/blob/master/opal_client/opa/healthcheck/opal.rego).

## <a name="callbacks"></a> Data update callbacks

#### What is the update callback feature?
This feature, if activated, will trigger a callback (HTTP call to a configurable url) after every successful [data update](https://github.com/permitio/opal/blob/master/docs/HOWTO/trigger_data_updates.md). It allows you to track which data updates completed successfully and were correctly saved to OPA cache.

#### When should I use update callbacks?
If you are using OPAL to sync your policy agents, you typically have a service (let's call it the **update source** service) that pushes updates via OPAL server, and OPAL server propagates this update to your fleet of agents.

If you want your **update source** service to know that an update was successful (i.e: to resend if failed, to know when you submit bad configuration, etc), you should configure update callbacks.

#### How can I activate the update callback feature?
Set the following env var to turn on the feature:
```
OPAL_SHOULD_REPORT_ON_DATA_UPDATES=True
```

Set a default callback (will be called after each successful data update):
```
OPAL_DEFAULT_UPDATE_CALLBACKS={"callbacks":["http://opal_server:7002/data/callback_report"]}
```
You can check out a complete docker-compose configuration that uses this feature [here](https://github.com/permitio/opal/blob/master/docker/docker-compose-with-callbacks.yml).

#### What are the values I can set inside `callbacks`?

As you see, the `callbacks` key is a list; you may define more than one callback url.

Each item in the list can either be a url, or a tuple.

If the item is a url, the configuration defined in `OPAL_DEFAULT_UPDATE_CALLBACK_CONFIG` will be used. By default:
* The HTTP `POST` method will be used.
* The following headers will be used: `{"content-type": "application/json"}`.

You may pass a (url, HttpFetcherConfig) tuple instead of a url (i.e: if your callback needs special headers, bearer token, etc.)

For example, you may set a default callback (will be called after each successful data update) that has special headers like this:
```
OPAL_DEFAULT_UPDATE_CALLBACKS={"callbacks":[("http://opal_server:7002/data/callback_report",{"headers":{"X-My-Token":"token"}})]}
```

#### If my update was successful, what is the expected log output?
After triggering an update via the API, your OPAL server log will look something like this:

```
opal_server.data.data_update_publisher  | INFO  | [10] Publishing data update to topics: ['policy_data'], reason: , entries: [('https://api.country.is/23.54.6.78', 'PUT', '/users/bob/location')]
uvicorn.protocols.http.httptools_impl   | INFO  | 172.27.0.1:63456 - "POST /data/config HTTP/1.1" 200
fastapi_websocket_pubsub.event_notifier | INFO  | calling subscription callbacks for sub_id=0d949d8473824c8280a3ff6ab9146cd0 with topic=policy_data
fastapi_websocket_pubsub.event_broadc...| INFO  | Broadcasting incoming event
fastapi_websocket_pubsub.event_notifier | INFO  | calling subscription callbacks for sub_id=ee10bc3da76444e899c58b861b0079c2 with topic=policy_data
```

OPAL client will receive the update and will call the callback url (last log line):

```
opal_client.data.rpc                    | INFO  | Received notification of event: policy_data
opal_client.data.updater                | INFO  | Updating policy data, reason:
opal_client.data.updater                | INFO  | Triggering data update with id: c83b6862aa354d338d1a9e23794e3efc
opal_client.data.updater                | INFO  | Fetching policy data
opal_client.data.fetcher                | INFO  | Fetching data from url: https://api.country.is/23.54.6.78
opal_client.data.updater                | INFO  | Saving fetched data to policy-store: source url='https://api.country.is/23.54.6.78', destination path='/users/bob/location'
opal_client.policy_store.opa_client     | INFO  | processing store transaction: {'id': 'c83b6862aa354d338d1a9e23794e3efc', 'actions': ['set_policy_data'], 'success': True, 'error': ''}
opal_client.policy_store.opa_client     | INFO  | persisting health check policy: ready=true, healthy=true
opal_client.data.updater                | INFO  | Reporting the update to requested callbacks
opal_client.data.fetcher                | INFO  | Fetching data from url: http://opal_server:7002/data/callback_report
```

The called-back server will then receive the update:
(in the example you see here, the OPAL server is the one receiving the callback payload, as was configured in the [example config](https://github.com/permitio/opal/blob/master/docker/docker-compose-with-callbacks.yml).)

```
opal_server.data.api                    | INFO  | Recieved update report: {'update_id': 'c83b6862aa354d338d1a9e23794e3efc', 'reports': [{'entry': {'url': 'https://api.country.is/23.54.6.78', 'config': {}, 'topics': ['policy_data'], 'dst_path': '/users/bob/location', 'save_method': 'PUT'}, 'fetched': True, 'saved': True, 'hash': '3eb2f338beb6691bbeeeca60dc3f4afad74ec8c5881f8abe3aa23d57ffa48424'}]}
uvicorn.protocols.http.httptools_impl   | INFO  | 172.27.0.4:49720 - "POST /data/callback_report HTTP/1.1" 200
```

#### Setting up a one-time callback in the update message
When [triggering an update](https://github.com/permitio/opal/blob/master/docs/HOWTO/trigger_data_updates.md) using the OPAL server REST API, you can pass a callback definition inside the update message, like this:

Assuming your opal server is deployed at `http://my-opal-server.com:7002`, you will send a POST request to the `/data/config` route:

```
POST http://my-opal-server.com:7002/data/config
```

You will need to pass the following data in the HTTP POST request body:
```json
{
    "entries": [
        ...
    ],
    "callback": {
      "callbacks": [
        [
          "http://opal_server:7002/data/callback_report",
        ]
      ]
    }
}
```