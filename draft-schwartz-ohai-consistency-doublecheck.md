---
title: "Key Consistency for Oblivious HTTP by Double-Checking"
abbrev: "Key Consistency Double Check"
category: std

docname: draft-schwartz-ohai-consistency-doublecheck-latest
v: 3
area: sec
workgroup: ohai
keyword: Internet-Draft
venue:
  github: "bemasc/access-services"

author:
 -
    fullname: Benjamin M. Schwartz
    organization: Google LLC
    email: bemasc@google.com

normative:

informative:


--- abstract

The assurances provided by Oblivious HTTP depend on the client's ability to verify that it is using the same Request Resource and public keys as many other users.  This specification defines a protocol to enable this verification.

--- middle

# Introduction

Oblivious HTTP {{!I-D.ietf-ohai-ohttp}} presumes at least three parties to each exchange: the client, the proxy, and the target (formally, the Oblivious Request Resource).  When used properly, Oblivious HTTP enables the client to send requests to the target in such a way that the target cannot tell whether two requests came from the same client and the proxy cannot see the contents of the requests.

Oblivious HTTP's threat model assumes that at least one of the proxy and the target is acting properly, i.e. complying with the protocol and keeping certain information confidential.  If either proxy or target misbehaves, the only effect must be a denial of service.

In order for these security guarantees to hold, several preconditions must be met:

1. The client must be one of many users who might be using the proxy.  Otherwise, use of the proxy reveals the user's identity to the target.
2. The client must hold an authentic KeyConfig for the target.  Otherwise, they could be speaking to the proxy, impersonating the target.
3. All users of this proxy must be equally likely to use this URI and KeyConfig for this target, regardless of their prior activity.  Otherwise, the encrypted request identifies the user to the target.
4. (optional) The target must not learn the IP addresses of the clients, collectively.  Otherwise, the target might be able to deanonymize requests by correlating them with external information about the clients.

This specification defines behaviors for the client, proxy, and target that achieve preconditions 2-4.  (This specification does not address precondition 1.)

