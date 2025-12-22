---
title: "OAuth 2.0 direct interation for native clients using federation"
abbrev: "OAuth native clients with federation"
category: std

docname: draft-zehavi-oauth-native-clients-direct-interation-with-federation-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - native apps
 - first-party
 - oauth
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "yaron-zehavi/oauth-native-clients-direct-interation-with-federation"
  latest: "https://yaron-zehavi.github.io/oauth-native-clients-direct-interation-with-federation/draft-zehavi-oauth-native-clients-direct-interation-with-federation.html"

author:
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
  RFC6749:
  RFC7159:
  RFC7515:
  RFC7519:
  RFC7591:
  RFC7636:
  RFC8259:
  RFC8414:
  RFC8628:
  RFC8707:
  RFC9126:
  RFC9396:
  RFC9449:
  RFC9470:
  I-D.ietf-oauth-first-party-apps:
  I-D.ietf-oauth-cross-device-security:
  OpenID.Native-SSO:
    title: OpenID Connect Native SSO for Mobile Apps
    target: https://openid.net/specs/openid-connect-native-sso-1_0.html
    author:
      - ins: G. Fletcher
    date: November 2022
  OpenID:
    title: OpenID Connect Core 1.0
    target: https://openid.net/specs/openid-connect-core-1_0.html
    date: November 8, 2014
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore
  OpenID.Federation:
    title: OpenID Federation 1.0
    target: https://openid.net/specs/openid-federation-1_0.html
    date: March 5, 2025
    author:
      - ins: R. Hedberg, Ed.
      - ins: M.B. Jones
      - ins: A.A. Solberg
      - ins: J. Bradley
      - ins: G. De Marco
      - ins: V. Dzhuvinov
  OpenID4VP:
    title: OpenID for Verifiable Presentations 1.0
    target: https://openid.net/specs/openid-4-verifiable-presentations-1_0.html
    date: July 9, 2025
    author:
      - ins: O. Terbu
      - ins: T. Lodderstedt
      - ins: K. Yasuda
      - ins: D. Fett
      - ins: J. Heenan
  IANA.oauth-parameters:
  IANA.JWT:
  USASCII:
    title: "Coded Character Set -- 7-bit American Standard Code for Information Interchange, ANSI X3.4"
    author:
      name: "American National Standards Institute"
    date: 1986
  SHS:
    title: "\"Secure Hash Standard (SHS)\", FIPS PUB 180-4, DOI 10.6028/NIST.FIPS.180-4"
    author:
      name: "National Institute of Standards and Technology"
    date: August 2015
    target: http://dx.doi.org/10.6028/NIST.FIPS.180-4

informative:
  RFC6750:
  RFC8252:
  I-D.ietf-oauth-browser-based-apps:
  I-D.ietf-oauth-attestation-based-client-auth:

--- abstract

{{I-D.ietf-oauth-first-party-apps}} defined native OAuth 2.0 **direct interaction**,
whereby clients call the *Native Authorization Endpoint* as an HTTP REST API,
obtaining instructions from authorization servers on information to collect from
users to satisfy authorization server's policies and requirements.

While FiPA {{I-D.ietf-oauth-first-party-apps}} focused on a one-to-one relationship between
client and authorization server, this document acts as its **extension** adding support
for authorization servers to federate the interaction to a downstream authorization server,
instruct the usage of a native app for user interaction, or instruct collection of additional
information from users to guide request routing.

--- middle

# Introduction

This document, OAuth 2.0 direct interation for native clients using federation,
extends FiPA {{I-D.ietf-oauth-first-party-apps}} to enable federation based flows,
while retaining client's direct interaction with end-user.

The client calls the *Native Authorization Endpoint* as an HTTP REST API, and receives
instructions according to the protocol established by FiPA,
guiding client to call downstream authorization servers and providing their responses
to federating authorization servers. This establishes a multi authorization server
federated flow, whose user interactions are driven by the client app.

This document extends FiPA {{I-D.ietf-oauth-first-party-apps}} with new error responses:
`federate`, `redirect_to_app`, `insufficient_information` and
`native_authorization_federate_unsupported`.

It also adds additional response parameters:
`federation_uri`, `federation_body`, `response_uri`, `deep_link`.

And adds the `native_callback_uri` request parameter.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

There are three primary ways this specification extends FiPA:

