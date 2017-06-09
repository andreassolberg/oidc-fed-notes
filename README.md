# OpenID Connect Federation

This is a draft note describing an alternative approach to OpenID Connect Federation. Here are some highlights:

* Enforces use of asymmetric crypto for clients
* Allows use of out of regular OpenID Connect without specific support for «Federation»
* No use of the OpenID Connect Dynamic Registration
* Reduces the amount of state necessary to be kept both in clients and providers.



## 1. Client configures a key pair to use with OpenID Connect

The client generates a key pair. The public key may look like this:

```
{
  "kty": "RSA",
  "kid": "https://serviceprovider.org/application#1",
  "n": "l7rt1yRvbiOKg8XeP_ICo0yDif-kOLWkUL5FAWKVWhWWAdnN2o1t_otuBX1xLeItE24he4qGHBzh2PQ4SRqau6ZVzx4-aJFzGZSbw6SswVXPlFR5dRkJMn4wxFOOVsSUnltO4K27X2Pf-gwlLFdH4q4QTNU5U8ijr76BnuUThdBYrxf2UQT7DDz6cPHaRdOUbuj_Ids9CmV6HyzdIFOfBx7DKS8o2fqH9Fa6-PKdMtDJiZ1KfjgstiNB04JAbQ1RI9Bl-No6NTUcZbD7Q0JF8iqY3Hogo9J_mL-SgQFGgwAoxQKoNeLk7uLHc69yIlyBJegrVkmHUKehIp3OZ5CW9w",
  "e": "AQAB"
}
```

## 2. Client chooses a `client_id`

The client will choose a `client_id` that includes the hostname of the client application. The hostname used in the client_id will be used for discovery by the OpenID Connect Provider.

In example the client can use this `client_id`:

* `https://serviceprovider.org/application`



## 3. User and client discovers the OpenID Connect Provider

Using some kind of discovery process the client selects an OpenID Connect Provider aided by the end-user.

Discovery process is specified [in OpenID Connect Core](http://openid.net/specs/openid-connect-core-1_0.html).

Including Provider Metadata statement in the discovery response is described [in OpenID Connect Federation](http://openid.net/specs/openid-connect-federation-1_0.html).


## 4. Client sends an authentication request


```
GET /authorize?
    response_type=code
    &scope=openid%20profile%20email
    &client_id=https%3A%2F%2Fserviceprovider.org%2Fapplication
    &state=7582e571-3fe7-415f-9add-cea770d76324
    &redirect_uri=https%3A%2F%serviceprovider.org/application%2Fcallback HTTP/1.1
  Host: provider.org
```

If the client would like to send a [signed authentication request as described in *OpenID Connect Core section 6*](http://openid.net/specs/openid-connect-core-1_0.html#JWTRequests), it MUST sign the request using the same key pair as defined in section 1.

A deployment profile in a federation MAY require the request to be signed.

## 5. Provider receives the authentication request

If the provider has no preconfigured trust with the incoming request `client_id`, and if the `client_id` is a valid URL, and if the provider support OpenID Connect Federation it will proceed with discovery of the client.

## 6. Provider performs discovery of the client


OpenID Connect Federation uses the following discoverable `rel` value in [WebFinger [RFC7033]](https://tools.ietf.org/html/rfc7033):

* `http://openid.net/specs/connect/1.0/provider`

The provider performs normalization rules to the `client_id` to determine the hostname. The client_id URL will be the `resource`.

In example the `client_id` `https://serviceprovider.org/application` may result in the following WebFinger request:

```
GET /.well-known/webfinger?
  resource=https%3A%2F%2Fserviceprovider.org%2Fapplication&
  rel=http%3A%2F%2Fopenid.net%2Fspecs%2Fconnect%2F1.0%2Fprovider
  HTTP/1.1
Host: example.com
```

The response may include a link for every federation root that has a metadata statement about the client. Each link SHOULD include a property that identifies the trust root of this federation. This aids the provider to fetch and process only relevant metadata. The property `http://openid.net/specs/connect/1.0/federation` is reserved for this purpose.


```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/jrd+json

{
  "subject" : "https://serviceprovider.org/application",
  "links" :
  [
    {
      "rel" : "http://openid.net/specs/connect/1.0/provider",
      "href" : "https://metadata.feide.no/237645",
      "properties": {
        "http://openid.net/specs/connect/1.0/federation": "https://feide.no"
      }
    },
    {
      "rel" : "http://openid.net/specs/connect/1.0/provider",
      "href" : "https://mds.edugain/openid/238476238764",
      "properties": {
        "http://openid.net/specs/connect/1.0/federation": "https://edugain.org"
      }
    },
    {
      "rel" : "http://openid.net/specs/connect/1.0/provider",
      "href" : "https://openid.clarin.org/entries/123",
      "properties": {
        "http://openid.net/specs/connect/1.0/federation": "https://clarin.org"
      }
    }
  ]
}
```

The metadata document MAY be hosted by the client it self or by the federation provider registry or anywhere else.


## 7. Provider fetches and process metadata

In order to establish trust with the client, the provider may fetch all the relevant metadata documents using the links from the discovery described above.

Fetching a metadata is as simple as a GET on the link.

The response will be a signed JSON Web Token with MIME type `application/jwt`. No plain text metadata is allowed.

Nested signed metadata statements are described [in OpenID Connect Federation](http://openid.net/specs/openid-connect-federation-1_0.html).
