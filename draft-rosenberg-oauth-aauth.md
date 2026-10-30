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
    email: "pat.white@traego.com"

normative:

  RFC2119:
  RFC8628:
  RFC8693:





informative:

draft-ietf-oauth-v2:


--- abstract

AI Agents are increasingly accessing web resources on behalf of the users they serve. Traditionally these agents are granted normal OAuth access tokens through existing OAuth grant flows. However, these flows were designed in the era of traditional software, and when applied to AI Agents, multiple problems have arisen. This document outlines some of the problems, and then proposes a simple OAuth2.1 extension that enables AI Agents to obtain access tokens at time of resource access using Human-In-the-Loop or other approval flows. We call this the Agent Authorization Grant (aauth) grant flow. This document also compares the proposed solution with another, non-backwards compatible aauth solution. 



--- middle

# Introduction {#intro}

OAuth 2.1 [RFC6749] provides a framework for delegated access via bearer tokens. It has been wildly successful, with widespread deployment across the Internet. It was built to solve a key problem - how to allow software applications to access resources on a user's behalf, without requiring the user to give the application their username and password or other credentials. 

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
: Issues `request_code` handles, manages user consent, and issues access tokens.

Resource Server (RS):
: API server that protects resources via OAuth tokens and publishes human-readable scope descriptions at `/.well-known/aauth.json`.

request_code:
: Opaque handle returned by the AS to correlate an agent’s request with subsequent token retrieval.

reason:
: Human-readable explanation provided by the agent, shown to the user during consent.

:scope_descriptions:
: Natural-language text for each requested scope, fetched from the RS’s discovery document.


# The AAuth Grant Flow

This flow is modeled on the OAuth 2.0 Device Authorization Grant [RFC8628] but is tailored for confidential agent clients with long-lived identities, and optimized for performant delivery of tokens to the agent.

The flow executes throught a series of steps.


## Step 1: Requesting the Mimimal Scope AI Agent TOken

In the first step, the AI Agent requests an access token with very limited scope. This scope has a very specific meaning in the context of this specification. It refers to the set of access that is permitted by an AI Agent which is completely safe and should not require the resource owner - or software or AI agents operating on their behalf -  to explicitly approve. The precise set of API calls that fit under this umbrella are entirely a matter of local implementation in the resource server. 

We call this scope the "Minimal AI Agent scope. This specification standardizes a well-known value for this scope - "ai-agent-minimal". This is similar in concept to OIDC which defines a small set of well-defined scopes [OIDC].

AI Agents gain access tokens associated with this scope through any of the existing OAuth2.0 grant flows. 

For the authorization code grant flow, it is RECOMMENDED that Authorization Servers render information to the resource owner, informing them that this is an AI Agent, and they may be required to perform additional approvals to when the AI Agent does something that has some risk. It is RECOMMENDED that examples are provided on what type of access is considered risky. For example, a travel website may allow an AI Agent to list flights, or enumerate airports. However, listing of a user's travel plans or making flight changes, is considered risky. 

In all ways, the access token obtained with this scope is a normal access token and the AI Agent can use it in the same way it uses any other access token. 

## Step 2: Access the Resource

When the AI Agent accesses a resource on the resource server using the minimal AI Agent scope, if the minimal scope associated with the token is sufficient, the resource server will accept the request and all is well.

However, if the AI Agent attempts to perform an action which the resource server determines is beyond the minimal AI agent scope, it rejects it with a 403 Forbidden HTTP response and includes an insufficient_scope error code [RFC6750] in the response. It will also include the set of scopes that are required to perform the operation. Importantly, it also adds a new parameter - "reason" - which contains a natural language explanation of the reason that the minimal AI agent scope was insufficient.


## Step 3: Initiate AAauth Grant Flow with AS