* `federate` response: Sends the client to interact with a downstream authorization server.
* `insufficient_information` response: Instructs the client to collect information from end-user required to decide where to federate to. For example this could be an email address which identifies the trust domain.
* `redirect_to_app`: Instructs the client to natively invoke an app to interact with end user.

## Example flow: Native client federated and redirected to app

~~~ ascii-art
                                                +--------------------+
                                                |   Authorization    |
                          (B)Native             |      Server 1      |
             +----------+ Authorization Request |+------------------+|
(A)User  +---|          |---------------------->||     Native       ||
   Starts|   |          |                       ||  Authorization   ||
   Flow  +-->|  Client  |<----------------------||    Endpoint      ||
             |          | (C)Federate Error     |+------------------+|
             |          |        Response       +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (D)Native             |      Server 2      |
             |          | Authorization Request |+------------------+|
             |          |---------------------->||     Native       ||
             |          |                       ||  Authorization   ||
             |          |<----------------------||    Endpoint      ||
             |          | (E) Redirect to       |+------------------+|
             |          |     App Response      +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          | (F) Invoke App        |                    |
             |          |---------------------->|   Native App of    |
             |          |                       |   Auth Server 2    |
             |          |<----------------------|                    |
             |          | (G)Authorization code +--------------------+
             |          |   For Auth Server 1
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (H)Authorization Code |      Server 1      |
             |          |    For Auth Server 1  |+------------------+|
             |          |---------------------->||    Response      ||
             |          |                       ||       Uri        ||
             |          |<----------------------||    Endpoint      ||
             |          | (I) Authorization     |+------------------+|
             |          |     Code Response     |                    |
             |          |         :             |                    |
             |          |         :             |                    |
             |          | (J) Token Request     |+------------------+|
             |          |---------------------->||      Token       ||
             |          |                       ||     Endpoint     ||
             |          |<----------------------||                  ||
             |          | (K) Access Token      |+------------------+|
             |          |                       +--------------------+
             |          |
             +----------+
~~~
Figure: Native client federated, then redirected to app

- (A) The client starts the flow.
- (B) The client initiates the authorization request by making a POST request to the Native Authorization Endpoint of Authorization Server 1.
- (C) Authorization Server 1 decides to federate the user to Authorization Server 2. To do so it contacts Authorization Server 2's PAR {{RFC9126}} endpoint, then returns the `federate` error code together with the *federation_uri*, *federation_body*, *response_uri* and *auth_session* response attributes.
- (D) The client calls Authorization Server 2's Native Authorization Endpoint, as instructed by Authorization Server 1.
- (E) Authorization Server 2 decides to use a native app, and therefore responds with the `redirect_to_app` error code together with the *deep_link* response attribute.
- (F) The client invokes the app using the deep_link.
- (G) The app interacts with user and if satisifed, returns an authorization code, regarded as Authorization Server 2's response to Authorization Server 1's federation to it.
- (H) The client provides the authorization code to Authorization Server 1.
- (I) Authorization Server 1 returns an authorization code.
- (J) The client sends the authorization code received in step (I) to obtain a token from the Token Endpoint.
- (K) Authorization Server 1 returns an Access Token from the Token Endpoint.

# Protocol Endpoints

## Native Authorization Endpoint {#native-authorization-endpoint}

The native authorization endpoint defined by FiPA {{I-D.ietf-oauth-first-party-apps}} is used by this document.

This document adds the *native_callback_uri* parameter to the native authorization endpoint, to support
cross-app native user navigation.

Before authorization servers instruct a client to federate to a downstream authorization server, they MUST ensure it offers a *native_authorization_endpoint*, otherwise return the error native_authorization_federate_unsupported*.

When federating to downstream authorization servers, the usage of PAR {{RFC9126}} with client authentication is REQUIRED, as the native client calling the Native Authorization Endpoint of a federated authorization server is not *its* OAuth client and therefore has no other means of authenticating.
When using PAR with client authentication, the request_uri provided to the Native Authorization Endpoint attests that client authentication took place.

## Native Authorization Request {#native-auth-request}

The native authorization endpoint is called as defined by FiPA {{I-D.ietf-oauth-first-party-apps}}.
This document adds the following request parameter:

