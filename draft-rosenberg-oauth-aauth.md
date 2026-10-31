---
title: "AAuth - Agentic Authorization OAuth 2.1 Extension"
abbrev: "AAuth"
category: std

docname: draft-rosenberg-oauth-aauth-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - next generation
 - unicorn
 - AI-native
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "jdrosen/draft-rosenberg-oauth-aauth"
  latest: "https://jdrosen.github.io/draft-rosenberg-oauth-aauth/draft-rosenberg-oauth-aauth.html"

author:
 -
    fullname: "Jonathan Rosenberg"
    organization: jdrosen.net
    email: "jdrosen@jdrosen.net"
 -
    fullname: "Pat White"
    organization: Bitwave
    email: "pat.white@bitwave.io"

normative:
  RFC5646:
  RFC6455:
  RFC6585:
  RFC6749:
  RFC6750:
  RFC7515:
  RFC7517:
  RFC7519:
  RFC8259:
  RFC8414:
  RFC8628:
  RFC8725:
  RFC9110:
  RFC9728:
  HTML:
    title: "HTML Living Standard - Server-sent events"
    target: "https://html.spec.whatwg.org/multipage/server-sent-events.html"
    author:
      org: "WHATWG"
  I-D.bhutton-json-schema:

informative:
  I-D.ietf-oauth-v2-1:
  RFC4647:
  RFC7591:
  RFC7662:
  RFC8693:
  RFC8705:
  RFC9068:
  RFC9449:
  RFC9470:
  RFC9700:
  OIDC:
    title: "OpenID Connect Core 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-core-1_0.html"
    author:
      -
        ins: N. Sakimura
      -
        ins: J. Bradley
      -
        ins: M. Jones
      -
        ins: B. de Medeiros
      -
        ins: C. Mortimore
    date: December 2023


--- abstract

AI Agents are increasingly accessing web resources on behalf of the users they serve. Traditionally these agents are granted normal OAuth access tokens through existing OAuth grant flows. However, these flows were designed in the era of traditional software, and when applied to AI Agents, multiple problems have arisen. This document outlines some of the problems, and then proposes a simple OAuth2.1 extension that enables AI Agents to obtain access tokens at time of resource access using Human-In-the-Loop or other approval flows. We call this the Agent Authorization Grant (aauth) grant flow. This document also compares the proposed solution with another, non-backwards compatible aauth solution.



--- middle

# Introduction {#intro}

OAuth 2.0 {{RFC6749}}, and its consolidation as OAuth 2.1 {{I-D.ietf-oauth-v2-1}}, provides a framework for delegated access via bearer tokens. It has been wildly successful, with widespread deployment across the Internet. It was built to solve a key problem - how to allow software applications to access resources on a user's behalf, without requiring the user to give the application their username and password or other credentials.

The key assumption behind OAuth is that software applications know in advance what kinds of access they require. They can therefore request scopes - which are an umbrella set of permissions - associated with that acess. Ther user, presented with that set of scopes, can approve (or reject), resulting in the delivery of an access token to the software application. Once delivered, then the software application can make use of it to access the resource.

AI Agents - like traditional software applications - also need to invoke APIs on behalf of resource owners, and most implementations make use of OAuth grant flows to enable those AI agents to gain access tokens. However - key differences between AI Agents and traditional software applications have introduced several problems:

1. It is not known what APIs an AI Agent might need to access ahead of time. AI Agents make decisions adaptively, based on the needs of the user. This can be solved by granting wide token scopes that grant all access. However, this introduces a security problem.

2. AI Agents can make mistakes. As a result, giving them broadly scoped tokens could potentially lead to disastrous results.

There is currently no easy way to solve these problems jointly.

This document proposes a simple solution to the problem, in the form of a new OAuth grant flow - which we call the Agent Authorization Grant (aauth) Flow, and a new well-defined scope.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology


AI Agent:
: A piece of software that acts as the OAuth client, and operates in an agentic nature, requiring usage of the grant flow defined in this specification. Like all other clients, it has a stable, long-lived identity (`client_id`).

Authorization Server (AS):
: Validates Scope Challenges, issues `request_code` handles, obtains the resource owner's approval, and issues access tokens.

Resource Server (RS):
: API server that protects resources via OAuth tokens. It issues Scope Challenges when a request exceeds the minimal AI Agent scope, and publishes its signing keys and scope descriptions through OAuth Protected Resource Metadata {{RFC9728}}.

Scope Challenge:
: A JWT, signed by the RS, returned to the AI Agent when a request exceeds the minimal AI Agent scope. It carries the scopes required to perform the operation and the reason they are required. Defined in {{scope-challenge}}.

request_code:
: Opaque handle returned by the AS to correlate an agent's request with subsequent token retrieval.

reason:
: Human-readable explanation, generated by the RS and carried in the Scope Challenge, shown to the user during approval.

Scope Descriptions Document:
: A JSON document published by the RS that provides human-readable, localizable descriptions of each scope it supports. Defined in {{scope-descriptions}}.


# The AAuth Grant Flow

This flow is modeled on the OAuth 2.0 Device Authorization Grant {{RFC8628}} but is tailored for confidential agent clients with long-lived identities, and optimized for performant delivery of tokens to the agent.

The flow executes through a series of steps.


## Step 1: Requesting the Minimal Scope AI Agent Token {#step1}

In the first step, the AI Agent requests an access token with very limited scope. This scope has a very specific meaning in the context of this specification. It refers to the set of access that is permitted by an AI Agent which is completely safe and should not require the resource owner - or software or AI agents operating on their behalf -  to explicitly approve. The precise set of API calls that fit under this umbrella are entirely a matter of local implementation in the resource server.

We call this scope the "Minimal AI Agent scope". This specification standardizes a well-known value for this scope - "ai-agent-minimal". This is similar in concept to OIDC which defines a small set of well-defined scopes {{OIDC}}.

AI Agents gain access tokens associated with this scope through any of the existing OAuth2.0 grant flows.

For the authorization code grant flow, it is RECOMMENDED that Authorization Servers render information to the resource owner, informing them that this is an AI Agent, and they may be required to perform additional approvals to when the AI Agent does something that has some risk. It is RECOMMENDED that examples are provided on what type of access is considered risky. For example, a travel website may allow an AI Agent to list flights, or enumerate airports. However, listing of a user's travel plans or making flight changes, is considered risky.

In all ways, the access token obtained with this scope is a normal access token and the AI Agent can use it in the same way it uses any other access token.

## Step 2: Access the Resource {#step2}

When the AI Agent accesses a resource on the resource server using the minimal AI Agent scope, if the minimal scope associated with the token is sufficient, the resource server will accept the request and all is well.

