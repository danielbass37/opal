# How to configure external data sources

This document instructs you how to serve a different (dynamic) value of `OPAL_DATA_CONFIG_SOURCES` to different client - i.e: each client has their own unique configuration of DataSourceEntries.

OPAL server can **redirect to an external API** that will serve a different value based on the client identity.

The document has 3 parts:
1) General background about `OPAL_DATA_CONFIG_SOURCES`
2) External data sources - architecture and flows - explains how external data sources work
3) Real code samples of an External Data Source API server in python

## General background about `OPAL_DATA_CONFIG_SOURCES`
OPAL clients have two **distinct** types of data sources:

Data Source Type | Format of data | Function | Source of data
--- | --- | --- | ---
**Policies (and static data)** | **Policy bundle** - data format is very similar to OPA native bundles. | Each bundle contains the policies (rego) and static data that is usually included as part of the policy. By "static", we mean the data rarely changes, same as declaring the data *inside* the rego file. | Git repository (we are thinking about expanding this to more sources in the near future).
**Dynamic Data** | **DataSourceEntry** - think about it as a "directive" how to fetch the data and where to put it in OPA document tree - i will give an example below. | Realtime updates about data that changes in the rate of the application (i.e: as a result of user action), for example - we invited a user to a document in google drive. If we implement this internally as the user now having a `Viewer` role on the document, we can send a data update containing the new role, with instructions to put it inside the roles document in OPA cache. | Can be anything - a database, a 3rd party API, your APIs, etc - extensible by fetch providers. We support http apis out of the box.

### Q: What are `OPAL_DATA_CONFIG_SOURCES`?
* They are **dynamic data** sources, so they are **directives** how to fetch the data and where to put it in OPA.
* `OPAL_DATA_CONFIG_SOURCES` are statically configured because they contain the list of "complete picture" data sources - which are fetched after OPAL client loads, and after every disconnect between OPAL client and OPAL server. Think of it as directives how to get the up-to-date "clean slate" data.

## External data sources - architecture and flows

### When do you need to configure an external data source?
If each of your OPAL clients needs a **slightly different** configuration, you cannot use a static config var like `OPAL_DATA_CONFIG_SOURCES` to store your configuration. In such case, you want OPAL server to return a different `DataSourceEntry` configuration **based on the identity** of the OPAL client asking for the config.

### Can different OPAL client have different "identities"?
**Yes!! And they should!!**