"native_callback_uri":
: OPTIONAL. Native client app's **redirect_uri**, claimed as deep link. *native_callback_uri* SHALL be natively invoked by authorization server's user-interacting app to provide its response to the client app. If native_callback_uri is included in a native authorization request, authorization server MUST include the native_callback_uri when federating to another authorization server.

## Native Authorization Response {#native-response}

### Error Response {#native-authorization-error-response}

This document extends FiPA {{I-D.ietf-oauth-first-party-apps}} by adding the following error codes:

"error":
:    REQUIRED.  A single ASCII {{USASCII}} error code from the following:

     "insufficient_information":
     :     the Authorization Server requires additional user input,
           other than an authentication challenge, to determine the
           target authorization server to federate to.
           See {{insufficient-information}} for details.

     "federate":
     :     The Authorization Server wishes to federate to another
           authorization server, which it is a client of. This
           response MUST include the *federation_uri* response parameter.
           See {{federating-response}} for details.

     "redirect_to_app":
     :     The Authorization Server wishes to fulfill the user interaction
           using another native app. This response MUST include the
           *federation_uri* response parameter.
           See {{redirect-to-app-response}} for details.

     "native_authorization_federate_unsupported":
     :     The authorization server intended to federate to
           a downstream authorization server, but it does not
           support the native authorization endpoint.

And adds the following response attributes:

"federation_uri":
:    OPTIONAL.  The Native Authorization Endpoint of a downstream
     authorization server to federate to.

"deep_link":
:    OPTIONAL.  A URI of native app to be invoked to handle the request.

"federation_body":
:    OPTIONAL.  A string of application/x-www-form-urlencoded
     request parameters according to this specification for the
     downstream authorization server's Native Authorization Endpoint.

"response_uri":
:    OPTIONAL.  A URI of an endpoint of federating authorization server
     which shall receive the response from the federated authorization server.

#### Federating Response {#federating-response}

If the authorization server decides to federate to another authorization server, it
responds with error code *federate* and MUST return the *federation_uri*,
*federation_body*, *response_uri* and *auth_session* response attributes.

When federating to another authorization server:
* Federating authorization server MUST use PAR {{RFC9126}} and include *request_uri* in federation_body.
* If *native_callback_uri* was included in the native authorization request, it MUST be included when calling federated authorization server's Native Authorization Endpoint.

Example **federating** response:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "federate",
        "auth_session": "ce6772f5e07bc8361572f",
        "response_uri": "https://prev-as.com/native-authorization",
        "federation_uri": "https://next-as.com/native-authorization",
        "federation_body": "client_id=s6BhdRkqt3&request_uri=
          urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX"
    }

Client MUST call the *federation_uri* using HTTP POST, and provide it *federation_body*
as application/x-www-form-urlencoded request body. Example:

    POST /native-authorization HTTP/1.1
    Host: next-as.com
    Content-Type: application/x-www-form-urlencoded

    client_id=s6BhdRkqt3&request_uri=
    urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX

The client MUST provide any response obtained from the **federated** authorization server,
as application/x-www-form-urlencoded request body for the *response_uri* of the respective
**federating** authorization server which SHALL be invoked using HTTP POST.

However, when **federated** authorization server returns the following error codes:
*federate*, *insufficient_authorization*, *insufficient_information*, *redirect_to_app*,
*redirect_to_web*, client MUST handle these errors according to this specification.

Example client calling receiving an authorization code response {{authorization-code-response}} from the federated
authorization server:

    HTTP/1.1 200 OK
    Server: next-as.com
    Content-Type: application/json
    Cache-Control: no-store

    {
      "authorization_code": "uY29tL2F1dGhlbnRpY"
    }

And providing it to the federating authorization server's response_uri,
adding previously obtained auth_session:

    POST /native-authorization HTTP/1.1
    Host: prev-as.com
    Content-Type: application/x-www-form-urlencoded

    authorization_code=uY29tL2F1dGhlbnRpY
    &auth_session=ce6772f5e07bc8361572f

#### Redirect to app response {#redirect-to-app-response}

If the authorization server decides to use a native app to interact with
end user, it responds with error code *redirect_to_app* and MUST return the
*deep_link* response attribute.

Example **redirect_to_app** response:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "redirect_to_app",
        "deep_link": "https://next-as.com/native-authorization?
          client_id=s6BhdRkqt3&request_uri=
          urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX"
    }

