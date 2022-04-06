---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "HTTP Access Service Description Objects"
abbrev: "HTTP Access Service Descriptions"
category: info

docname: draft-schwartz-masque-access-descriptions
v: 3
area: tsv
workgroup: masque
keyword: Internet-Draft
venue:
  github: bemasc/access-services

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

Recently, a variety of new standardized proxy-like services have emerged for HTTP.  These new services are defined by URI templates that produce one or more distinct paths for each service, allowing distinct instances of the same service type to be served by a single origin.  These services include:

* DNS over HTTPS
* CONNECT-UDP
* CONNECT-IP
* Oblivious HTTP

This specification provides a unified format for describing a collection of such access services, and a mechanism for reaching such services when the initial information contains only an HTTP origin.

# Format

An access service collection is defined by a JSON dictionary containing keys specified in the corresponding registry ({{iana}}).  Inclusion of each key is OPTIONAL.

The "doh", "udp", and "ip" keys are each defined to hold a JSON dictionary containing the key "template" with a value that is a URI template suitable for configuring DNS over HTTPS, CONNECT-UDP, or CONNECT-IP, respectively.  The "ohai" key contains a dictionary whose "request" key contains the URI of an Oblivious HTTP "request resource".

For example, a description making use of all four initial keys might look like:

~~~JSON
{
  "doh": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "udp": {
    "template": "https://proxy.example.org:4443/masque{?target_host,target_port}"
  },
  "ip": {
    "template": "https://proxy.example.org:4443/masque{?target,ip_proto}"
  },
  "ohai": {
    "request": "https://example.com/ohai/"
  }
}
~~~

# Discovery from an Origin

In cases where the HTTP access service is identified only by an origin (e.g. when configured as a Secure Web Proxy), operators can publish an associated access service collection at the path "/.well-known/access-services", with the Content-Type "application/json".

Clients MAY fetch this access description and use the indicated services (in addition to any origin-scoped services) automatically.  Clients MUST refresh the description periodically in accordance with its indicated HTTP cache lifetime.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations {#iana}

IANA is requested to open a Specification Required registry entitled "HTTP Access Service Descriptors", with the following initial contents:

| Key  | Specification   |
|------|-----------------|
| doh  | (This document) |
| udp  | (This document) |
| ip   | (This document) |
| ohai | (This document) |


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