Next, the AI Agent initiates the flow defined in this specification to obtain a higher scoped, short-lived access token with the set of required scopes. To do that, it initiates an HTTP POST operation to the AS, to a new endpoint defined in this specification - "/agent_authorization", and includes the required scopes - and importantly - the value of the reason that it received from the resource server. (TODO - should probably add signatures onto the reason and scopes so a malicious AI agent cannot modify them). 

An example of such a request is:

~~~
POST /agent_authorization HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=urn:ietf:params:oauth:grant-type:agent_authorization&
scope=urn:example:resource.read urn:example:resource.write&
reason="<human-readable reason>"
~~~

The following parameters are included in the body:

* grant_type: MUST be `urn:ietf:params:oauth:grant-type:agent_authorization`.

* scope: Space-delimited list of scope URIs.

* reason: Concise, human-readable explanation provided by the resource server; MUST be echoed verbatim by the AS.

The quality of the AAuth flow depends on the reason string being a concise, simple to understand explanatio of the action in a way that the user can make a decision on approval, or not. 

### Step 4: AS Processes Grant Request

The AS validates the client credentials as per normal OAuth processing.

In order for the approver - whether it is a human or an AI agent operating on their behalf - to approve the request for higher scopes - natural language descriptions of those scopes are needed. To obtain them, the AS fetches scope descriptions from a well-known URI defined by this specification:

~~~
https://{rs}/.well-known/aauth.json#scope_descriptions`
~~~

The response comes back as text meant to be rendered to the user, one per scope. 
TODO - define a JSON format for this; also need to deal with localization. 

Assuming it is able to retrieve a natural language description of each scope, it returns success to the AI Agent, in the following form:


~~~
{
  "request_code": "GhiJkl-QRstuVwxyz",
  "token_endpoint": "https://auth.example.com/token",
  "user_approval_endpoint" : "https://auth.example.com/approve",
  "poll_interval": 5,
  "expires_in": 600,
  "poll_sse_endpoint": "https://auth.example.com/agent_authorization/sse",
  "poll_ws_endpoint": "wss://auth.example.com/agent_authorization/ws"
}
~~~

This response is not providing the AI Agent with a token - it is informing the AI Agent that user permission is required from the user to proceed. The AI Agent will need to come back and obtain the token later, once the user has given permission. Since the AI Agent doesn't know how long it will take for such permission to be granted, the AS provides the AI Agent with a few options. 

The AS MUST include a token_endpoint parameter in the response, which is a URL that the AI Agent can poll to retrieve the token. The AS MAY include a poll_internal in the body which specifies the requested polling interval, in seconds. If not provided, the default is 5 seconds. (TODO - add exponential backoff and AS-provided updates to poll intervals). The AS MAY include an expires_in parameter, which indicates the maximum duration the AS is willing to wait for user approval. This is also the upper bound on the time for which the AI Agent can poll.

AS MAY support SSE or WS to allow for push-based notifications once the user has granted permission. In those cases, it includes an HTTP URI in the poll_sse_endpoint and poll_ws_endpoint, respectively.

When the AI Agent accesses any of these endpoints, it MUST pass the request_code back to the AS in a request_code URI parmaeter. This allows the AS to echo state back to itself, for scaling and security reasons. The AS SHOULD associate this request_code with the requested scopes and reason.

To facilitate human-in-the-loop approvals, the AS MAY include a user_approval_endpoint field in the response. This is a resource meant to be rendered to the user to request their approval. 

## User Approval


The AS MUST obtain user approval in order to grant the token. It may do so using the simple approval technqiue defined here, or employ any number of mechanisms outside the scope of this specification to do so. 

### Non-Standardized Approval Techniques

Examples of how the AS can gain approval include sending the user an SMS or email, or sending a push notification to the user's mobile device.

As an example, consider an AI Agent that wishes to book a dinner reservation on behalf of a user, using OpenTable APIs. In this case, OpenTable is the resource server, and its AS is the one executing the AAuth grant flow. If the user has the OpenTable app on their mobile phone, they could get a push notification like, "AI Agent ReserveBot wants to book dinner at Joe's Italian at 9pm on Friday. Do you approve?". The text in this case, comes from the "reason" parameter generated on the RS. This allows the AS to perform this push notification without knowing intimate details on the behavior of the APIs on the RS. 