However, if the AI Agent attempts to perform an action which the resource server determines is beyond the minimal AI Agent scope, it rejects the request with a 403 (Forbidden) response carrying an `insufficient_scope` error code in the `WWW-Authenticate` header, as defined in {{Section 3 of RFC6750}}. Alongside the `scope` auth-param defined by {{RFC6750}}, the RS includes a new auth-param defined by this specification, `scope_challenge`. Its value is a Scope Challenge ({{scope-challenge}}): a JWT, signed by the RS, that carries both the set of scopes required to perform the operation and a `reason` - a natural language explanation of why the minimal AI Agent scope was insufficient for this particular operation. Because the challenge is signed, neither the AI Agent nor any other party between the RS and the AS can alter the scopes or the reason that will ultimately be shown to the user.

~~~
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
  scope="urn:example:reservations.write",
  resource_metadata="https://api.example.com/.well-known/
    oauth-protected-resource",
  scope_challenge="eyJhbGciOiJFUzI1NiIsInR5cCI6InNjb3BlLWNoYWxsZW5
    nZStqd3QiLCJraWQiOiJyczEifQ.eyJpc3MiOiJodHRwczovL2FwaS5leGFtcG
    xlLmNvbSIsImF1ZCI6Imh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbSIsImlhdCI6
    MTc5MzQ2OTYwMCwiZXhwIjoxNzkzNDY5OTAwLCJqdGkiOiI5ZjFjMmUzYS00Yj
    VkLTRlNmYtOGE5Yi0wYzFkMmUzZjRhNWIiLCJjbGllbnRfaWQiOiJyZXNlcnZl
    Ym90Iiwic3ViIjoiMjQ4Mjg5NzYxMDAxIiwic2NvcGUiOiJ1cm46ZXhhbXBsZT
    pyZXNlcnZhdGlvbnMud3JpdGUiLCJyZWFzb24iOiJSZXNlcnZlQm90IHdhbnRz
    IHRvIGJvb2sgYSB0YWJsZSBmb3IgdHdvIGF0IEpvZSdzIEl0YWxpYW4gb24gRn
    JpZGF5IGF0IDlwbS4ifQ.DtsKx1Q...2Qw"
~~~
{: title="Resource server response carrying a Scope Challenge (line breaks within header values added for readability)"}

The RS MUST issue a Scope Challenge only in response to a request that presented a valid access token. A request with a missing, expired, revoked, or otherwise invalid token is rejected with the `invalid_token` error as usual, and no Scope Challenge is issued. The RS SHOULD also include the `resource_metadata` auth-param defined in {{Section 5.1 of RFC9728}}, so that the AI Agent and the AS can locate the RS's metadata ({{rs-metadata}}).

This exchange is similar in structure to the step-up authentication challenge of {{RFC9470}}, in which an RS tells a client that a token obtained with stronger authentication is required. Here, the RS tells the client that additional scopes - and the resource owner's approval - are required, and says why.

### The Scope Challenge {#scope-challenge}

A Scope Challenge is a JSON Web Token {{RFC7519}} signed by the resource server as a JWS {{RFC7515}}, using a key published in the JWK Set {{RFC7517}} referenced by the `jwks_uri` member of its Protected Resource Metadata {{RFC9728}}.

The JOSE header MUST include the following parameters:

* `typ`: MUST be `scope-challenge+jwt`, as recommended by {{Section 3.11 of RFC8725}}.
* `alg`: MUST be an asymmetric digital signature algorithm. The value `none` and MAC-based algorithms MUST NOT be used. The RS SHOULD list the algorithms it uses in the `resource_signing_alg_values_supported` member of its metadata.
* `kid`: SHOULD be present, and identify the signing key within the RS's JWK Set.

The following claims are defined:

`iss`:
: REQUIRED. The resource identifier of the RS, as defined in {{RFC9728}}. The AS uses this value to locate the RS's Protected Resource Metadata and, through it, the RS's signing keys.

`aud`:
: REQUIRED. The issuer identifier {{RFC8414}} of the AS that is expected to process this challenge.

`iat`:
: REQUIRED. The time at which the challenge was issued.

`exp`:
: REQUIRED. The time after which the challenge MUST NOT be accepted. The lifetime SHOULD be short - five minutes is RECOMMENDED - since it bounds only the time the AI Agent has to present the challenge to the AS, not the time available for approval.

`jti`:
: REQUIRED. A unique identifier for the challenge, used by the AS to detect replay.

`client_id`:
: REQUIRED. The client identifier of the AI Agent whose request was refused, as determined from the access token the AI Agent presented (for example, from the `client_id` claim of a JWT access token {{RFC9068}}, or from a token introspection response {{RFC7662}}).

`sub`:
: REQUIRED when the access token presented to the RS was issued on behalf of a resource owner; otherwise absent. The subject identifier of that resource owner, as known to the AS. This tells the AS whose approval it must obtain.

`scope`:
: REQUIRED. A space-delimited list of the scopes required to perform the operation, in the format of the `scope` parameter of {{Section 3.3 of RFC6749}}.

`reason`:
: REQUIRED. A human-readable explanation of why the operation requires these scopes, suitable for display to the resource owner. See {{reason-guidance}}.

Additional claims MAY be present. The AS MUST ignore claims it does not understand.

The claims of the challenge shown in the example above, decoded, are:

~~~
{
  "iss": "https://api.example.com",
  "aud": "https://auth.example.com",
  "iat": 1793469600,
  "exp": 1793469900,
  "jti": "9f1c2e3a-4b5d-4e6f-8a9b-0c1d2e3f4a5b",
  "client_id": "reservebot",
  "sub": "248289761001",
  "scope": "urn:example:reservations.write",
  "reason": "ReserveBot wants to book a table for two at Joe's
             Italian on Friday at 9pm."
}
~~~

#### Content of the Reason {#reason-guidance}

The quality of the AAuth flow depends on the reason being a concise, simple-to-understand explanation of the specific action the AI Agent is attempting, in a form that lets the resource owner decide whether to approve it. It is generated by the RS at the time of the refused request, and so can describe that request precisely ("book a table for two at Joe's Italian on Friday at 9pm") rather than describing the scope in general ("make reservations").

The reason MUST be plain text. It MUST NOT contain markup, and the AS MUST NOT interpret it as anything other than plain text when rendering it. It SHOULD be short enough to display in constrained contexts such as mobile push notifications; the AS MAY truncate reasons that exceed its display limits. The reason MUST NOT contain credentials, tokens, or other secrets, and SHOULD NOT contain personal data beyond what is needed to describe the action. The RS SHOULD write the reason in the resource owner's preferred language when that is known to it.

## Step 3: Initiate AAuth Grant Flow with AS {#step3}

Next, the AI Agent initiates the flow defined in this specification to obtain a higher scoped, short-lived access token with the set of required scopes. To do that, it sends an HTTP POST to a new endpoint of the AS defined in this specification - the agent authorization endpoint - and includes the Scope Challenge it received from the resource server. The AS learns the required scopes and the reason from the Scope Challenge; the AI Agent does not supply them separately, and cannot modify them. The location of the agent authorization endpoint is published in the AS's metadata ({{as-metadata}}).

An example of such a request is:

