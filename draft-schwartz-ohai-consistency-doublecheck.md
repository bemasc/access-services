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

The assurances provided by Oblivious HTTP depend on the client's ability to verify that it is using the same Request Resource and KeyConfig as many other users.  This specification defines a protocol to enable this verification.

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

In the Key Consistency Double-Check procedure, the Client emits two HTTP GET requests: one to the Proxy, and one through the Proxy to the Target.  The Proxy will forward the first request to the Target if the response is not in cache.

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

The Oblivious Request Resource MUST publish an Access Description {{!I-D.schwartz-masque-access-descriptions}} containing the "ohttp.request" key, e.g.:

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

This resource MUST be available over HTTP/3 {{!RFC9114}}, so that it can be accessed via the proxy's CONNECT-UDP service (see {{proxy}}).

The Oblivious Request Resource MUST include a "strong validator" ETag ({{Section 2 of !RFC7232}}) in any response to a GET request for this access description, and MUST support the "If-Match" HTTP request header ({{Section 3 of !RFC7232}}).  The response MUST indicate "Cache-Control: public, no-transform, s-maxage=(...), immutable" {{!RFC9111}}{{!RFC8246}}.  For efficiency reasons, the max age SHOULD be at least 60 seconds, and preferably much longer.

If this Access Description changes, and the resource receives a request whose "If-Match" header identifies a previously served version that has not yet expired, it MUST return a success response containing the previous version.  This response MAY indicate "Cache-Control: private".

## Oblivious Proxy {#proxy}

The Oblivious Proxy MUST publish an Access Description that includes the "ohttp.proxy" and "udp" keys, indicating support for CONNECT-UDP {{!I-D.ietf-masque-connect-udp}}.  It SHOULD also contain the "dns" key, indicating support for DNS over HTTPS {{!RFC8484}}, to enable the use of HTTPS records {{?SVCB=I-D.draft-ietf-dnsop-svcb-https}} with CONNECT-UDP.

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

The Oblivious Proxy Resources MUST allow use of the GET method to retrieve small JSON responses, and SHOULD make ample cache space available in order to avoid eviction of Access Descriptions.  The proxy SHOULD share cache state among all clients, to ensure that they use the same Access Descriptions for each Oblivious Request Resource.  If the cache must be partitioned for architectural or performance reasons, operators SHOULD keep the number of users in each partition as large as possible.

Oblivious Proxies MUST preserve the ETag response header on cached responses, and MUST add an Age header ({{!RFC9111, Section 5.1}}) to all proxied responses.  Oblivious Proxies MUST respect the "Cache-Control: immutable" directive, and MUST NOT revalidate fresh immutable cache entries in response to any incoming requests.  (Note that this is different from the general recommendation in {{Section 2.1 of !RFC8246}}).  Oblivious Proxies also MUST NOT accept PUSH_PROMISE frames from the target.

Proxies SHOULD employ defenses against malicious attempts to fill the cache.  Some possible defenses include:

* Rate-limiting each client's use of GET requests.
* Prioritizing preservation of cache entries that have been served to many clients, if eviction is required.

Oblivious Proxies that are not intended for general-purpose proxy usage MAY impose strict transfer limits or rate limits on HTTP CONNECT and CONNECT-UDP usage.

If the proxy offers a DNS over HTTPS resolver, it MUST NOT enable EDNS Client Subnet support {{!RFC7871}}.

## Client

The Client is assumed to know an "https" URI of an Oblivious Request Resource's Access Description.  To use that Request Resource, it MUST perform the following "double-check" procedure:

1. Send a GET request to the Oblivious Proxy's template (`ohttp.proxy.template`) with `request_uri` set to the Access Description URI.
1. Record the response (A).
1. Check that response A's "Cache-Control" values indicates "public" and "immutable".
1. Establish a CONNECT-UDP tunnel through the proxy to the Access Description URI's origin.
1. Fetch the Access Description URI through this tunnel, using a GET request with "If-Match" set to response A's ETag.
1. Record the response (B).
1. Check that responses A and B were successful and the contents are identical, otherwise fail.

This procedure ensures that the Access Description is authentic and will be shared by all users of this proxy.  Once response A or B expires, the client MUST refresh it before continuing to use this Access Description, and MUST repeat the "double-check" process if either response changes.