Each of your opal clients should use a unique JWT token issued to it by the OPAL server.
[You can learn how to generate your OPAL client token here](https://github.com/permitio/opal/blob/master/docs/HOWTO/get_started_with_opal_using_docker.md#step-2-obtain-client-jwt-token-optional).

When [issued via the API](https://opal.permit.io/redoc#operation/generate_new_access_token_token_post), each OPAL client token (JWT) supports custom claims (the `claims` attribute).

### How to configure an external data source?
OPAL server supports redirecting to an **external API server** that can serve different `DataSourceEntries` based on the JWT token identifying the OPAL client. This external API server should have access to OPAL's public key so that it can validate OPAL JWTs correctly and make sure the OPAL identity passed in a Bearer token is valid.

This is how a typical configuration of `OPAL_DATA_CONFIG_SOURCES` looks like: ([a breakdown of this config can be found here](https://github.com/permitio/opal/blob/master/docs/HOWTO/get_started_with_opal_using_docker.md#step-5-server-config---data-sources))
```
{
    "config": {
        "entries": [
            {
                "url": "https://api.permit.io/v1/policy-config",
                "topics": [
                    "policy_data"
                ],
                "config": {
                    "headers": {
                        "Authorization": "Bearer FAKE-SECRET"
                    }
                }
            }
        ]
    }
}
```

A `OPAL_DATA_CONFIG_SOURCES` configuration that redirects to an external data source looks slightly different:
```
{
    "external_source_url": "https://your-api.com/path/to/api/endpoint"
}
```

As you see, the config is now much simpler. **OPAL server will simply redirect the query.**

### How the redirect actually works?
Upon being asked for data source entries, OPAL server will simply redirect to the `external_source_url`, but will also concat a [url query param](https://en.wikipedia.org/wiki/Query_string) containing the OPAL CLIENT JWT.

Request by OPAL client:
```
GET https://opal-server.com/data/config
```

Response by OPAL server:
```
HTTP 307 Temporary Redirect
https://your-api.com/path/to/api/endpoint?token=<OPAL_CLIENT_JWT>
```

**This is why you should never use an `external_source_url` that is not HTTPS.**
OPAL relies on the TLS/SSL encryption so it can pass the JWT as a query param, knowing it will be encrypted.

### Actions that should be taken by the external API
The external API should do the following:
* Expose an http GET endpoint with same path as in OPAL config: i.e: `/path/to/api/endpoint`
* When hit, this endpoint should extract the OPAL client JWT from the `token` query param.
* The endpoint should validate that the token is a valid OPAL JWT (by using the OPAL public key to verify) and use the custom claims in the JWT to determine what data sources to return.
* The endpoint should return a JSON, containing the unique value (to that OPAL client) of `OPAL_DATA_CONFIG_SOURCES` (i.e: the actual Data Source Entries).

## Real code samples of an External Data Source API server in python
You can use the following code samples in your **external API server** that serves data sources.
These are taken from our fastapi python service.

### Step 0: when issuing OPAL client token use custom claims
These custom claims will identify one OPAL client from another.

For example, this is how we assign OPAL client tokens to our customers' OPAL clients:
```python
from opal_common.schemas.security import AccessTokenRequest, PeerType
from acalla.config import OPAL_SERVER_URL, OPAL_MASTER_TOKEN

...

async with aiohttp.ClientSession(headers={"Authorization": f"bearer {OPAL_MASTER_TOKEN}", **self._headers}) as session:
    token_params = AccessTokenRequest(
         type=PeerType.client,
         ttl=timedelta(days=CLIENT_TOKEN_TTL_IN_DAYS),
         claims={'permit_client_id': pdp.client_id},
    ).json()
    async with session.post(f"{OPAL_SERVER_URL}/token", data=token_params) as response:
         data: dict = await response.json()
         token = data.get("token", None)
```

This function hits opal server's `/token` endpoint (see: [API reference](https://opal.permit.io/redoc#operation/generate_new_access_token_token_post)), generates a client token (peer type == client) and adds a custom claim that identifies the `permit_client_id` - which is unique for each of our SaaS clients.

**Since OPAL JWTs are cryptographically signed - this cannot be forged.**

### Step 1: in your external api server, be able to parse JWTs from http query param

First, we have a custom class to verify OPAL JWT from HTTP **query param**.
Notice we use the opal-common pip package as a library in our python server.
You can of course write this code in any language you want - according to API spec we detailed above.

```python
from typing import Optional

from fastapi import APIRouter, Depends, status, HTTPException, Query
from opal_common.authentication.types import JWTClaims
from opal_common.authentication.deps import _JWTAuthenticator, verify_logged_in
from opal_common.authentication.verifier import Unauthorized

class RedirectJWTAuthenticator(_JWTAuthenticator):
    """
    OPAL JWT authentication via HTTP query params.
    throws 401 if a valid jwt is not provided via the `token` query param.

    (same as JWTAuthenticator, but tries to get the JWT from a query param).
    """
    def __call__(self, token: Optional[str] = Query(None)) -> JWTClaims:
        """
        called when using the verifier as a dependency with Depends()
        """
        if token is None:
            raise Unauthorized(description="Access token was not provided!")
        return verify_logged_in(self.verifier, token)
```

### Step 2: in your external api server, expose the endpoint to serve dynamic data sources
You dynamic data sources endpoints should read your custom claim and understand how to fetch the correct config based on these claims.
```python
from typing import Optional
from fastapi import APIRouter, Depends, status, HTTPException

from opal_common.authentication.types import JWTClaims
from opal_common.schemas.data import DataSourceConfig, DataSourceEntry
from opal_common.fetcher.providers.http_fetch_provider import HttpFetcherConfig
from opal_common.authentication.verifier import JWTVerifier, Unauthorized
from opal_common.config import opal_common_config
...

def init_dynamic_data_sources_router():
    """
    inits router for opal data sources.

    this router will output different data sources depending on the jwt claims
    in the provided bearer token (claim will include pdp client id).
    """
    router = APIRouter()

    verifier = JWTVerifier(
        public_key=opal_common_config.AUTH_PUBLIC_KEY,
        algorithm=opal_common_config.AUTH_JWT_ALGORITHM,
        audience=opal_common_config.AUTH_JWT_AUDIENCE,
        issuer=opal_common_config.AUTH_JWT_ISSUER,
    )
    # notice - we defined RedirectJWTAuthenticator in our previous code sample (step 1)
    authenticator = RedirectJWTAuthenticator(verifier)

    async def extract_pdp_from_jwt_claims_or_throw(db: Session, claims: JWTClaims) -> PDP:
        claim_client_id = claims.get('permit_client_id', None)
        if claim_client_id is None:
            raise Unauthorized(description="provided JWT does not have an permit_client_id claim!")

        return await PDP.from_id_or_throw_401(db, claim_client_id)

    @router.get('/data/config', response_model=DataSourceConfig)
    async def get_opal_data_sources(db: Session = Depends(get_db), claims: JWTClaims = Depends(authenticator)):
        # at this point the jwt is valid, but we still have to extract a valid project from its claims
        pdp: PDP = await extract_pdp_from_jwt_claims_or_throw(db, claims)
        topics = [f"policy_data/{pdp.client_id}"]
        return DataSourceConfig(
            entries=[
                DataSourceEntry(
                    url=f"{BACKEND_PUBLIC_URL}/v1/policy/config",
                    topics=topics,
                    config=HttpFetcherConfig(
                        headers={'Authorization': f"Bearer {pdp.client_secret}"}
                    )
                )
            ],
        )

    return router
```

Notice how our endpoint simply returns `opal_common.schemas.data.DataSourceConfig`, specific to each client. In our case all data is currently piped from our API - but each client has a unique client secret to identify it in our systems.