~~~
POST /agent_authorization HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=urn:ietf:params:oauth:grant-type:agent_authorization
&scope_challenge=eyJhbGciOiJFUzI1NiIsInR5cCI6InNjb3BlLWNoYWxsZW5
nZStqd3QiLCJraWQiOiJyczEifQ.eyJpc3MiOiJodHRwczovL2FwaS5leGFtcGxl
LmNvbSIsImF1ZCI6Imh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbSIs...fQ.DtsKx1Q
...2Qw
&timeout=300
~~~
{: title="Agent authorization request (line breaks added for readability)"}

The following parameters are included in the body:

`grant_type`:
: REQUIRED. MUST be `urn:ietf:params:oauth:grant-type:agent_authorization`.

`scope_challenge`:
: REQUIRED. The Scope Challenge, exactly as received from the RS.

`timeout`:
: OPTIONAL. The maximum time, in seconds, that the AI Agent is willing to wait for approval. See {{timeout}}.

The AI Agent MUST authenticate to the agent authorization endpoint using the client authentication method registered for it, as described in {{Section 2.3 of RFC6749}}. This grant is defined for confidential clients only; an AS MUST reject a request from a public client with an `unauthorized_client` error.

### Agent-Supplied Timeout {#timeout}

An AI Agent often knows how long it can usefully wait for approval - for example, because it is itself operating under a deadline set by its user. The AI Agent MAY communicate this by including a `timeout` parameter in the agent authorization request.

When `timeout` is present, the AS MUST NOT return an `expires_in` value ({{step4-response}}) greater than the agent's `timeout`. The AS MAY return a smaller value in accordance with its own policy; the effective lifetime of the request is therefore the lower of the two. When `timeout` is absent, the AS determines the lifetime alone.

If the agent's `timeout` is shorter than the AS is able to support - for example, shorter than its approval channel can reasonably deliver a decision - the AS MAY reject the request with an `invalid_request` error rather than initiate an approval that the resource owner cannot complete in time. In that case the AS SHOULD state the minimum it supports in the `error_description`.

Whichever party's limit applies, once it has elapsed the request is expired: the AS MUST NOT deliver a token for it on any channel, and SHOULD withdraw any outstanding approval prompt so that the resource owner is not asked to approve an action the AI Agent has already abandoned.

## Step 4: AS Processes Grant Request {#step4}

The AS validates the client credentials as per normal OAuth processing.

### Validating the Scope Challenge {#validate-challenge}

The AS then validates the Scope Challenge. If any of the following checks fails, the AS MUST reject the request with an `invalid_scope_challenge` error ({{errors}}), except where another error code is stated.

1. The value MUST parse as a JWS-signed JWT with a `typ` header parameter of `scope-challenge+jwt`.

2. The `iss` claim MUST identify a resource server that the AS is configured to issue tokens for. The AS MUST NOT process challenges from resource servers it does not recognize; see {{sec-challenge}}.

3. The AS obtains the RS's Protected Resource Metadata from the well-known location derived from `iss`, as described in {{Section 3 of RFC9728}}, and the RS's JWK Set from the `jwks_uri` member of that metadata. The AS SHOULD cache both documents in accordance with the HTTP caching directives they are served with, and MAY re-fetch the JWK Set when it encounters an unrecognized `kid`, subject to rate limiting ({{sec-dos}}).

4. The signature MUST verify using a key from that JWK Set, with an algorithm acceptable to the AS and, if the RS publishes `resource_signing_alg_values_supported`, listed there.

5. The `aud` claim MUST match the AS's issuer identifier.

6. The current time MUST be before `exp`. The AS MAY also reject challenges whose `iat` is unreasonably far in the past.

7. The `client_id` claim MUST match the identifier of the authenticated client.

8. The `jti` claim MUST NOT have been accepted previously. The AS MUST retain the `jti` values it has accepted at least until the corresponding `exp` times have passed, and MUST reject any challenge whose `jti` it has already accepted. This prevents an AI Agent from using a single challenge to initiate multiple approval requests.

9. The AS MUST be willing, under its own policy, to issue each of the scopes in `scope` to this client for this resource server. If it is not, it MUST reject the request with an `invalid_scope` error.

### Obtaining Scope Descriptions

In order for the approver - whether it is a human or an AI agent operating on their behalf - to approve the request for higher scopes, natural language descriptions of those scopes are needed. The AS obtains them from the RS's Scope Descriptions Document ({{scope-descriptions}}), located through the `scope_descriptions_uri` member of the RS's Protected Resource Metadata ({{rs-metadata}}), or from its own configuration. The AS SHOULD cache the Scope Descriptions Document in accordance with the HTTP caching directives it is served with.

The AS MUST NOT ask the resource owner to approve a scope for which it has no human-readable description. If a description is unavailable for any scope in the challenge, the AS MUST reject the request with an `invalid_scope` error.

### Agent Authorization Response {#step4-response}

Having validated the challenge and obtained a description of each scope, the AS creates a pending authorization request, initiates the approval process ({{approval}}), and returns an HTTP 200 (OK) response to the AI Agent with a JSON body of the following form:

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "request_code": "GhiJkl-QRstuVwxyz",
  "token_endpoint": "https://auth.example.com/token",
  "user_approval_endpoint" : "https://auth.example.com/approve",
  "poll_interval": 5,
  "expires_in": 300,
  "poll_sse_endpoint":
    "https://auth.example.com/agent_authorization/sse",
  "poll_ws_endpoint":
    "wss://auth.example.com/agent_authorization/ws"
}
~~~

This response is not providing the AI Agent with a token - it is informing the AI Agent that permission is required from the resource owner to proceed. The AI Agent will need to come back and obtain the token later, once permission has been given. Since the AI Agent doesn't know how long it will take for such permission to be granted, the AS provides the AI Agent with a few options.

`request_code`:
: REQUIRED. An opaque handle that the AI Agent presents when retrieving the token. The AS MUST bind the request_code to the authenticated client, to the subject, scopes, and reason of the Scope Challenge, and to the outcome of the approval. The request_code MUST contain enough entropy that it cannot be guessed; at least 128 bits is RECOMMENDED. It is single-use: once a token has been delivered for it on any retrieval channel, it is consumed.

`token_endpoint`:
: REQUIRED. The URL of the AS's token endpoint, which the AI Agent can poll to retrieve the token ({{polling}}).

`expires_in`:
: REQUIRED. The lifetime of the request_code in seconds. This is the maximum duration the AS is willing to wait for approval, and the upper bound on the time for which the AI Agent may poll. When the AI Agent supplied a `timeout`, this value MUST NOT exceed it ({{timeout}}).

`poll_interval`:
: OPTIONAL. The minimum interval, in seconds, that the AI Agent MUST wait between polling requests. If absent, the default is 5 seconds. The AS can adjust the interval later using the `Retry-After` header ({{errors}}).

`poll_sse_endpoint`:
: OPTIONAL. An `https` URL at which the AI Agent can subscribe to a Server-Sent Events stream that delivers the outcome ({{sse}}).

