# Triggering Data Updates
OPAL allows for other components to notify it (and through it all the OPAL clients , and their next-door policy agents) of data updates, triggering each client [subscribed to the published topic] to fetch the data it needs.

### What is this good for?
Let's try an example - say your application has a billing service, and you want to allow access only to users who have billing enabled (enforced via a policy agent).
You now need changes to the state of the billing service to be propagated to each of the enforcement points/agents (and preferably instantly [Users who've paid - don't like to wait 😅 ]). </br>
With the OPAL's data-update-triggers feature the billing-service, another service monitoring it, or even a person can trigger updates as they need - knowing OPAL will take it from there to all the points that need it.

### <a name="datasource-token"></a>First, you need to obtain a data-source identity token (JWT)

Every service that **publishes** to OPAL needs a `datasource` identity token.
Obtaining one is easy, but you need access to the corresponding OPAL Server **master token**.

Obtain a data source token with the cli:
```
opal-client obtain-token MY_MASTER_TOKEN --uri=https://opal.yourdomain.com --type datasource
```

If you don't want to use the cli, you can obtain the JWT directly from the deployed OPAL server via its REST API:
```
curl --request POST 'https://opal.yourdomain.com/token' \
--header 'Authorization: Bearer MY_MASTER_TOKEN' \
--header 'Content-Type: application/json' \
--data-raw '{
  "type": "datasource",
}'
```
The `/token` API endpoint can receive more parameters, as [documented here](https://opal.permit.io/redoc#operation/generate_new_access_token_token_post).

This example assumes that:
* You deployed OPAL server to `https://opal.yourdomain.com`
* The master token of your deployment is `MY_MASTER_TOKEN`.
  * In real life, use a cryptographically secure secret. If you followed our tutorials while deploying OPAL, you probably generated one with `opal-server generate-secret`.

## How to trigger updates
There are a few ways to trigger updates:</br>

### Option 1: Trigger a data update with the CLI
Can be run both from opal-client and opal-server.

Example:
  - With `$token` being a JWT we generated in [step 1](#datasource-token).
  - we publish a data-event regarding two topics `users` and `billing` pointing clients to `http://mybillingserver.com/users` to obtain the data they need. we also provide the clients with the credentials they'll need to connect to the server (as HTTP authorization headers)

  -
    ```sh
    opal-client publish-data-update $token --src-url http://mybillingserver.com/users -t users -t billing --src-config '{"headers":{"authorization":"bearer secret-token"}}'
    ```
-   (Yes... We did... We put authorization in your authorization 😜 &nbsp;😅 )

- See this recording showing the command including getting the JWT for the server with the `obtain-token` command.
    <p><a href="https://asciinema.org/a/JYBzx1VrqJ17QnvmOnDYylOE6?t=1" target="_blank"><img src="https://asciinema.org/a/JYBzx1VrqJ17QnvmOnDYylOE6.svg"/></a></p>

### Option 2: Trigger a data update with OPAL Server REST API
- All the APIs in opal are OpenAPI / Swagger based (via FastAPI).
- Check out the [API docs on your running OPAL-server](http://localhost:7002/docs#/Data%20Updates/publish_data_update_event_data_config_post) -- this link assumes you have the server running on `http://localhost:7002`
- You can also [generate an API-client](https://github.com/OpenAPITools/openapi-generator) in the language of your choice using the [OpenAPI spec provided by the server](http://localhost:7002/openapi.json)

### Option 3: Write your own - import code from the OPAL's packages
- One of the great things about OPAL being written in Python is that you can easily reuse its code.
See the code for the `DataUpdate` model at [opal_common/schemas/data.py](https://github.com/permitio/opal/blob/master/opal_common/schemas/data.py) and use it within your own code to send an update to the server