Client MUST use OS mechanisms to invoke the obtained deep_link.
If no app claiming deep_link is found on the device, client MUST terminate the
flow and MAY attempt a normal non-native OAuth flow.

The invoked app handles the native authorization request:

* Validates the request and ensures it contains a *native_callback_uri*, Otherwise terminates the flow.
* Establishes trust in *native_callback_uri* and validates that an app claiming it is on the device. Otherwise terminates the flow.
* Authenticates end-user and authorizes the request.
* Uses OS mechanisms to natively invoke *native_callback_uri* and return to the client, providing it a response according to this specification's response from a Native Authorization Endpoint, as url-encoded query parameters.

Note - trust establishment mechanisms in *native_callback_uri* are out of scope of this specification.
However we assume closed ecosystems could employ an allowList, and open ecosystems could leverage
{{OpenID.Federation}}:

  * Extract native_callback_uri's DNS domain.
  * Add the path /.well-known/openid-federation and perform trust chain resolution.
  * Inspect client's metadata for redirect_uri's and validate **native_callback_uri** is included among them.

When the client is invoked on its native_callback_uri, it shall regard the invocation as a
response **from** the authorization server which redirected the client to the app.
In other words, this response's audience is not the authorization server which redirected
the client to the app. See {{federating-response}} for details.

Example URI used to invoke of client app on its claimed native_callback_uri:

    https://client.example.com/cb?authorization_code=uY29tL2F1dGhlbnRpY

Example client invoking the response_uri **of the authorization server which federated it**
to the authorization server, which redirected it to the app:

    POST /native-authorization HTTP/1.1
    Host: **prev-as.com**
    Content-Type: application/x-www-form-urlencoded

    authorization_code=uY29tL2F1dGhlbnRpY
    &auth_session=ce6772f5e07bc8361572f

#### Additional Information Required Response {#insufficient-information}

If additional user input is required, for example to determine where to federate to,
the response body shall contain the following additional properties:

logo:
: OPTIONAL. URL or base64-encoded logo of *Authorization Server*, for branding purposes.

userPrompt:
: REQUIRED. A JSON object containing the prompt definition. The following parameters MAY be used:

- options: OPTIONAL. A JSON object that defines a dropdown/select input with various options to choose from. Each key is the parameter name to be sent in the response and each value defines the option:

  - title: OPTIONAL. A string holding the input's title.
  - description: OPTIONAL. A string holding the input's description.
  - values: REQUIRED. A JSON object where each key is the selection value and each value holds display data for that value:

    - name: REQUIRED. A string holding the display name of the selection value.
    - logo: OPTIONAL. A string holding a URL or base64-encoded image for that selection value.
- inputs: OPTIONAL. A JSON object that defines an input field. Each key is the parameter name to be sent in the response and each value defines the input field:

  - title: OPTIONAL. A string holding the input's title.
  - hint: OPTIONAL. A string holding the input's hint that is displayed if the input is empty.
  - description: OPTIONAL. A string holding the input's description.

Example of requesting end-user for 2 multiple-choice inputs:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "insufficient_information",
        "auth_session": "ce6772f5e07bc8361572f",
        "logo": "uri or base64-encoded logo of Authorization Server",
        "userPrompt": {
            "options": {
                "bank": {
                    "title": "Bank",
                    "description": "Choose your Bank",
                    "values": {
                        "bankOfSomething": {
                            "name": "Bank of Something",
                            "logo": "uri or base64-encoded logo"
                        },
                        "firstBankOfCountry": {
                            "name": "First Bank of Country",
                            "logo": "uri or base64-encoded logo"
                        }
                    }
                },
                "segment": {
                    "title": "Customer Segment",
                    "description": "Choose your Customer Segment",
                    "values": {
                        "retail": "Retail",
                        "smb": "Small & Medium Businesses",
                        "corporate": "Corporate",
                        "ic": "Institutional Clients"
                    }
                }
            }
        }
    }

Example of requesting end-user for text input entry (email):

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "insufficient_information",
        "auth_session": "ce6772f5e07bc8361572f",
        "action": "prompt",
        "id": "request-identifier-2",
        "logo": "uri or base64-encoded logo of Authorization Server",
        "userPrompt": {
            "inputs": {
                "email": {
                    "hint": "Enter your email address",
                    "title": "E-Mail",
                    "description": "Lorem Ipsum"
                }
            }
        }
    }

