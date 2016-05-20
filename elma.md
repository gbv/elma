# Introduction

The **Entity Lookup Microservice API** (**ELMA**) is an HTTP API to look up and
search for entities. An **entity** within the scope of this specification is
any object identified by an URI and referred to by one primary label per
language. In particular the API defines two [request methods]:

[entity lookup]
  : looking up an entity by its URI to get its primary labels and possibly other
    properties.

[entity search]
  : searching with a query string to get a list of possibly matching entities, 
    each with label, descriptions, and entity URIs. This use case is also known 
    as search suggestions, autocomplete, or type-ahead.

Use cases of ELMA microservices include semantic tagging, entity linking, and
browsing in knowledge organization systems.

## Status of this document

ELMA is currently being developed as part of project [coli-conc] and [DINI AG
KIM Normdaten] as subset of JSKOS-API.  The ELMA specification is hosted at
<http://gbv.github.io/elma/> and managed in the public GitHub repository
<https://github.com/gbv/elma>. Feedback, for instance [in form of GitHub
issues](https://github.com/gbv/elma/issues), is appreciated!

This document can be shared freely under the terms of
[CC-BY-SA](http://creativecommons.org/licenses/by-sa/3.0/).

[coli-conc]: https://coli-conc.gbv.de
[DINI AG KIM Normdaten]: https://wiki.dnb.de/display/DINIAGKIM/Normdaten+Gruppe

## Conformance requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119].

# Base URLs

An ELMA microservice MUST provide a base URL for [entity lookup] and SHOULD
provide a base URL for [entity search]. Both base URLs MAY be identical. Base
URLs MAY contain fixed query parameters except any of the reserved parameter
names "uri", "search", "callback", "language", and "unique".

<div class="example">

* An ELMA microservice provides both entity lookup and entity search via
  base URL <http://example.org/>.

* An ELMA microservice provides entity lookup at
  <http://example.com/?entity=lookup> and entity search at
  <http://example.com/?entity=search>.

* An ELMA microservice provides entity lookup at 
  <http://example.org/lookup/?format=json>.

* The base URL <http://example.org/elma?search=concept> is not allowed 
  because it contains the reserved parameter name "search".

</div>

# Request and response format

## Requests

An ELMA client SHOULD include the HTTP request header "Accept" with value
"application/json" in every HTTP request. It should also include the
"Accept-Language" request header for content negotiation as specified
in [RFC 7231]. 

An ELMA microservice MUST support HTTP GET requests and it SHOULD support HTTP
OPTIONS requests and HTTP HEAD requests.

## Responses

An ELMA microservice MUST include the following HTTP response headers
in every HTTP response:

* "Content-Type" with value "application/json" or "application/json; charset=utf-8" 

* "Content-Language" in [entity search] responses and [error responses] to denote
  the language of human-readable strings in the JSON response

* "Access-Control-Allow-Origin" with the the value "\*" or another appropriate
  value, as specified by [CORS]

The response body of an ELMA microservice response to a HTTP GET request MUST
be a syntacticaly valid JSON document ([RFC 4627]). This also applies to [error
responses]. If an ELMA microservice supports JSONP, the JSON document can also
be wrapped in a JavaScript function call.  All string values in a JSON response
SHOULD be normalized to Unicode Normal Form C.

An ELMA microservice SHOULD support Cross-Origin Resource Sharing ([CORS]),
including preflight requests (HTTP request method OPTIONS and HTTP response
headers "Allow", "Access-Control-Allow-Methods", and
"Access-Control-Allow-Headers").

<div class="example">
An ELMA microservice at <http://example.org/> is accessed via HTTP GET:

    GET /?search=Paris
    Host: example.org
    Accept: application/json
    Accept-Language: de, en;q=0.5, fr;q=0.2
    Origin: http://example.com

The abbreviated response:
    
    HTTP/1.1 200 OK
    Access-Control-Allow-Origin: *
    Content-Type: application/json

    ...JSON body...

</div>

An ELMA microservice MAY support JSONP as following: if a HTTP GET request URL
contains the query parameter "callback" and if the value of this parameter only
contain alphanumeric characters and underscores, then:

* The value of response header "Content-Type" MUST be changed to
  "application/javascript" or "application/javascript; charset=utf-8".

* The response body MUST be wrapped in a JavaScript call to the given callback
  function.

## Error responses
[error response]: #error-responses

Error responses with HTTP status code 4xx (client error) or 5xx (server error)
MUST be returned with a JSON object response body. The following keys are
RECOMMENDED:

code
  : the HTTP status error code

error
  : a custom error code. 
    Allowed characters include `a-z`, `0-9` and underscore (`_`)

message
  : an optional human-readable error message, intended to be shown to 
    an end user

error_description
  : an optional human-readable error description, intended for a developer, 
    not an end user

error_uri
  : an optional URL of a human-readable web page with information about the error

The response header "Content-Language" MUST indicate the language of
human-readable error message and description. The request header
"Accept-Language" SHOULD be supported for content negotiation if error messages
and descriptions are available in multiple languages.

<div class="example">
An ELMA microservice implemented rate limiting and issued an error when
the number of requests in a given span of time had been exhausted:

~~~
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Content-Language: en

~~~
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

# Request methods

## Entity lookup
[entity lookup]: #entity-lookup

Entities can be looked up in ELMA microservice by an HTTP GET request at its
lookup base URL and query parameter "uri". The parameter value MUST be a
syntactically correct IRI ([RFC 3987]). An [error response] with status code
422 SHOULD be returned if parameter "uri" is syntactically invalid or, if it is
repeated.

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

Applications MAY include additional fields in uppercase letters or conforming to
JSKOS data format.

<div class="note">
The response format is aligned with [JSKOS data format for Knowledge
Organization Systems](https://gbv.github.io/jskos/) but neither ELMA services
nor ELMA clients need to know about details of JSKOS.
</div>

<div class="note">
The language tag "und" can be used for labels of unknown or unspecified language.
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
      "ja": "ヨハン・ヴォルフガング・フォン・ゲーテ",
      "-": "..."
    }
  }
]
~~~
</div>