`poll_ws_endpoint`:
: OPTIONAL. A `wss` URL at which the AI Agent can open a WebSocket connection that delivers the outcome ({{ws}}).

`user_approval_endpoint`:
: OPTIONAL. The URL of a page, rendered by the AS, through which the resource owner can approve or reject the request. See {{web-approval}} and {{sec-approval-channel}}.

The AS MAY support SSE or WebSocket to allow for push-based notification once the resource owner has granted permission; it advertises these by including the corresponding endpoints in the response.

When the AI Agent accesses any of these endpoints, it MUST pass the request_code back to the AS, and MUST authenticate as the client that made the agent authorization request. The AS MUST reject any attempt to use a request_code by a client other than the one it was issued to with an `invalid_grant` error. The request_code allows the AS to echo state back to itself, for scaling and security reasons.

## User Approval {#approval}

The AS MUST obtain user approval in order to grant the token. It may do so using the simple approval technique defined here, or employ any number of mechanisms outside the scope of this specification to do so.

In this architecture, the AS plays the role of advocate for the resource owner, making sure their security interests are met. As such, it relies on a trust relationship between the resource owner and the AS.

Whatever technique is used, the information presented to the approver MUST be taken from the validated Scope Challenge and from the AS's own records: the registered name of the client ({{RFC7591}}), the identity of the resource server (for example, the `resource_name` from its metadata), the `reason`, and the human-readable description of each requested scope. The AS MUST NOT present text supplied directly by the AI Agent. See {{sec-approval-channel}} for the properties the approval channel needs to have.

### Non-Standardized Approval Techniques

Examples of how the AS can gain approval include sending the user an SMS or email, sending a push notification to the user's mobile device, or making a policy decision through programmatic or agentic means.

As an example, consider an AI Agent that wishes to book a dinner reservation on behalf of a user, using OpenTable APIs. In this case, OpenTable is the resource server, and its AS is the one executing the AAuth grant flow. If the user has the OpenTable app on their mobile phone, they could get a push notification like, "AI Agent ReserveBot wants to book dinner at Joe's Italian at 9pm on Friday. Do you approve?". The text in this case, comes from the "reason" parameter generated on the RS. This allows the AS to perform this push notification without knowing intimate details on the behavior of the APIs on the RS.

As another example, the AS can avoid a HITL entirely by running an AI agent whose sole goal is to determine whether or not the approve the access.


### Web Approval Technique {#web-approval}

This specification defines a simple web based approval technique, and is OPTIONAL to implement on an AS. If the AS supports it, it includes the user_approval_endpoint field in the response to the agent_authorization request.

The user_approval_endpoint URI points to a page, renderable in a browser, that presents the user with the reason for the approval, and then lists and describes the scopes that are being requested. This page is rendered by the AS.

The AS SHOULD authenticate the end user, and once authenticated, present the information in a way that makes it clear that an AI Agent is asking for these permissions, and why.

Once the AS has collected the users approval (or rejection), it can now pass (or not pass) the token to the AI Agent using the next steps in the flow. The approval (or rejection) are bound to the request_code, so that the AI Agent can retrieve it.

Note that the AI agent itself is not involved in the rendering of this approval page. The AI Agent receives the URI, and then it MUST launch a browser and open that page, and let the human user perform the approval. The AI Agent silently awaits approval, using one of the token retrieval techniques defined here.

## Token Retrieval {#retrieval}

After the agent authorization request has been accepted, the AI Agent obtains its access token via one of three techniques: HTTP polling of the token endpoint, a Server-Sent Events stream, or a WebSocket connection. The following rules apply to all three:

* The AI Agent MUST authenticate as the client that made the agent authorization request, using its registered client authentication method. For the SSE and WebSocket endpoints, which are accessed with GET, a client that authenticates with a client secret MUST use the HTTP Basic scheme and MUST NOT place the secret in the query component; a client that authenticates with a `client_assertion` sends it as a query parameter.
* The AI Agent MUST present the request_code.
* The AS MUST deliver the token for a given request_code at most once, across all channels. The AI Agent MAY use more than one technique for the same request_code (for example, an SSE stream with polling as a fallback), but the token will be delivered on exactly one of them; the others will report that the request_code has been consumed.
* Every response that carries a token or request state MUST include `Cache-Control: no-store`.
* All endpoints MUST be served over TLS.

On success, the token response is a standard OAuth 2.0 token response as defined in {{Section 5.1 of RFC6749}}. The issued token SHOULD be short-lived, and its scope MUST NOT exceed the scope of the Scope Challenge.

### HTTP Polling {#polling}

The AI Agent can retrieve the token by posting to the token endpoint:

~~~
POST /token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=urn:ietf:params:oauth:grant-type:agent_authorization
&request_code=GhiJkl-QRstuVwxyz
~~~

Note that the endpoint requires the AI Agent to authenticate itself using its client ID and secret. Note further that the body contains the request_code parameter, which acts as the primary key bound to the approval and the resulting token.

The AS communicates the state of the request through the HTTP status code, so that the AI Agent can determine the outcome without parsing the response body:

| Status | State | Body |
|---|---|---|
| 200 (OK) | Approved; token issued | Token response ({{Section 5.1 of RFC6749}}) |
| 202 (Accepted) | Approval pending | `{"status":"authorization_pending"}`. `Retry-After` SHOULD be present. |
| 403 (Forbidden) | Resource owner or policy denied the request | `{"error":"access_denied"}` |
| 410 (Gone) | request_code expired, or already consumed | `{"error":"expired_token"}` or `{"error":"invalid_grant"}` |
| 429 (Too Many Requests) | AI Agent is polling too quickly | `{"error":"slow_down"}`. `Retry-After` MUST be present. |
{: title="Token endpoint status codes for the agent authorization grant"}

Requests that are malformed, that present an unknown request_code or one issued to another client, or that fail client authentication, receive the status codes and error responses defined in {{Section 5.2 of RFC6749}} (400, or 401 for `invalid_client`). Server-side failures receive 5xx status codes. {{errors}} gives the complete mapping.

This departs from {{RFC8628}}, which returns 400 for the `authorization_pending` and `slow_down` conditions. {{Section 5.2 of RFC6749}} permits extensions to specify status codes other than 400 for their errors.

Examples:

~~~
HTTP/1.1 202 Accepted
Content-Type: application/json
Cache-Control: no-store
Retry-After: 5

{"status":"authorization_pending"}
~~~

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "urn:example:reservations.write"
}
~~~

Once a request has been denied, has expired, or has been consumed, the AS MUST continue to return the corresponding status for subsequent requests presenting that request_code, for at least the request_code's original lifetime.

### Server-Sent Events (SSE) {#sse}

If the agent authorization response included a `poll_sse_endpoint`, the AI Agent can subscribe to a Server-Sent Events stream {{HTML}} on which the AS delivers the outcome as soon as it is known.

#### Establishing the Stream

~~~
GET /agent_authorization/sse?request_code=GhiJkl-QRstuVwxyz HTTP/1.1
Host: auth.example.com
Accept: text/event-stream
Authorization: Basic <base64(client_id:client_secret)>
~~~