The client gathers the required additional information and makes a POST request to the Native Authorization Endpoint. Example of response following end-user multiple-choice:

    POST /native-authorization HTTP/1.1
    Host: example.as.com
    Content-Type: application/x-www-form-urlencoded

    auth_session=ce6772f5e07bc8361572f
    &bank=bankOfSomething
    &segment=retail

Example of *Client App* response following end-user input entry:

    POST /native-authorization HTTP/1.1
    Host: example.as.com
    Content-Type: application/x-www-form-urlencoded

    auth_session=ce6772f5e07bc8361572f
    &email=end_user@example.as.com

# Security Considerations {#security-considerations}

## Non First-Party applications of federated authorization servers {#first-party-applications}

A federated authorization server should consider end-user's privacy and security
to determine if it should present authorization challenges in federation scenarios.
For example, it can label **federating** clients as such and avoid serving them
authorization challenges, as user-serving clients receiving those challenges are
not first party clients.

# IANA Considerations

## OAuth Parameters Registration

IANA has (TBD) registered the following values in the IANA "OAuth Parameters" registry of {{IANA.oauth-parameters}} established by {{RFC6749}}.

**Parameter name**: `native_callback_uri`

**Parameter usage location**: Native Authorization Endpoint

**Change Controller**: IETF

**Specification Document**: Section 5.4 of this specification

--- back

# Example User Experiences

This section provides non-normative examples of how this specification may be used to support specific use cases.

## Native client federated and redirected to app

### Diagram

~~~ ascii-art
                                                +--------------------+
                                                |   Authorization    |
                          (B)Native             |      Server 1      |
             +----------+ Authorization Request |+------------------+|
(A)User  +---|          |---------------------->||     Native       ||
   Starts|   |          |                       ||  Authorization   ||
   Flow  +-->|  Client  |<----------------------||    Endpoint      ||
             |          | (C)Federate Error     |+------------------+|
             |          |        Response       +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (D)Native             |      Server 2      |
             |          | Authorization Request |+------------------+|
             |          |---------------------->||     Native       ||
             |          |                       ||  Authorization   ||
             |          |<----------------------||    Endpoint      ||
             |          | (E) Redirect to       |+------------------+|
             |          |     App Response      +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          | (F) Invoke App        |                    |
             |          |---------------------->|   Native App of    |
             |          |                       |   Auth Server 2    |
             |          |<----------------------|                    |
             |          | (G)Authorization code +--------------------+
             |          |   For Auth Server 1
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (H)Authorization Code |      Server 1      |
             |          |    For Auth Server 1  |+------------------+|
             |          |---------------------->||    Response      ||
             |          |                       ||       Uri        ||
             |          |<----------------------||    Endpoint      ||
             |          | (I) Authorization     |+------------------+|
             |          |     Code Response     |                    |
             |          |         :             |                    |
             |          |         :             |                    |
             |          | (J) Token Request     |+------------------+|
             |          |---------------------->||      Token       ||
             |          |                       ||     Endpoint     ||
             |          |<----------------------||                  ||
             |          | (K) Access Token      |+------------------+|
             |          |                       +--------------------+
             |          |
             +----------+
~~~
Figure: Native client federated, then redirected to app

### Client makes initial request and receives "federate" error

Client calls the native authorization endpoint and includes the *native_callback_uri* parameter:

    POST /native-authorization HTTP/1.1
    Host: as-1.com
    Content-Type: application/x-www-form-urlencoded

    client_id=t7CieSlru4&native_callback_uri=
    https://client.example.com/cb

The first authorization server, as-1.com, decides to federate to as-2.com after validating
it supports the native authorization endpoint. If it does not, as-1.com returns:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "native_authorization_federate_unsupported"
    }

If native authorization endpoint is supported by the federated authorization server,
as-1.com performs a PAR {{RFC9126}} request to as-2.com's pushed authorization endpoint,
including the original *native_callback_uri*:

    POST /par HTTP/1.1
    Host: as-2.com
    Content-Type: application/x-www-form-urlencoded

    client_id=s6BhdRkqt3&native_callback_uri=
    https://client.example.com/cb