All URIs included in an [entity search] responses MUST result in a non-empty
lookup response of the same ELMA microservice. 


## Entity search
[entity search]: #entity-search

An ELMA microservice with entity search base URL, in response to a HTTP GET
request at this URL with query parameter "search", MUST return a list of
matching entities as JSON array of the following four values:
 
query string
  : the (possibly normalized) query as given with query parameter "search"

labels
  : an array of entity labels which MUST be non-empty strings

descriptions
  : an array of additional entity descriptions which MAY be empty strings

identifiers
  : an array of distinct entity URIs, each given as string

<div class="note">
The result format is fully compatible with [OpenSearch Suggestions], so every
search base URL of an ELMA microservice can also be used as OpenSarch
Suggestions service.
</div>

The order of entities is expected to result from relevance ranking. The query
parameter "search" SHOULD NOT be expected to follow a specific search syntax or
query language but a plain string.

The response of an entity search MUST include a "Content-Language" header
giving the language of labels and descriptions in the response. The language
SHOULD be influenced by Accept-Language response field and an optional request 
parameter "language".


The response to a HTTP GET request at entity search base URL *without* query
parameter "search" is not specified by this specification. A service can treat
such request equal a request with parameter "search" set to the empty string
but it can also response with another kind of JSON document. If entity search
base URL and entity lookup base URL are equal, a missing parameter "search"
SHOULD be handled equal to a missing parameter "uri". 

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

* Make entity search optional
* Make error format recommended instead of required
* Extend examples, improve language and structure
* Remove User-Agent header 

### 0.0.3 (2016-05-04) {.unnumbered}

* Changed query paramater "q" to "search"

### 0.0.2 (2016-05-02) {.unnumbered}

* Allow language ranges in response format