The AI Agent sends a GET request to the `poll_sse_endpoint` with the request_code as a query parameter, an `Accept` header of `text/event-stream`, and client authentication.

The AS evaluates the current state of the request at the time of the GET, exactly as it would for a polling request:

* If client authentication fails, the request is malformed, the request_code is unknown or was issued to another client, the client is being rate-limited, or the request has already been denied, has expired, or has been consumed, the AS responds with the same status code and JSON body it would return to a polling request ({{polling}}, {{errors}}), and does not open a stream.
* Otherwise, the AS responds with a 200 (OK) status, a `Content-Type` of `text/event-stream`, and `Cache-Control: no-store`, and begins the event stream. If the request has already been approved, the AS delivers the `token_response` event immediately.

#### Event Format

Events are formatted as described in {{HTML}}. Each event carries an `event` field naming the event type and a `data` field containing a single JSON object {{RFC8259}}. The object MAY be split across several `data` lines, as {{HTML}} permits, since the newlines introduced when the lines are joined are insignificant whitespace in JSON. The AS MAY include `id` and `retry` fields. Events are not replayed on reconnection, so the `Last-Event-ID` request header has no effect. The AI Agent MUST ignore event types and JSON members it does not recognize.

This specification defines the following event types:

`authorization_pending`:
: Sent once, immediately after the stream is opened, to confirm that the subscription is active. The `data` object contains the `request_code` and `expires_in`, the number of seconds remaining before the request expires. Not terminal.

`token_response`:
: Sent when the request has been approved. The `data` object is a token response as defined in {{Section 5.1 of RFC6749}}. Terminal.

`error`:
: Sent when the request has reached a terminal state other than approval. The `data` object is an error response as defined in {{Section 5.2 of RFC6749}}, with `error` set to `access_denied` (the resource owner or policy denied the request), `expired_token` (the request expired while the stream was open), `invalid_grant` (the token was delivered on another channel), or `server_error`. Terminal.

#### Stream Lifecycle

While the stream is open, the AS SHOULD send a comment line (a line beginning with a colon) at least every 30 seconds, so that the connection is not closed as idle by intermediaries. AI Agents ignore comment lines, as {{HTML}} requires.

After sending a terminal event, the AS MUST close the connection. The AI Agent MUST treat the request as complete and MUST NOT reconnect for that request_code.

If the connection is closed without a terminal event having been received - because of a network failure, an AS restart, or an intermediary timeout - the AI Agent MAY reconnect, after waiting the interval given by the most recent `retry` field or, if none was given, `poll_interval`. The AS handles the new request exactly as described above, according to the current state of the request. The AS MUST NOT send a `token_response` event for a request_code on which it has already sent one; an AI Agent that reconnects after the token was delivered receives a 410 (Gone) response, and if it did not receive the token, must restart the flow from {{step2}}. See {{sec-push}}.

When `expires_in` elapses while the stream is open, the AS MUST send an `error` event with `expired_token` and close the connection. The AS MAY close a stream earlier without a terminal event, for example to rebalance load; the AI Agent then reconnects as described above.

#### Example

~~~
event: authorization_pending
data: {"request_code":"GhiJkl-QRstuVwxyz","expires_in":287}

: keepalive

: keepalive

event: token_response
data: {"access_token":"2YotnFZFEjr1zCsicMWpAA",
data:  "token_type":"Bearer","expires_in":900,
data:  "scope":"urn:example:reservations.write"}

~~~
{: title="SSE stream ending in token delivery; the AS closes the connection after the final event"}

~~~
event: authorization_pending
data: {"request_code":"GhiJkl-QRstuVwxyz","expires_in":287}

: keepalive

event: error
data: {"error":"access_denied",
data:  "error_description":"The user denied the request."}

~~~
{: title="SSE stream ending in denial"}

### WebSocket {#ws}

If the agent authorization response included a `poll_ws_endpoint`, the AI Agent can open a WebSocket connection {{RFC6455}} on which the AS delivers the outcome as soon as it is known.

#### Establishing the Connection

~~~
GET /agent_authorization/ws?request_code=GhiJkl-QRstuVwxyz HTTP/1.1
Host: auth.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: aauth.agent-flow
Authorization: Basic <base64(client_id:client_secret)>
~~~

The `poll_ws_endpoint` MUST use the `wss` scheme. The AI Agent sends the opening handshake with the request_code as a query parameter, client authentication, and a `Sec-WebSocket-Protocol` header requesting the `aauth.agent-flow` subprotocol.

The AS evaluates the current state of the request at handshake time, exactly as for SSE: in the conditions listed in {{sse}} under which no stream is opened, the AS responds with the corresponding HTTP status code and JSON body instead of completing the handshake. Otherwise it completes the handshake with a 101 (Switching Protocols) response that includes `Sec-WebSocket-Protocol: aauth.agent-flow`. If the AS does not select this subprotocol, the AI Agent MUST close the connection.

#### Message Format

Every message is a text frame (opcode 0x1) whose payload is exactly one JSON object {{RFC8259}}. The AS MUST NOT send binary frames. Each object has a `type` member identifying the message. The AI Agent MUST ignore messages whose `type` it does not recognize, and members it does not recognize.

This specification defines the following AS-to-agent message types, whose semantics are those of the SSE events of the same name:

`authorization_pending`:
: Sent once, immediately after the handshake completes. Contains `request_code` and `expires_in`. Not terminal.

`token_response`:
: Sent when the request has been approved. The remaining members of the object are those of a token response ({{Section 5.1 of RFC6749}}). Terminal.

`error`:
: Sent when the request has reached a terminal state other than approval. The remaining members are those of an error response ({{Section 5.2 of RFC6749}}), with the same `error` values as the SSE `error` event. Terminal.

This specification defines no agent-to-AS data messages. An AS MUST ignore data messages it does not recognize.

#### Connection Lifecycle

While the connection is open, the AS SHOULD send a Ping control frame at least every 30 seconds, and the AI Agent MUST respond with a Pong frame as required by {{Section 5.5.3 of RFC6455}}. Either party MAY close a connection on which the peer does not respond to Pings.

After sending a terminal message, the AS MUST close the connection with status code 1000 (Normal Closure). The AI Agent MUST treat the request as complete and MUST NOT reconnect for that request_code.

When `expires_in` elapses while the connection is open, the AS MUST send an `error` message with `expired_token` and then close with status code 1000. An AS that must shut down a connection before a terminal message has been sent SHOULD close with status code 1001 (Going Away); the AI Agent MAY then reconnect after waiting `poll_interval`, and the AS handles the new handshake according to the current state of the request. As with SSE, the AS MUST NOT send a `token_response` for a request_code on which it has already sent one; an AI Agent that lost the connection before receiving the token must restart the flow from {{step2}}.

An AS MAY close a connection with status code 1003 (Unsupported Data) if it receives a binary frame, or 1008 (Policy Violation) if the AI Agent otherwise violates this specification.