as-1.com receives a request_uri from as-2.com's PAR endpoint, which it
includes in its response to client, in the *federation_body* attribute:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "federate",
        "auth_session": "ce6772f5e07bc8361572f",
        "response_uri": "https://as-1.com/native-authorization",
        "federation_uri": "https://as-2.com/native-authorization",
        "federation_body": "client_id=s6BhdRkqt3&request_uri=
          urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX"
    }

See {{federating-response}} for more details.

### Client calls federated authorization server and is redirected to app

Client calls the *federation_uri* it got from as-1.com using HTTP POST with
*federation_body* as application/x-www-form-urlencoded request body:

    POST /native-authorization HTTP/1.1
    Host: as-2.com
    Content-Type: application/x-www-form-urlencoded

    client_id=s6BhdRkqt3&request_uri=
    urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX

as-2.com decides to use its native app to interact with end-user and responds:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "redirect_to_app",
        "deep_link": "https://as-2.com/native-authorization?
          client_id=s6BhdRkqt3&request_uri=
          urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX"
    }

Client locates an app claiming the obtained deep_link and invokes it.
See {{redirect-to-app-response}} for more details.
The invoked app handles the native authorization request and then natively invokes native_callback_uri:

    https://client.example.com/cb?authorization_code=uY29tL2F1dGhlbnRpY

### Client calls federating authorization server

Client invokes the response_uri of as-1.com as it is the authorization server which federated
it to as-2.com and the app's response is regarded as the response of as-2.com:

    POST /native-authorization HTTP/1.1
    Host: as-1.com
    Content-Type: application/x-www-form-urlencoded

    authorization_code=uY29tL2F1dGhlbnRpY
    &auth_session=ce6772f5e07bc8361572f

And receives in response an authorization code, which it is the audience of (no further federations) to resolve:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "authorization_code": "vZ3:uM3G2eHimcoSqjZ"
    }

Client proceeds to exchange code for tokens.

## Usage with Digital Credentials {#digital-credentials}

Digital credentials (stored in wallets) may be used by authorization servers to authenticate a user or approve a transaction. This example flow shows how wallet credential presentation can be incorporated into this spec using DC API/{{OpenID4VP}}.

~~~ ascii-art
User            First Party Client        AS               Wallet/DC API        Verifier
----            ------------------        ---              --------------       --------
|                     |                  |                    |                  |
|                     |                  |                    |                  |
| (1) Start Flow      |                  |                    |                  |
|-------------------->|                  |                    |                  |
|                     | (2) Native Authorization Request      |                  |
|                     |----------------->|                    |                  |
|                     | (3) Authorization Error Response      |                  |
|                     |   (insufficient_authorization,        |                  |
|                     |     dc_credential_required)           |                  |
|                     |<-----------------|                    |                  |
|                     |                  |                    |                  |
+===================== DC API branch ============================================+
|                     | (4) Presentation Request (OID4VP)     |                  |
|                     |-------------------------------------->|                  |
|                     | (5) Presentation Response (vp_token)  |                  |
|                     |<--------------------------------------|                  |
|                     | (6) vp_token                          |                  |
|                     |----------------->|                    |                  |
|                     |                  | (7) vp_token       |                  |
|                     |                  |-------------------------------------->|
|                     |                  | (8) Verification Result               |
|                     |                  |<--------------------------------------|
+================================================================================+
+=================== No DC API branch ===========================================+
|------- Same device path -------------------------------------------------------|
|                     | (9) Invoke Wallet via Deeplink        |                  |
|                     |-------------------------------------->|                  |
|                     |                  |                    | (10) Presentation Response (vp_token)
|                     |                  |                    |----------------->|
|                     |                  | (11) redirect_uri_client?handle       |
|                     |                  |<--------------------------------------|
|                     | (12) Redirect to client w/ openid4vp_redirect_uri?handle=123
|                     |<--------------------------------------|                  |
|                     | (13) Handle                           |                  |
|                     |----------------->|                    |                  |
|                     |                  | (14) Get Presentation Result          |
|                     |                  |-------------------------------------->|
|                     |                  | (15) Presentation Result              |
|                     |                  |<--------------------------------------|
|------- Cross device path ------------------------------------------------------|
| (16) Display QR Code|                  |                    |                  |
|<--------------------|                  |                    |                  |
| (17) Scan QR Code   |                  |                    |                  |
|----------------------------------------------------------->>|                  |
|                     |                  |                    | (18) Presentation Response (vp_token)
|                     |                  |                    |----------------->|
|                     +---- Loop ------------------------------------------------+
|                     | (19) Poll for authorization response                     |
|                     |----------------->|                                       |
|                     |                  | (20) Get Presentation Result          |
|                     |                  |<------------------------------------->|
|                     |                  | (21) Pending                          |
|                     |                  |<--------------------------------------|
|                     | (22) Pending     |                                       |
|                     |<-----------------|                                       |
|                     +---- Break when presentation done ------------------------|
|                     |                  | (23) Presentation Result              |
|                     |                  |<--------------------------------------|
+================================================================================+
|                     |                  |
|  Note: Assuming the happy path. In case of error, error responses are sent back accordingly.
|                     |                  |
|                     | (24) Code        |
|                     |<-----------------|
|                     | (25) Token Request w/ code
|                     |----------------->|
|                     | (26) Tokens      |
|                     |<-----------------|
|                     |                  |
~~~

