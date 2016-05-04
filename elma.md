# Introduction

The **Entity Lookup Microservice API** (ELMA) is an API to look up and search
for entities via HTTP requests. An **entity** within the scope of this
specification is any object identified by an URI and referred to by one name
per language. In particular the API covers two use cases:

[entity search]
  : searching with a query string to get a list of possibly matching entities, 
    each with label, descriptions, and entity URIs. This use case is also known 
    as search suggestions, autocomplete, or type-ahead.

[entity lookup]
  : looking up an entity by its URI to get its name and possibly other properties.

ELMA services can be used for instance for semantic tagging and named-entity
resolution by selecting identified concepts from knowledge organization
systems.

## Status of this document

ELMA is currently being developed as part of project [coli-conc] and
[DINI AG KIM Normdaten].  The ELMA specification is hosted at
<http://gbv.github.io/elma/> in the public GitHub repository
<https://github.com/gbv/elma>. Feedback, for instance 
[in form of GitHub issues](https://github.com/gbv/elma/issues) is appreciated!

This document can be shared freely under the terms of
[CC-BY-SA](http://creativecommons.org/licenses/by-sa/3.0/).

[coli-conc]: https://coli-conc.gbv.de
[DINI AG KIM Normdaten]: https://wiki.dnb.de/display/DINIAGKIM/Normdaten+Gruppe

## Conformance requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119].

# General requirements

## Base URLs
[lookup base URL]: #base-urls
[search base URL]: #base-urls

An ELMA microservice MUST provide a **base URL** for [entity lookup] and a base
URL for [entity search]. Both base URLs can be queried by HTTP GET and HTTP
OPTIONS requests ([RFC 7231]) and both MAY be identical. The base URLs MAY
contain fixed query parameters but it MUST NOT contain any of the reserved
parameter names "uri", "search", "callback", "language", and "unique".

<div class="example">
An ELMA microservice could 

* be served via base URL <http://example.org/> for both lookup and search

* provide two base URLs, e.g. <http://example.org/?entity=lookup> and <http://example.org/?entity=search>

The base URL <http://example.org/lookup?search=concept> is not valid because it
contains the reserved parameter name "search".
</div>

## HTTP headers

An ELMA client SHOULD include the following HTTP request headers in every
request:

User-Agent
  : an approriate client name and version number

Accept
  : the value "application/json"

An ELMA microservice MUST include the following HTTP response header in every
response to a HTTP GET response:

Content-Type
  : the value "application/json" or "application/json; charset=utf-8"

## Response format

The response body of an ELMA microservice response, if given, MUST always be a
syntacticaly valid JSON document ([RFC 4627]), or a JSON document wrapped in a
JavaScript function call if [JSONP] is supported. This also applies to [error
responses].

## Cross-Origin Resource Sharing

For Cross-Origin Resource Sharing ([CORS]) an ELMA client MUST include the 
following HTTP request headers:

Origin
  : where the cross-origin request originates from

Access-Control-Request-Method
  : the HTTP verb "GET" (only if the request is an OPTIONS preflight request)

In response to a CORS request an ELMA microservice MUST include the following
HTTP response headers:

Access-Control-Allow-Origin
  : the value "\*" or another approriate value as defined by [CORS]

Access-Control-Allow-Headers
  : a comma-separated list of usable request headers as defined by [CORS].
    MUST at least contain the value "Content-Type"

Note that ELMA clients do not need to respect CORS rules. CORS preflight
requests in browsers can be avoided by omitting the request header "Accept".

## JSONP

An ELMA microservice MAY support JSONP by respecting the additional query
parameter "callback".  If a HTTP request URL contains this query parameter and
if its value only contain alphanumeric characters and underscores, the HTTP
response is modified as following: 

* The value of response header "Content-Type" is changed to
  "application/javascript" or "application/javascript; charset=utf-8".

* The response body is wrapped in a JavaScript call to the given callback
  function.

## Error responses
[error response]: #error-responses

Error responses with HTTP status code 4xx (client error) or 5xx (server error)
SHOULD be returned with a JSON object response body having the following
fields:

code
  : the HTTP status error code (REQUIRED)

error
  : a custom error code (REQUIRED). 
    Allowed characters include `a-z`, `0-9` and underscore (`_`).

message
  : a human-readable error message, intended to be shown to an end user
    (OPTIONAL)

error_description
  : a human-readable error description, intended for a developer, 
    not an end user (OPTIONAL)

error_uri
  : an URL of a human-readable web page with information about the error
    (OPTIONAL)

The response header "Content-Language" MUST indicate the language of
human-readable error message and description. The request header
"Accept-Language" SHOULD be supported if error messages and descriptions are
available in multiple languages.

<div class="example">
An ELMA microservice may implement rate limiting and issue an error if
the number of requests in a given span of time has been exhausted:

~~~json
{
  "code": 429,
  "error": "too_many_requests",
  "message": "Too many requests",
  "description": "Please slow down your request rate!",
  "error_uri": "http://example.org/api-policy" 
}
~~~
</div>

# Entity search
[entity search]: #entity-search

Entities can be searched in a ELMA microservice by an HTTP GET request at its
[search base URL] and query parameter "search". A client SHOULD further include
the **Accept-Language** request header in search requests.

The response of an ELMA microservice entity search MUST be a JSON array of the
following four values:
 
query string
  : the (possibly normalized) query as given with query parameter "search"

labels
  : an array of entity labels which MUST be non-empty strings

descriptions
  : an array of additional entity descriptions which MAY be empty strings

identifiers
  : an array of distinct entity URIs, each given as string

The order of entities is expected to result from relevance ranking. The query
parameter "search" SHOULD NOT be expected to follow a specific search syntax or
query language but a plain string.

The response of an entity search MUST include a **Content-Language** header
giving the language of labels and descriptions in the response. The language
SHOULD be influenced by Accept-Language response field and an optional request 
parameter "language".

<div class="note">
The result format is fully compatible with [OpenSearch Suggestions], so every
search base URL of an ELMA microservice can also be used as OpenSarch
Suggestions service.
</div>

<div class="example">
Request:

<http://example.org/?search=Go%CC%88the> ("Göthe" with Unicode decomposed characters)

Response (Unicode normalization form C, relevance ranking):

    Content-Lanuage: en
    Content-Type: application/json

~~~json
[
  "Göthe",
  [
   "Johann Wolfgang von Goethe",
   "Göthe Emanuel Hedlund",
   "Göthe Grefbo"
  ],[
   "German writer, artist, and politician (1749-1832)",
   "Speed skater (1918-2003)",
   "Swedish Actor (1921-1991)"
  ],[
   "http://www.wikidata.org/entity/Q5879",
   "http://www.wikidata.org/entity/Q3431981",
   "http://www.wikidata.org/entity/Q2054509"
  ]
]
~~~
</div>

# Entity lookup
[entity lookup]: #entity-lookup

Entities can be looked up in ELMA microservice by an HTTP GET request at its
[lookup base URL] and query parameter "uri". The parameter value MUST be a
syntactically correct IRI ([RFC 3987]). An [error response] with status code
422 SHOULD be returned if parameter "uri" contains is syntactically invalid, if
it is repeated, or if both parameters "uri" and "search" are given with a
non-empty value.

The response to an entity lookup request MUST be an empty JSON array if no
entity was found with the given URI or a JSON array containing one member
otherwise. The response code MUST NOT be 404 if the entity was not found.

A found entity MUST be expressed as JSON object that conforms to the following
restrictions:

* The entity object MUST contain a key "uri" with the entity URI as value

* The entity object SHOULD contain a key "prefLabel". The value of this object
  MUST be a JSON object with [RFC 3066] language tags as keys and entity names 
  as string values. The choice of language tags SHOULD be influencable with the
  Accept-Language HTTP header and an optional request parameter "language".

* The "prefLabel" object MAY contain additional keys ending with the character
  "-" to indicate existence of additional labels. These keys can also be
  referred to as language ranges. Values of these keys SHOULD be ignored as they
  only act as boolean existence flags.

Applications MAY include additional fields as long as their name starts with an
uppercase letter.  Additional fields starting with lowercase letters may be
specified in an extended version of ELMA.

<div class="note">
The response format is aligned with [JSKOS data format for Knowledge
Organization Systems](https://gbv.github.io/jskos/) but neither ELMA services
nor ELMA clients need to know about details of JSKOS.
</div>

<div class="example">
Request:

<http://example.org/api?uri=http%3A%2F%2Fwww.wikidata.org%2Fentity%2FQ3431981>

Response:

    Content-Type: application/json

~~~json
[
  {
    "uri": "http://www.wikidata.org/entity/Q5879",
    "prefLabel": {
      "de": "Johann Wolfgang von Goethe",
      "en": "Johann Wolfgang von Goethe",
      "ja": "ヨハン・ヴォルフガング・フォン・ゲーテ"
    }
  }
]
~~~
</div>

All URIs included in an [entity search] responses MUST result in a non-empty
lookup response of the same ELMA microservice. The entity label in field
"prefLabel" SHOULD be identical to the label included in an entity search
response for the given language tag in response header Content-Language.

# References

## Normative references {.unnumbered}

* S. Bradner: *Key words for use in RFCs to Indicate Requirement Levels*.
  RFC 2119, March 1997. <https://tools.ietf.org/html/rfc2119>

* D. Crockford: *The application/json Media Type for JavaScript Object Notation (JSON)*.
  RFC 4627, July 2006. <https://tools.ietf.org/html/rfc4627>

* M. Dürst, M. Suignard: *Internationalized Resource Identifiers (IRIs)*.
  RFC 3987, January 2005. <https://tools.ietf.org/html/rfc3987>

* R. Fielding, J. Reschke: *Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content*
  RFC 7231, June 2014. <https://tools.ietf.org/html/rfc7231>

* A. Phillips, M. Davis: *Tags for Identifying Languages*.
  RFC 3066, September 2006. <https://tools.ietf.org/html/rfc3066>

* A. van Kesteren *Cross-Origin Resource Sharing*
  W3C Recommendation, January 2014. <http://www.w3.org/TR/cors/>

[RFC 2119]: https://tools.ietf.org/html/rfc2119
[RFC 4627]: https://tools.ietf.org/html/rfc4627
[RFC 3987]: https://tools.ietf.org/html/rfc3987
[RFC 3066]: https://tools.ietf.org/html/rfc3066
[RFC 7231]: https://tools.ietf.org/html/rfc7231
[CORS]: http://www.w3.org/TR/cors/

## Informative references {.unnumbered}

* M. Davis, K. Whistler: *Unicode Normalization Forms*.
  Unicode Standard Annex #15.
  <http://www.unicode.org/reports/tr15/>

* A. Phillips, M. Davis: *Tags for Identifying Languages*.
  RFC 4646, October 2005.
  <http://tools.ietf.org/html/rfc4646>

* A. Phillips, M. Davis: *Matching of Language Tags*.
  RFC 4647, September 2006.
  <http://tools.ietf.org/html/rfc4647>

* D. Clintin: *OpenSearch Suggestions 1.0*. 2005.
  <http://www.opensearch.org/Specifications/OpenSearch/Extensions/Suggestions>

* J. Voß: *JSKOS data format for Knowledge Organization Systems*.
  February 2016. <https://gbv.github.io/jskos/>

[RFC 4646]: http://tools.ietf.org/html/rfc4646
[RFC 4647]: http://tools.ietf.org/html/rfc4647
[OpenSearch Suggestions]: http://www.opensearch.org/Specifications/OpenSearch/Extensions/Suggestions/1.0

# Changelog {.unnumbered}

### 0.0.3 (2016-05-04) {.unnumbered}

* Changed query paramater "q" to "search"

### 0.0.2 (2016-05-02) {.unnumbered}

* Allow language ranges in response format