This draft is an instantiation of the "Single Proxy Discovery" architecture for key consistency, defined in {{Section 4.2 of ?I-D.wood-key-consistency}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Overview

In the Key Consistency Double-Check procedure, the Client emits two requests: one to the Proxy, and one through the Proxy to the Target.  The Proxy will forward the first request to the Target if the response is not in cache.

~~~
                +--------+       +-------+      +--------+
                |        |<=====>|       |<---->|        |
                | Client |       | Proxy |      | Target |
                |        |<====================>|        |
                +--------+       +-------+      +--------+
~~~
{: title="Overview of Key-Consistency Double-Check"}

The proxy caches the response, ensuring that all clients share it during its freshness lifetime.  The client checks this against the authenticated response from the Target, preventing forgeries.

# Requirements

## Oblivious Request Resource

The Oblivious Request Resource MUST publish an Access Description ((!I-D.schwartz-masque-access-desriptions)) available over HTTP/3, containing the `ohttp.request` key, e.g.:

~~~JSON
{
  "ohttp": {
    "request": {
      "uri": "https://example.com/ohttp/",
      "key": "(KeyConfig in Base64)"
    }
  }
}
~~~

The Oblivious Request Resource MUST include a "strong validator" ETag ({{Section 2 of !RFC7232}}) in any response to a GET request for this access description, and MUST support the "If-Match" HTTP request header ({{Section 3 of !RFC7232}}).  The response MUST indicate "Cache-Control: public, no-transform, s-maxage=(...), immutable" {{!I-D.ietf-httpbis-cache}}{{!RFC8246}}.  For efficiency reasons, the max age SHOULD be at least 60 seconds.

If this Access Description changes, and the resource receives a request whose "If-Match" header identifies a previously served version that has not yet expired, it MUST return a success response containing the previous version.  This response MAY indicate "Cache-Control: private".

## Oblivious Proxy {#proxy}

The Oblivious Proxy MUST publish an Access Description that includes the `ohttp.proxy` and `udp` keys, indicating support for CONNECT-UDP {{!I-D.ietf-masque-connect-udp}}.  It SHOULD also contain the `dns` key, indicating support for DNS over HTTPS {{!RFC8484}}, to enable the use of HTTPS records.

~~~JSON
{
  "dns": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "udp": {
    "template":
        "https://proxy.example.org/masque{?target_host,target_port}"
  },
  "ohttp": {
    "proxy": {
      "template": "https://proxy.example.org/ohttp{?request_uri}"
    }
  }
}
~~~
{: title="Example Proxy Access Description"}

The Oblivious Proxy Resources MUST allow use of the GET method to retrieve small JSON responses, and SHOULD make ample cache space available in order to cache Access Descriptions.  Each proxy instance (as defined by its external-facing network interface) SHOULD share cache state among all clients to ensure that they use the same Access Descriptions for each Oblivious Request Resource.

Oblivious Proxies MUST preserve the ETag response header on cached responses, and MUST add an Age header ({{!I-D.ietf-httpbis-cache-19, Section 5.1}}) to all proxied responses.  Oblivious Proxies MUST respect the "Cache-Control: immutable" directive, never revalidating these cached entries, and MUST NOT accept PUSH_PROMISE frames from the target.

Proxies SHOULD employ defenses against malicious attempts to fill the cache.  Some possible defenses include:

* Rate-limiting each client's use of GET requests.
* Prioritizing preservation of cache entries that have been served to many clients, if eviction is required.

Oblivious Proxies that are not intended for general-purpose proxy usage MAY impose strict transfer limits or rate limits on HTTP CONNECT and CONNECT-UDP usage.

## Client

The Client is assumed to know an "https" URI of an Oblivious Request Resource's Access Description.  To use that Request Resource, it MUST perform the following "double-check" procedure:

1. Send a GET request to the Oblivious Proxy's template with `request_uri` set to the Access Description URI.
2. Record the response (A).
3. Fetch the Access Description URI from its origin via CONNECT-UDP, with "If-Match" set to response A's ETag.
4. Record the response (B).
5. Check that responses A and B were successful and the contents are identical, otherwise fail.
6. Check that response A's "Cache-Control" values indicates "public" and "immutable".

This procedure ensures that the Access Description is authentic and will be shared by all users of this proxy.  Once response A or B expires, the client MUST refresh it before continuing to use this Access Description, and repeat the "double-check" process if a response changes.

# Example: Oblivious DoH

The client has been configured with an Oblivious DoH server and an Oblivious Proxy.

The Oblivious DoH server is identified by an Access Description at "https://doh.example.com/config.json" with the following contents:

~~~JSON
{
  "dns": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "ohttp": {
    "request": {
      "uri": "https://example.com/ohttp/",
      "key": "(KeyConfig in Base64)"
    }
  }
}
~~~

The Oblivious Proxy is identified as "proxy.example.org", which implies an Access Description at "https://proxy.example.org/.well-known/access-services".  This resource's contents are:

~~~JSON
{
  "dns": {
    "template": "https://proxy.example.org/dns-query{?dns}",
  },
  "udp": {
    "template":
        "https://proxy.example.org/masque{?target_host,target_port}"
  },
  "ohttp": {
    "proxy": {
      "template": "https://proxy.example.org/ohttp{?request_uri}"
    }
  }
}
~~~

The following exchanges then occur between the client and the proxy:

~~~
HEADERS
:method = GET
:scheme = https
:authority = proxy.example.org
:path = /ohttp?request_uri=https%3A%2F%2Fdoh.example.com%2Fconfig.json
accept: application/json

                              HEADERS
                              :status = 200
                              cache-control: public, immutable, \
                                  no-transform, s-maxage=86400
                              age: 80000
                              etag: ABCD1234
                              content-type: application/json
                              [Access Description contents here]

HEADERS
:method = CONNECT
:protocol = connect-udp
:scheme = https
:authority = proxy.example.org
:path = /masque?target_host=doh.example.com,target_port=443
capsule-protocol = ?1

                              HEADERS
                              :status = 200
                              capsule-protocol = ?1
~~~

The client now has a CONNECT-UDP tunnel to `doh.example.com`, over which it performs the following request using HTTP/3:

~~~
HEADERS
:method = GET
:scheme = https
:authority = doh.example.com
:path = /config.json
if-match = ABCD1234

                              HEADERS
                              :status = 200
                              cache-control: public, immutable, \
                                  no-transform, s-maxage=86400
                              etag: ABCD1234
                              content-type: application/json
                              [Access Description contents here]
~~~

Having successfully fetched the Access Description from both locations, the client confirms that:

* The responses are identical.
* The cache-control response from the proxy contained the "public" and "immutable" directives.

The client can now use the KeyConfig in this Access Description to reach the Oblivious DoH server, by forming Binary HTTP requests for "https://doh.example.com/dns-query" and delivering the encapsulated requests to "https://example.com/ohttp/" via the proxy.

# Security Considerations

## In scope

A malicious proxy could attempt to learn the contents of the oblivious request by forging an inauthentic KeyConfig for the Request Resource.  This is prevented by the client's requirement that the KeyConfig be served to it by the origin over HTTPS ({{client}}).

A malicious target could attempt to link multiple requests together by issuing each user a unique, persistent KeyConfig.  This attack is prevented by the client's requirement that the KeyConfig be fresh according to the proxy's cache ({{client}}).

A malicious target could attempt to rotate its entry in the proxy's cache in several ways:

* Using HTTP PUSH_PROMISE frames.  This attack is prevented by disabling PUSH_PROMISE at the proxy ({{proxy}}).
* By also acting as a client and sending requests designed to replace the Access Description in the cache before it expires:
  - By sending requests with a "Cache-Control: no-cache" or similar directive.  It is prevented by the response's "Cache-Control: public, immutable" directives, which are verified by the client ({{client}}).
  - By filling the cache with new entries, causing its previous Access Description to be evicted.  {{proxy}} describes some possible mitigations.

A malicious client could use the proxy to send abusive traffic to any destination on the internet.  Abuse concerns can be mitigated by imposing a rate limit at the proxy ({{proxy}}).

## Out of scope

This specification assumes that the client starts with identities of the proxy and target that are authentic and widely shared.  If these identities are inauthentic, or are unique to the user, then the security goals of this specification are not achieved.

This specification assumes that at most a small fraction of clients are acting on behalf of a malicious target.  If a large fraction of the clients are malicious, they could conspire to flood the proxy cache with entries that seem popular, leading to rapid eviction of the malicious target's Access Descriptions.  Similar concerns apply if a malicious target can compel naive clients to fetch a very large number of Access Descriptions.

# IANA Considerations {#iana}

IANA is requested to open a Specification Required registry entitled "HTTP Access Service Descriptors", with the following initial contents:

| Key   | Specification   |
|-------|-----------------|
| dns   | (This document) |
| udp   | (This document) |
| ip    | (This document) |
| ohttp | (This document) |

IANA is requested to add the following entry to the "Well-Known URIs" registry

| URI Suffix      | Change Controller | Reference       | Status    | Related Information |
| --------------- | ----------------- | --------------- | --------- | ------------------- |
| access-services | IETF              | (This document) | permanent | Sub-registry at (link)      |

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