The verifier is displayed here as a separate instance, but can also be part of the authorization server. In both cases, it is transparent to the client, as the client only talks to the authorization server's Native Authorization Endpoint ({{native-authorization-endpoint}}).

1. User opens app
2. The client indicates, as part of the Native Authorization Request ({{native-auth-request}}), the following:
    - Login via digital credentials (wallet) desired
    - DC API support
    - Same device/cross device (only, if DC API is not supported)
    - a redirect URI intended for redirect from a wallet to the client in {{OpenID4VP}} same device flows (hereafter called openid4vp_redirect_uri)

          POST /native-authorization HTTP/1.1
          Host: server.example.com
          Content-Type: application/x-www-form-urlencoded

          &client_id=bb16c14c73415
          &scope=profile
          &dc_api=false
          &login_hint=wallet
          &openid4vp_flow_type=same_device
          &openid4vp_redirect_uri=https%3A%2F%2Fdeeplink.example.client

   Note that the client may collect this information from the user before making the initial request, or provide what it knows initially and respond gradually to additional requests for information from the authorization server.

3. Authorization server returns with error response (insufficient_authorization), indicating in the response body that digital credential presentation is required. The authorization server's response may look like this:

       HTTP/1.1 400 Bad Request
       Content-Type: application/json
       Cache-Control: no-store

       {
         "error": "insufficient_authorization",
         "auth_session": "ce6772f5e07bc8361572f",
         "type": "digital_credentials_required",
         "request": "openid4vp://?request_uri=..." // omitted if the authorization server cannot yet return the presentation request
       }

4. Client invokes the DC API.
5. The DC API returns the vp_token to the client
6. Client sends vp_token to the authorization server

        POST /native-authorization HTTP/1.1
        Host: server.example.com
        Content-Type: application/json

        {
          "auth_session": "ce6772f5e07bc8361572f",
          "vp_token": "....."
        }

7. The authorization server asks the verifier to verify the vp_token
8. The verifier verifies the vp_token and returns the result. The authorization server evaluates the verification result and returns either a code or an error as per {{native-response}}. Here we assume the happy path. Continue with step 24.
9. If DC API is NOT supported and in case of same device flow, the client invokes the Wallet through Deep Link.
10. Wallet presents credentials to verifier.
11. Verifier responds with a URL instructing wallet to redirect as per {{OpenID4VP}}.
12. Wallet redirects back to the client using the received URL as a Deep Link.
13. Client extracts the handle from the received URL and provides it to the authorization server.

        POST /native-authorization HTTP/1.1
        Host: server.example.com
        Content-Type: application/json

        {
          "auth_session": "ce6772f5e07bc8361572f",
          "presentation_id" : "87248924n2f"
        }

14. The authorization server uses the handle at verifier to look retrieve the presentation result.
15. The verifier responds with the presentation result which the authorization server evaluates, then returns either a code or an error as per {{native-response}}. Here we assume the happy path. Continue with step 24.
16. If DC API is NOT supported and in case of cross device, the client renders the presentation request as QR code and displays it to the user.
17. The user scans the QR code with the wallet.
18. The wallet presents the credentials to the verifier.
19. Meanwhile, client polls the authorization server for the presentation status until it receives a non-pending result.

        POST /native-authorization HTTP/1.1
        Host: server.example.com
        Content-Type: application/json

        {
          "auth_session": "ce6772f5e07bc8361572f"
        }