#### Example

On open:

~~~
{"type":"authorization_pending",
 "request_code":"GhiJkl-QRstuVwxyz","expires_in":287}
~~~

On approval:

~~~
{"type":"token_response",
 "access_token":"2YotnFZFEjr1zCsicMWpAA",
 "token_type":"Bearer","expires_in":900,
 "scope":"urn:example:reservations.write"}
~~~

On denial:

~~~
{"type":"error","error":"access_denied",
 "error_description":"The user denied the request."}
~~~

On expiration:

~~~
{"type":"error","error":"expired_token",
 "error_description":"The request_code has expired."}
~~~


## Error Handling and Back-Off {#errors}

### Status Codes

The AS uses HTTP status codes {{RFC9110}} to report the outcome of requests to the agent authorization endpoint, to the token endpoint when the `grant_type` is the one defined here, and to the SSE and WebSocket endpoints when no stream is opened. The AI Agent can act on the status code alone; the JSON body, when present, carries the OAuth error code and an optional `error_description`.

| Status | OAuth `error` | Meaning |
|---|---|---|
| 200 (OK) | - | Agent authorization request accepted; or token issued. |
| 202 (Accepted) | - (`status` is `authorization_pending`) | Token endpoint: approval pending. Poll again after `Retry-After`. |
| 400 (Bad Request) | `invalid_request`, `invalid_grant`, `invalid_scope`, `unauthorized_client`, `unsupported_grant_type`, `invalid_scope_challenge` | The request is malformed or unacceptable, including an unknown request_code or one issued to another client. Not retryable without change. |
| 401 (Unauthorized) | `invalid_client` | Client authentication failed. The response MUST include a `WWW-Authenticate` header, as required by {{Section 5.2 of RFC6749}}. |
| 403 (Forbidden) | `access_denied` | The resource owner, or policy, denied the request. Terminal. |
| 410 (Gone) | `expired_token`, `invalid_grant` | The request_code has expired (`expired_token`), or a token has already been delivered for it (`invalid_grant`). Terminal. |
| 429 (Too Many Requests) {{RFC6585}} | `slow_down` | The client has exceeded a rate limit: polling too quickly, opening too many push connections, or creating too many agent authorization requests. `Retry-After` MUST be present. Retryable. |
| 500 (Internal Server Error) | `server_error` | Unexpected server-side failure. Retryable. |
| 503 (Service Unavailable) | `temporarily_unavailable` | The server is overloaded or under maintenance. `Retry-After` SHOULD be present. Retryable. |
{: title="HTTP status codes and OAuth error codes used by this specification"}

The AS MUST use these status codes. In particular, it MUST return 429 rather than 400 when rate limiting, MUST return a 5xx status for server-side failures rather than reporting them as client errors, and MUST return 410 once a request_code has expired or been consumed. Error codes defined by {{RFC6749}} retain the status codes assigned there (400, or 401 for `invalid_client`); the states particular to asynchronous approval (202, 403, 410, and 429) are the additions of this specification.

`invalid_scope_challenge` is a new error code defined by this specification. It is returned by the agent authorization endpoint when the Scope Challenge fails any of the checks in {{validate-challenge}}.

All error responses that have a body use the format of {{Section 5.2 of RFC6749}} and MUST include `Cache-Control: no-store`.

Once a stream has been opened, the SSE and WebSocket endpoints report terminal outcomes through `error` events and messages rather than HTTP status codes; the mapping is given in {{sse}} and {{ws}}.

### Back-Off

The following rules govern the AI Agent's behavior when polling and when connecting to push endpoints:

* The AI Agent MUST NOT poll more frequently than `poll_interval`, and MUST NOT make a further request before any `Retry-After` interval it has received on a 202, 429, or 503 response has elapsed. `Retry-After` is the means by which the AS adjusts the polling interval after the initial response.
* On receiving a 429 or 503 response without a `Retry-After` header, the AI Agent MUST at least double the interval it last used before retrying, and SHOULD add random jitter.
* On receiving a 500 response, the AI Agent MAY retry, with exponential back-off and jitter.
* On receiving a 400, 401, 403, or 410 response, the AI Agent MUST stop polling for that request_code; these outcomes are not retryable.
* The AI Agent MUST stop polling and treat the request as expired once `expires_in` has elapsed since the agent authorization response, whether or not it has received a 410 response.
* An AI Agent that receives a 429 response when attempting to open an SSE or WebSocket connection MUST wait at least the `Retry-After` interval before attempting again.

The AS SHOULD apply rate limits per client, and SHOULD limit the number of concurrent push connections, and of outstanding agent authorization requests, per client and per resource owner ({{sec-dos}}).

# Discovery Metadata {#metadata}

## Authorization Server Metadata {#as-metadata}

This specification defines the following OAuth Authorization Server Metadata {{RFC8414}} member:

`agent_authorization_endpoint`:
: URL of the AS's agent authorization endpoint ({{step3}}). An AS that supports this specification SHOULD publish it.

## Protected Resource Metadata {#rs-metadata}

A resource server that supports this specification publishes OAuth Protected Resource Metadata {{RFC9728}} at the well-known location derived from its resource identifier. In that metadata, the RS:

* MUST include `jwks_uri`, referencing a JWK Set that contains the public key(s) used to sign Scope Challenges. Keys used for this purpose SHOULD carry a `use` value of `sig`.
* SHOULD include `resource_signing_alg_values_supported`, listing the algorithms it uses to sign Scope Challenges.
* SHOULD include `authorization_servers`, listing the authorization servers for which it issues Scope Challenges. The AS SHOULD verify that it is listed.
* SHOULD include `scope_descriptions_uri`, defined below, unless the AS obtains scope descriptions by other means.
* SHOULD include `resource_name`, which the AS can display to the resource owner to identify the RS.

This specification defines one new Protected Resource Metadata member:

`scope_descriptions_uri`:
: URL of the RS's Scope Descriptions Document ({{scope-descriptions}}). This URL MUST use the `https` scheme.

# Security Considerations

## Client Credentials

AI Agents are confidential clients and MUST protect their long-lived credentials. The security of every step of this flow depends on client authentication: the Scope Challenge is bound to a client, the request_code is bound to a client, and the token is delivered only to that client. The guidance of {{RFC9700}} applies.

## Integrity of the Reason and Scopes {#sec-challenge}

The reason and the scopes are the information on which the resource owner bases their decision. If an AI Agent could alter either, it could obtain approval for one action while presenting another - for example, by pairing a benign reason with a broader set of scopes. The Scope Challenge prevents this: both values are signed by the RS, which is the authority on what its operations require, and the AS verifies the signature against keys published in the RS's Protected Resource Metadata. Signing the reason alone would not be sufficient, since an agent could then request additional scopes under cover of a legitimate reason; for this reason the scopes and the reason are bound together in a single signed object.

The AS MUST accept Scope Challenges only from resource servers it is configured to serve. An AS that accepted challenges from any resource identifier would allow an attacker to stand up a resource server of their own and use the AS's approval user interface to deliver arbitrary text to the AS's users, lending it the AS's credibility.