Clients MUST perform each fetch to the origin (step 4) as a fully isolated request.  Any state related to this origin (e.g. cached DNS records, CONNECT-UDP tunnels, QUIC transport state, TLS session tickets, HTTP cookies) MUST NOT be shared with prior or subsequent requests.

# Example: Oblivious DoH

In this example, the client has been configured with an Oblivious DoH server and an Oblivious Proxy.  The Oblivious DoH server is identified by an Access Description at "https://doh.example.com/config.json" with the following contents:

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

The client now has a CONNECT-UDP tunnel to `doh.example.com`, over which it performs the following GET request using HTTP/3:

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

# Performance Implications

## Latency

Suppose that the client-proxy Round-Trip Time (RTT) is `A`, and the proxy-target RTT is `B`.  Suppose additionally that the client has a persistent connection to the proxy that is already running.  Then the procedure described in {{client}} requires:

* `A` for the GET request to the proxy
  - `+B` if the requested Access Description is not in cache
  - `+B` if the proxy does not have a TLS session ticket for the target
* `A` for the CONNECT-UDP request to the proxy
* `A + B` for the QUIC handshake to the target
* `A + B` for the GET request to the target

This is a total of `4A + 4B` in the worst case.  However, clients can reduce the latency by issuing the requests to the proxy in parallel, and by using CONNECT-UDP's "false start" support.  The target can also optimize performance, by issuing long-lived TLS session tickets.  With these optimizations, the expected total time is `2A + 2B`.

This procedure only needs to be repeated if the Access Description has expired.  Access Descriptions do not need to change frequently, so a cache lifetime of 1 day may be suitable.  Clients MAY perform this procedure in advance of an expiration to avoid a delay.

## Thundering Herds

All clients of the same proxy and target will have locally cached Access Descriptions with the same expiration time.  When this entry expires, all active clients will send refresh GET requests to the proxy at their next request.  Proxies SHOULD use "request coalescing" to avoid duplicate cache-refresh requests to the target.

If the Access Description has changed, these clients will initiate GET requests through the proxy to the target to double-check the new contents.  Proxies and targets MAY use an HTTP 503 response with a "Retry-After" header to manage load spikes.

# Security Considerations

## In scope

### Forgery

A malicious proxy could attempt to learn the contents of the oblivious request by forging an Access Description containing its own KeyConfig.  This is prevented by the client's requirement that the KeyConfig be served to it by the configured origin over HTTPS ({{client}}).

### Deanonymization

A malicious target could attempt to link multiple Oblivious HTTP requests together by issuing each user a unique, persistent KeyConfig.  This attack is prevented by the client's requirement that the KeyConfig be fresh according to the proxy's cache ({{client}}).

A malicious target could attempt to rotate its entry in the proxy's cache in several ways:

* Using HTTP PUSH_PROMISE frames.  This attack is prevented by disabling PUSH_PROMISE at the proxy ({{proxy}}).
* By also acting as a client and sending requests designed to replace the Access Description in the cache before it expires:
  - By sending GET requests with a "Cache-Control: no-cache" or similar directive.  This is prevented by the response's "Cache-Control: public, immutable" directives, which are verified by the client ({{client}}), and by the proxy's obligation to to respect these directives strictly ({{proxy}}).
  - By filling the cache with new entries, causing its previous Access Description to be evicted.  {{proxy}} describes some possible mitigations.

A malicious target could attempt to link different requests for the Access Description, in order to link the Oblivious HTTP requests that follow shortly after.  This is prevented by fully isolating each request ({{client}}), and by disabling EDNS Client Subnet ({{proxy}}).

### Abusive traffic

A malicious client could use the proxy to send abusive traffic to any destination on the internet.  Abuse concerns can be mitigated by imposing a rate limit at the proxy ({{proxy}}).

## Out of scope

This specification assumes that the client starts with identities of the proxy and target that are authentic and widely shared.  If these identities are inauthentic, or are unique to the user, then the security goals of this specification are not achieved.

This specification assumes that at most a small fraction of clients are acting on behalf of a malicious target.  If a large fraction of the clients are malicious, they could conspire to flood the proxy cache with entries that seem popular, leading to rapid eviction of the malicious target's Access Descriptions.  Similar concerns apply if a malicious target can compel naive clients to fetch a very large number of Access Descriptions.

# IANA Considerations {#iana}

No IANA action is requested.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