20. The authorization server in turn retreives the presentation status from the verifier.
21. The verifier responds to the authorization server that the presentation result is not yet ready.
22. The authorization server responds to the client that the presentation result is not yet ready.

        HTTP/1.1 400 Bad Request
        Content-Type: application/json
        Cache-Control: no-store

        {
          "error": "insufficient_authorization",
          "status": "pending",
          "auth_session": "ce6772f5e07bc8361572f"
        }

23. Once the presentation is done, the verifier returns the result to the authorization server, which then evaluates the verification result and returns either a code or an error as per {{native-response}}, breaking the loop. Here we assume the happy path. Continue with step 24.
24. The authorization server returns an authorization code to the client as per {{native-response}}.
25. The client redeems the code for an access token as per {{token-request}}.
26. The authorization server responds to the token request as per {{token-request}}.

### RAR & Transaction Data

{{OpenID4VP}} supports transaction data, which is additional data to be signed and presented alongside the requested credentials. This can be mapped to RAR {{RFC9396}}. The following diagram depicts a wallet flow incorporating RAR. Details of how the wallet is invoked or how the presentation result reaches the authorization server are omitted for simplicity. Refer to {{digital-credentials}} for details.

~~~ ascii-art
First Party Client        AS              Wallet/DC API       Verifier
------------------        ---              --------------       --------
|                          |                    |                  |
| (1) Native Authorization Request w/ RAR       |                  |
|------------------------->|                    |                  |
|                          | (2) Create Presentation request (OIDC4VP) w/ tx_data
|                          |-------------------------------------->|
|                          | (3) Presentation request (OIDC4VP)    |
|                          |<--------------------------------------|
| (4) Presentation request (OIDC4VP)            |                  |
|<-------------------------|                    |                  |
| (5) Presentation Request (OIDC4VP)            |                  |
|---------------------------------------------->|                  |
|                          |                    | (6) Presentation Response (vp_token)
|                          |                    |----------------->|
|                          | (7) Presentation Result               |
|                          |<--------------------------------------|
| (8) code                |
|<-------------------------|
| (9) Token Request w/ code
|------------------------->|
| (10) Token Response w/ RAR
|<-------------------------|
~~~

1. The client initiates the OAuth request, including RAR.
2. The authorization server processes that RAR from the request and creates a transaction data object from it. The authorization server then sends a request including the transaction data to the verifier to create a presentation request.
3. The verifier responds with the presentation request.
4. The authorization server responds to client with the presentation request.
5. The client invokes the wallet. How exactly the wallet is invoked is omitted for simplicity. See Usage with Digital Credentials {{digital-credentials}} for details.
6. The wallet creates a vp_token including the requested transaction data and sends it to the verifier.
7. The verifier verifies the vp_token and eventually the authorization server learns about the result. How the authorization server learns about the result is omitted for simplicity. See Usage with Digital Credentials {{digital-credentials}} for details. The authorization server then evaluates the result, especially the transaction data.
8. The authorization server returns an authorization code to the client as per {{native-response}}.
9. The client redeems the code for an access token as per {{token-request}}.
10. The authorization server responds to the token request as per {{token-request}}. It also includes the authorization_details.

        HTTP/1.1 200 OK
        Content-Type: application/json
        Cache-Control: no-store

        {
          "access_token": "2YotnFZFEjr1zCsicMWpAA",
          "token_type": "Bearer",
          "expires_in": 3600,
          "authorization_details": [
            {
              "type": "account_information",
              "access": {
                "accounts": [
                  {
                    "iban": "DE2310010010123456789"
                  },
                  {
                    "maskedPan": "123456xxxxxx1234"
                  }
                ],
                "balances": [
                  {
                    "iban": "DE2310010010123456789"
                  }
                ],
                "transactions": [
                  {
                    "iban": "DE2310010010123456789"
                  },
                  {
                    "maskedPan": "123456xxxxxx1234"
                  }
                ]
              },
              "recurringIndicator": true
            }
          ]
        }

# Document History

-00

* Document creation


# Acknowledgments
{:numbered="false"}

The authors would like to thank the attendees of IETFs 123 & 124 in which this was discussed, as well as the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification: George Fletcher, Arndt Schwenkshuster, Filip Skokan.