The `client_id` claim binds the challenge to the AI Agent that encountered the refusal, so that a challenge obtained by one client cannot be redeemed by another. The `sub` claim binds it to the resource owner whose token was presented, so that an AI Agent cannot cause a different resource owner to be prompted. The `jti` claim and the short `exp` prevent a single challenge from being used to generate repeated approval prompts.

The AS trusts the RS for the content of the reason and of the scope descriptions. Even so, both are text that will be displayed to a person. The AS MUST render them as plain text, MUST NOT interpret markup in them, and SHOULD apply the same precautions against spoofing (for example, through bidirectional override characters) that it applies to any other externally supplied display string.

## request_code {#sec-request-code}

A request_code is a bearer handle to a pending approval and, once approved, to a token. It is protected by client authentication on every endpoint at which it is presented, so that the request_code alone is insufficient to retrieve a token; by sufficient entropy to resist guessing; by a short lifetime; and by single use.

The SSE and WebSocket endpoints receive the request_code in the query component of the request URI, which is more likely than a request body to be recorded in server, proxy, and client logs. The AS and AI Agents SHOULD NOT log request_code values, and operators SHOULD configure intermediaries accordingly.

The request_code SHOULD NOT be exposed to the resource owner or to a browser; see {{sec-approval-channel}}.

## Delivery of Tokens over Push Channels {#sec-push}

Delivering the access token over an SSE stream or a WebSocket connection is as secure as delivering it in a token endpoint response, provided that the following requirements are met:

* The connection MUST use TLS (`https` or `wss`). Plaintext `http` or `ws` endpoints MUST NOT be advertised or used.
* The connection MUST be client-authenticated, with the same client authentication the token endpoint requires. A push endpoint that accepted a request_code without client authentication would deliver the token to anyone who learned the request_code.
* The AS MUST verify that the request_code was issued to the authenticated client.
* The AS MUST deliver the token at most once. Because the AS cannot know whether a pushed message was received before a connection was lost, an AI Agent that loses the connection during delivery loses the token and must restart the flow. This is deliberate: the alternative, redelivering on reconnection, would make the token retrievable more than once.
* The AS MUST close the connection after delivering a terminal event.

Two limitations of push delivery are worth noting. First, sender-constrained tokens: a token delivered in a token endpoint response can be bound to a DPoP proof {{RFC9449}} presented with the request. No equivalent proof accompanies a pushed token, and this specification does not define a profile for binding a token to a proof presented when the push connection was established. Deployments that require DPoP-bound tokens MUST deliver them through the token endpoint ({{polling}}). Tokens bound to a client certificate {{RFC8705}} MAY be delivered over a push connection established with that certificate. Second, delivery confirmation: a token endpoint response is part of a request/response exchange, so the AS knows whether the token was delivered; a push is not.

Push connections are long-lived, and are often routed through different infrastructure - load balancers, connection-holding proxies - than ordinary API requests. Infrastructure that terminates TLS sees the token in cleartext, and MUST NOT log or buffer frame payloads in a way that retains tokens.

Editor's note: because of these limitations, a future revision of this specification may restrict the push channels to notifying the AI Agent that the request has been decided, with the token itself always retrieved from the token endpoint.

## Approval Channels and Agent-Mediated Approval {#sec-approval-channel}

The security of this grant rests on the approval being made by the resource owner, or by a policy agent acting for them, and not by the AI Agent whose access is being decided. An approval channel that the AI Agent can observe or operate does not provide this property. This deserves particular attention for AI Agents, since many are capable of driving a web browser or other user interface: an agent that can open a browser to an approval page can, in general, also activate the "approve" control on that page.

For this reason, the AS SHOULD obtain approval through a channel that it establishes with the resource owner directly, independent of the AI Agent - for example a push notification to an application the AS has previously enrolled, a message to a verified address, or an authenticated session in the AS's own user interface. The AS has such a relationship with the resource owner by construction: it authenticated them when it issued the minimal-scope token, and the Scope Challenge identifies them through its `sub` claim.

When the Web Approval Technique ({{web-approval}}) is used and the approval URL is returned to the AI Agent, the following considerations apply:

* The AI Agent is placed in the path between the AS and the resource owner. A malicious or compromised agent can suppress the URL, substitute a look-alike page that captures the resource owner's AS credentials, or display the genuine page inside a user interface it controls. Resource owners who become accustomed to following approval links supplied by agents also become easier to phish by other means. This is the same class of risk described in {{Section 5.4 of RFC8628}}, where it is accepted because a device may have no other channel to the user; an AS implementing this specification does have another channel.
* The approval URL SHOULD NOT contain the request_code. The request_code is the handle through which the token is retrieved and should be known only to the AI Agent and the AS. An AS SHOULD instead use a distinct, single-use, user-facing identifier bound to the request, analogous to the `user_code` of {{RFC8628}}.
* The AS MUST authenticate the resource owner at the approval page, and MUST verify that the authenticated user is the subject identified by the Scope Challenge. It SHOULD require a fresh authentication that includes a user-presence check (for example, a WebAuthn assertion) immediately before recording the approval; this reduces the risk that an automated agent operating the browser can approve on the resource owner's behalf.
* Approval MUST require an explicit user action, MUST be protected against cross-site request forgery, and MUST NOT be possible through a GET request. The page MUST NOT be embeddable in a frame (for example, through a Content Security Policy of `frame-ancestors 'none'`).
* The page MUST display the client's registered name, the resource server, the reason, and the description of each scope, taken from the validated Scope Challenge and the AS's own records, and MUST NOT display text supplied by the AI Agent.

## Rate Limiting and Resource Exhaustion {#sec-dos}

The AS SHOULD rate-limit polling and push connections per client, returning 429 as described in {{errors}}. It SHOULD limit the number of simultaneous push connections per client and per request_code, and SHOULD limit the number of outstanding approval requests per client and per resource owner, since each approval request may result in a notification to a person. The `jti` of the Scope Challenge ensures that each challenge can create at most one request.

Validation of a Scope Challenge may cause the AS to fetch an RS's metadata and JWK Set. The AS SHOULD cache these, and SHOULD rate-limit re-fetches triggered by unrecognized `kid` values, so that an AI Agent cannot cause the AS to make unbounded outbound requests.

## Scope Descriptions {#sec-scope-descriptions}

Scope descriptions are published by the RS and displayed to resource owners. They SHOULD describe the effect of a scope in terms the resource owner can act on, and SHOULD NOT expose sensitive details of the RS's implementation. The AS MUST render them as plain text, as it does the reason ({{sec-challenge}}).

## Elevated Tokens

Tokens issued through this grant are approved for a specific action, described by the reason. The AS SHOULD issue them with a short lifetime, and SHOULD NOT issue a refresh token with them: an AI Agent that needs to perform the action again, or to perform a further action, should obtain a new approval. In contrast to Token Exchange {{RFC8693}}, in which a client obtains a new token without resource owner involvement, every token issued through this grant corresponds to an approval.

