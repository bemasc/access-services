---
title: "HTTP Access Service Description Objects"
abbrev: "HTTP Access Service Descriptions"
category: std

docname: draft-schwartz-masque-access-descriptions-latest
v: 3
area: "Transport"
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
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

HTTP proxies can operate several different kinds of access services.  This specification provides a format for identifying a collection of such services.


--- middle

# Introduction

In HTTP/1.1, forward proxy service was originally defined in two ways: absolute-uri form (encrypted at most hop-by-hop), and HTTP CONNECT (potentially encrypted end-to-end).  Both of these services were effectively origin-scoped: the access service was a property of the origin, not associated with any particular path.

Recently, a variety of new standardized proxy-like services have emerged for HTTP.  These new services are defined by a URI template or path, allowing distinct instances of the same service type to be served by a single origin.  These services include:

* DNS over HTTPS
* CONNECT-UDP
* CONNECT-IP
* Oblivious HTTP

This specification provides a unified format for describing a collection of such access services, and a mechanism for reaching such services when the initial information contains only an HTTP origin.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Format

An access service collection is defined by a JSON dictionary containing keys specified in the corresponding registry ({{iana}}).  Inclusion of each key is OPTIONAL.

The "dns", "udp", and "ip" keys are each defined to hold a JSON dictionary containing the key "template" with a value that is a URI template suitable for configuring DNS over HTTPS, CONNECT-UDP, or CONNECT-IP, respectively.

The "ohai" key contains a dictionary with either or both of these keys:

* "proxy", containing a dictionary with a "uri" key indicating the Oblivious Proxy Resource.
* "request", containing a dictionary with a "uri" key indicating the Oblivious Request Resource and a "key" key conveying its KeyConfig in base64.

For example, a description making use of all four initial keys might look like:

~~~JSON
{
  "dns": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "udp": {
    "template": "https://proxy.example.org:4443/masque{?target_host,target_port}"
  },
  "ip": {
    "template": "https://proxy.example.org:4443/masque{?target,ip_proto}"
  },
  "ohai": {
    "proxy": {
      "uri": "https://proxy.example.org/ohai/"
    },
    "request": {
      "uri": "https://example.com/ohai/",
      "key": "(KeyConfig in Base64)"
    }
  }
}
~~~

# Discovery from an Origin

In cases where the HTTP access service is identified only by an origin (e.g. when configured as a Secure Web Proxy), operators can publish an associated access service collection at the path "/.well-known/access-services", with the Content-Type "application/json".

Clients MAY fetch this access description and use the indicated services (in addition to any origin-scoped services) automatically.  Clients MUST refresh the description periodically in accordance with its indicated HTTP cache lifetime.


# Security Considerations

TODO Security


# IANA Considerations {#iana}

IANA is requested to open a Specification Required registry entitled "HTTP Access Service Descriptors", with the following initial contents:

| Key  | Specification   |
|------|-----------------|
| dns  | (This document) |
| udp  | (This document) |
| ip   | (This document) |
| ohai | (This document) |

IANA is requested to add the following entry to the "Well-Known URIs" registry

| URI Suffix      | Change Controller | Reference       | Status    | Related Information |
| --------------- | ----------------- | --------------- | --------- | ------------------- |
| access-services | IETF              | (This document) | permanent | Sub-registry at (link)      |

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