### Web Approval Technique

This specification defines a simple web based approval technique that can be used. This technique is OPTIONAL to implement on an AS. If the AS supports it, it includes the user_approval_endpoint field in the response. 

HERE


The AS is fully responsible for user interaction:

1. Initiate the consent review flow with the user. This spec does not specify how this occurs, it could be through an app, a chat client, or some other mechanism.

2. Display the `reason` and each `scope_descriptions` entry to ensure the user has all the information necessary to assess the consent request.

3. Authenticate the user (including MFA) and, upon consent, bind approval to `request_code`.

The agent does **not** handle redirects or UI rendering—it passively awaits token availability.

## Token Retrieval

After user approval, the agent obtains its access token via one of three techniques.

### HTTP Polling

The agent can retrieve the token by hitting this endpoint:

~~~
   POST /token HTTP/1.1
   Host: auth.example.com
   Content-Type: application/x-www-form-urlencoded
   Authorization: Basic <base64(client_id:client_secret)>

   grant_type=urn:ietf:params:oauth:grant-type:device_code&
   device_code=GhiJkl-QRstuVwxyz


   *Pending*: `{"error":"authorization_pending"}`
   *Success*: Standard OAuth 2.x token response.
~~~

### Server-Sent Events (SSE)

The agent can use SSE.

~~~
   GET /agent_authorization/sse?request_code=GhiJkl-QRstuVwxyz HTTP/1.1
   Host: auth.example.com
   Authorization: Bearer <agent_jwt>
   Accept: text/event-stream
~~~

On approval:

~~~
   event: token_response
   data: {"access_token":"<JWT>","expires_in":900,"issued_token_type":"urn:ietf:params:oauth:token-type:jwt"}
~~~

### WebSocket

The agent can connect via websockets.

~~~
   GET /agent_authorization/ws?request_code=GhiJkl-QRstuVwxyz HTTP/1.1
   Host: auth.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Protocol: aauth.agent-flow
   Authorization: Bearer <agent_jwt>
~~~

   On open:

~~~
   {
     "type": "token_response",
     "access_token": "<JWT>",
     "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
     "expires_in": 900
   }
~~~

## Error Handling & Back-Off

With HTTP Polling, the agent MUST honor `error="slow_down"` with `Retry-After` headers; return standard OAuth error codes such as `authorization_pending`, `access_denied`, or `expired_token`.

With SSE, the following events are possible:

On token response:

~~~
    event: token_response
    data: {"access_token":"<JWT>","expires_in":900,"issued_token_type":"urn:ietf:params:oauth:token-type:jwt"}
~~~

On user rejection:

~~~
    event: error
    data: {"error":"access_denied","error_description":"The user denied the request."}
~~~

On request expiration:

~~~
    event: error
    data: {"error":"expired_token","error_description":"The request_code has expired."}
~~~

And for websockets, the following responses are possible:


On token response:

~~~
    {"type":"token_response","access_token":"<JWT>","issued_token_type":"urn:ietf:params:oauth:token-type:jwt","expires_in":900}
~~~

On user rejection or expiration:

~~~
    {"type":"error","error":"access_denied","error_description":"The user denied the request."}
~~~

On request expiration:

~~~
    {"type":"error","error":"expired_token","error_description":"The request_code has expired."}
~~~


# Security Considerations

* Agents MUST protect their long-lived credentials.
* Short-lived `request_code` and tokens limit replay risk.
* RS scope descriptions should not expose sensitive system details.
* Implement rate-limiting on polling and push channels to prevent abuse.

# IANA Considerations

This document registers the following new OAuth grant type:

~~~
urn:ietf:params:oauth:grant-type:agent_authorization
~~~



--- back