# IANA Considerations

## OAuth URI Registration

This specification registers the following value in the "OAuth URI" sub-registry of the "OAuth Parameters" registry:

* URN: `urn:ietf:params:oauth:grant-type:agent_authorization`
* Common Name: Agent Authorization Grant Type
* Change Controller: IETF
* Specification Document: this document

## OAuth Parameters Registration

This specification registers the following values in the "OAuth Parameters" registry established by {{RFC6749}}:

* Parameter name: `scope_challenge`; Parameter usage location: agent authorization request; Change controller: IETF; Specification document: {{step3}} of this document.
* Parameter name: `timeout`; Parameter usage location: agent authorization request; Change controller: IETF; Specification document: {{timeout}} of this document.
* Parameter name: `request_code`; Parameter usage location: token request; Change controller: IETF; Specification document: {{polling}} of this document.

## OAuth Extensions Error Registration

This specification registers the following value in the "OAuth Extensions Error" registry established by {{RFC6749}}:

* Error name: `invalid_scope_challenge`
* Error usage location: agent authorization endpoint error response
* Related protocol extension: Agent Authorization Grant
* Change controller: IETF
* Specification document: {{errors}} of this document

## OAuth Authorization Server Metadata Registration

This specification registers the following value in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

* Metadata name: `agent_authorization_endpoint`
* Metadata description: URL of the agent authorization endpoint
* Change controller: IETF
* Specification document: {{as-metadata}} of this document

## OAuth Protected Resource Metadata Registration

This specification registers the following value in the "OAuth Protected Resource Metadata" registry established by {{RFC9728}}:

* Metadata name: `scope_descriptions_uri`
* Metadata description: URL of the resource server's Scope Descriptions Document
* Change controller: IETF
* Specification document: {{rs-metadata}} of this document

## Media Type Registration

This specification registers the `application/scope-challenge+jwt` media type in the "Media Types" registry, for use as the `typ` header parameter value `scope-challenge+jwt` of a Scope Challenge ({{scope-challenge}}). The registration follows the template of {{RFC7519}} for `application/jwt`, with the security considerations of {{sec-challenge}}.

## WebSocket Subprotocol Registration

This specification registers the following value in the "WebSocket Subprotocol Name" registry established by {{RFC6455}}:

* Subprotocol Identifier: `aauth.agent-flow`
* Subprotocol Common Name: AAuth Agent Authorization Flow
* Subprotocol Definition: {{ws}} of this document

## Well-Defined Scope

There is no IANA registry for OAuth scope values. The `ai-agent-minimal` scope is defined by this specification ({{step1}}) and requires no registration.



--- back

# Scope Descriptions Document {#scope-descriptions}

This appendix defines the format of the Scope Descriptions Document that an RS publishes at its `scope_descriptions_uri` ({{rs-metadata}}), and gives a JSON Schema {{I-D.bhutton-json-schema}} for it.

Note: Human-readable, localizable scope descriptions published by the resource server are useful well beyond this specification. Any consent user interface, and any client that explains to a user what it is asking for, has the same need. It may make sense to publish this format as its own OAuth extension, with this specification simply referencing it. Feedback on that question is welcome.

## Format

The document is a JSON object {{RFC8259}}, served with a media type of `application/json` over HTTPS. The RS SHOULD serve it with caching directives that allow the AS to cache it. Its members are:

`default_locale`:
: OPTIONAL. A language tag {{RFC5646}} identifying the language of the top-level `title` and `description` of each scope. If absent, that language is unspecified.

`scope_descriptions`:
: REQUIRED. An object whose member names are scope values and whose member values are Scope Description objects.

A Scope Description object has the following members:

`title`:
: REQUIRED. A short label for the scope, suitable for use as a heading or as a single line in a list. At most 100 characters.

`description`:
: REQUIRED. One or two sentences describing what a client holding this scope can do, in terms the resource owner can act on. At most 500 characters.

`risk`:
: OPTIONAL. The RS's assessment of the risk of granting the scope: one of `low`, `medium`, or `high`. An AS MAY use this to adjust how prominently it presents the scope, or to inform an automated approval policy.

`localizations`:
: OPTIONAL. An object whose member names are language tags {{RFC5646}} and whose member values are objects containing a `title` and a `description` in that language, subject to the same constraints as the top-level members.

Unrecognized members MUST be ignored, to allow for extension.

When presenting a scope, the AS selects the localization that best matches the resource owner's language preferences - for example, by applying the lookup algorithm of {{RFC4647}} to the member names of `localizations` - and falls back to the top-level `title` and `description` if none matches. The AS MUST render all of these values as plain text.

## Example

~~~
{
  "default_locale": "en",
  "scope_descriptions": {
    "urn:example:reservations.read": {
      "title": "View your reservations",
      "description": "Lets the agent see your upcoming and past
                      restaurant reservations.",
      "risk": "low",
      "localizations": {
        "de": {
          "title": "Ihre Reservierungen anzeigen",
          "description": "Erlaubt dem Agenten, Ihre vergangenen
                          und kommenden Restaurantreservierungen
                          einzusehen."
        }
      }
    },
    "urn:example:reservations.write": {
      "title": "Make and change reservations",
      "description": "Lets the agent book, change, and cancel
                      restaurant reservations on your behalf.",
      "risk": "high"
    }
  }
}
~~~
{: title="Scope Descriptions Document (line breaks within strings added for readability)"}

## JSON Schema

~~~
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AAuth Scope Descriptions Document",
  "type": "object",
  "required": ["scope_descriptions"],
  "properties": {
    "default_locale": { "$ref": "#/$defs/languageTag" },
    "scope_descriptions": {
      "type": "object",
      "minProperties": 1,
      "additionalProperties": { "$ref": "#/$defs/scopeDescription" }
    }
  },
  "$defs": {
    "languageTag": {
      "type": "string",
      "description": "A language tag as defined by RFC 5646",
      "pattern": "^[A-Za-z]{2,8}(-[A-Za-z0-9]{1,8})*$"
    },
    "title": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "description": {
      "type": "string",
      "minLength": 1,
      "maxLength": 500
    },
    "localizedText": {
      "type": "object",
      "required": ["title", "description"],
      "properties": {
        "title": { "$ref": "#/$defs/title" },
        "description": { "$ref": "#/$defs/description" }
      }
    },
    "scopeDescription": {
      "type": "object",
      "required": ["title", "description"],
      "properties": {
        "title": { "$ref": "#/$defs/title" },
        "description": { "$ref": "#/$defs/description" },
        "risk": {
          "type": "string",
          "enum": ["low", "medium", "high"]
        },
        "localizations": {
          "type": "object",
          "propertyNames": { "$ref": "#/$defs/languageTag" },
          "additionalProperties": {
            "$ref": "#/$defs/localizedText"
          }
        }
      }
    }
  }
}
~~~
{: title="JSON Schema for the Scope Descriptions Document"}
