# WebID-OIDC認証の仕様
[![](https://img.shields.io/badge/project-Solid-7C4DFF.svg?style=flat-square)](https://github.com/solid/solid)

**Current Spec version:** `v.0.1.0` (see [CHANGELOG.md](CHANGELOG.md))

WebID-IODCは[Solid](https://github.com/solid/solid)などのWebIDベースの非中央集権型システム、およびほとんどのLDPベースのシステムに適した認証委任プロトコル（認証関連の検証技術の有用なツールキット）です。これは非中央集権型のOAuth2/OpenID Connectに基づいています。

## 目次
* [Introduction](#introduction)
    - [Benefits and Capabilities](#benefits-and-capabilities)
    - [If You're Unfamiliar with OIDC](#if-youre-unfamiliar-with-oidc)
* [Differences from Classic OpenID Connect](#differences-from-classic-openid-connect)
* [Brief Workflow Summary](#brief-workflow-summary)
* [Deriving WebID URI from ID Token](#deriving-webid-uri-from-id-token)
* [WebID Provider Confirmation](#webid-provider-confirmation)
* [Authorized OIDC Issuer Discovery](#authorized-oidc-issuer-discovery)
    - [Issuer Discovery From Link Header](#issuer-discovery-from-link-header)
    - [Issuer Discovery From WebID Profile](#issuer-discovery-from-webid-profile)
* [Detailed Sign In Workflow Example](#detailed-sign-in-workflow-example)
* [Detailed Application Workflow Example](#detailed-application-workflow-example)
* [Decentralized Authentication Glossary](#decentralized-authentication-glossary)

## はじめに
WebIDベースの認証ワークフローで最終的に得られるものは、検証されたWebID URI（具体的には、受信者がエージェントがそのURIをコントロールしていることを検証したもの）です。例えば、[WebID-TLS](https://github.com/solid/solid-spec/blob/master/authn-webid-tls.md)は、TLS証明書からWebID URIを導き出し、その証明書をエージェントのWebIDプロファイル内の公開鍵と照合して検証します。同様に、[OpenID Connect (OIDC)](https://openid.net/specs/openid-connectcore-1_0.html) ワークフローで最終的に得られるものは、検証済みのIDトークンです。 WebID-OIDCプロトコルは、OIDC IDトークンからWebID URIを取得するためのメカニズムを指定し、WebIDの非中央集権化された柔軟性とOpenID Connectで実証されたセキュリティの両方の利点を享受します。

参考: [Motivation for WebID-OIDC](motivation.md).

### メリットと機能
* 完全に非中央集権化されたクロスドメイン認証（任意のピアノードは、他のノードへのRelying Partyとしてだけでなく、認証情報の提供者としても機能します）
* 数十年にわたる実際の認証業界の経験に基づいています
* SAML、OpenIDとOpenID 2、OAuthとOAuth 2の教訓を組み入れ、それらの脅威モデルを修正します。例えば、[RFC 6819 - OAuth 2.0 Threat Model and Security Considerations](http://tools.ietf.org/html/rfc6819)を参照してください。-- OpenID Connectの大部分は、そこに概説されている脅威に対処するために開発されました
* 巨人（※GAFAなどの巨大なプラットフォーマー）の肩の上に立ちます（トークン表現、暗号化署名、および暗号化にJOSE標準スイート（[JWT](https://tools.ietf.org/html/rfc7519), [JWA](https://tools.ietf.org/html/rfc7518), [JWE](https://tools.ietf.org/html/rfc7516), [JWS](https://tools.ietf.org/html/rfc7515)）を使用します）
* サインオフ（およびシングルサインオフ）機能
* [失効](https://tools.ietf.org/html/rfc7009)機能、プロバイダとクライアントアプリの両方のブラックリスト機能とホワイトリスト機能を持つ
* ブラウザ内のJavascriptアプリ、従来のサーバーサイドWebアプリ、モバイルおよびデスクトップアプリ、IoTデバイスなど、あらゆる種類のエージェントやクライアントの認証をサポートします
* Solidサーバーのような既存の[Web Access Control](https://github.com/solid/web-access-control-spec)ACL実装との互換性。
* Solidに機能的な将来性を追加するためのインフラストラクチャを設定する。

### あなたがOIDCに慣れていない場合
OIDC/OAuth2のワークフローに慣れていない場合は、次の手順に従ってください。:

 * 以下の[Brief Workflow Summary](#brief-workflow-summary)のセクションを読んでください
 * 様々な用語（Relying Party、Providerなど）の意味は[Decentralized Authentication Glossary](#decentralized-authentication-glossary)を参考にしてください
 * [OpenID Connect explained](http://connect2id.com/learn/openid-connect)を読んでください。基本的なOIDCの概念に慣れることは、この仕様を理解する上で非常に役立ちます。

## Differences from Classic OpenID Connect

WebID-OIDC makes the following changes to the base OpenID Connect protocol
(which itself improves and builds on OAuth 2):

* Discusses and formalizes the [Provider
  Selection](example-workflow.md#21-provider-selection) step.
* Adds a procedure for [Deriving a WebID URI from ID
  Token](#deriving-webid-uri-from-id-token), since WebID-based protocols use
  the WebID URI as a globally unique identifier (rather than the combination of
  `iss`uer and `sub`ject claims).
* Adds an additional step: [WebID Provider
  Confirmation](#webid-provider-confirmation).
  After the WebID URI is extracted, the recipient of the ID Token must confirm
  that the Provider was indeed authorized by the holder of the WebID profile.
* Specifies the [Authorized OIDC Issuer
  Discovery](#authorized-oidc-issuer-discovery) process (used as part of
  Provider Confirmation, and during Provider Selection steps).

It's also worth mentioning that while traditional OpenID Connect use cases are
concerned with retrieving user-related claims from [UserInfo
endpoints](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo), most
WebID based systems replace the UserInfo mechanism with the contents of
[WebID Profile](https://github.com/solid/solid-spec#webid-profile-documents)
documents.

## Brief Workflow Summary

The overall sign in workflow used by the WebID-OIDC protocol is as follows.
For example, here is what happens when Alice tries to request the resource
`https://bob.example/resource1`.

1. [Initial Request](example-workflow.md#1-initial-request): Alice
   (unauthenticated) makes a request to `bob.example`, receives an HTTP `401
   Unauthorized` response, and is presented with a 'Sign In With...' screen.
2. [Provider Selection](example-workflow.md#21-provider-selection): She selects
   her WebID service provider by clicking on a logo, typing in a URI (for
   example, `alice.solidtest.space`), or entering her email.
3. [Local
   Authentication](example-workflow.md#3-local-authentication-to-provider):
   Alice gets redirected towards her service provider's own Sign In page, thus requesting
   `https://alice.solidtest.space/signin`, and authenticates using her preferred
   method (password, WebID-TLS certificate, FIDO 2 /
   [WebAuthn](https://w3c.github.io/webauthn/) device, etc).
4. [User Consent](example-workflow.md#4-user-consent): (Optional) She'd also be
   presented with a user consent screen, along the lines of "Do you wish to
   sign in to `bob.example`?".
5. [Authentication Response](example-workflow.md#5-authentication-response):
   She then gets redirected back towards `https://bob.example/resource1` (the
   resource she was originally trying to request). The server, `bob.example`, also
   receives a signed ID Token from `alice.solidtest.space` that was returned
   with the response in point 3, attesting that she has signed in.
6. [Deriving a WebID URI](#deriving-webid-uri-from-id-token):
   `bob.example` (the server controlling the resource) validates the ID Token, and
   extracts Alice's WebID URI from inside  it. She is now signed in to
   `bob.example` as user `https://alice.solidtest.space/#i`.
7. [WebID Provider Confirmation](#webid-provider-confirmation):
   `bob.example` confirms that `solidtest.space` is indeed Alice's authorized OIDC
   provider (by matching the provider URI from the `iss` claim with Alice's
   WebID).

There is a lot of heavy lifting happening under the hood, performed by `bob.example`
and `alice.solidtest.space`, the two servers involved in this exchange. They
establish a trust relationship with each other (via
[Discovery](example-workflow.md#22-provider-discovery), and [Dynamic
Registration](example-workflow.md#23-dynamic-client-registration-first-time-only)),
they verify each other's signatures against their public keys, and verify
Alice's client app (if she's using one). Fortunately, all of that complexity is
hidden from the user (and most of it is also hidden from the app developer).

## Deriving WebID URI from ID Token

A WebID-OIDC conforming Relying Party tries the following methods, in order, to
obtain a WebID URI from an ID Token:

##### Method 1 - Custom `webid` claim

First, check the ID Token payload for the `webid` claim. This claim is added to
the set of [OpenID Connect ID
Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) claims by
this WebID-OIDC spec. (Note that the set of ID Token claims is extensible, by
design, as explained in the OIDC Core spec.) If the `webid` claim is present in
the ID Token, its value should be used as the WebID URI by the Relying Party,
instead of the traditional `sub` (subject identifier) claim.

This method is useful when the identity providers issuing the tokens can
customize the contents of their ID tokens (and can add the `webid` claim).

##### Method 2 - Value of `sub` claim contains a valid HTTP(S) URI

If the `webid` claim is not present in the ID Token, the Relying Party should
check if the `sub` claim contains as its value a valid HTTP(S) URI. If the value
of the `sub`ject claim is a valid URI, it should be used as the WebID URI by the
Relying Party.

This method is useful when the identity providers issuing the tokens cannot add
claims but can set their own values of their `sub`ject claims (that is, they're
not automatically generated as UUIDs, etc).

##### Method 3 - UserInfo request + `website` claim

If a WebID URI is not found in either the `webid` or `sub` claim, the Relying
Party should proceed to make an OpenID Connect [UserInfo
Request](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo), with
the appropriate Access Token that it received alongside the ID Token. This
method is provided for cases where users do not have control over the contents
of the ID Tokens issued by their Providers and so would not be able to use the
previous methods. This would be the case, for example, if a user wanted to sign
in to a WebID-OIDC Relying Party using an existing mainstream Provider such as
Google. Once the UserInfo response is received by the Relying Party, the
standard `website` claim should be used as the WebID URI by that RP.

## WebID Provider Confirmation

#### The Problem

The OIDC spec uses the ID Token's `sub`ject claim as a unique user id. However,
it requires that the id is unique *for a given Provider*. Given that a WebID
is a *globally* unique user identifier, the WebID-OIDC protocol needs to take
an additional step and *confirm* that the holder of that WebID has authorized
a given Provider to use that WebID. Otherwise, the following situation can
happen:

1. Alice logs in to `bob.example` with her identity Provider of choice, `alice.example`.
   The ID Token from `alice.example` claims Alice's WebID is
   `https://alice.example/#i`. So far so good.
2. An attacker also logs in to `bob.example`, using `evilbox.com` as an identity
   Provider. And because they happen to control that server, they can put
   anything they want in the `webid` claim of any ID Token coming out of that
   server. So they *also* claim that their `webid` is `https://alice.example/#i`.

Without an additional confirmation step, how can a recipient of an ID Token
(here, `bob.example`) know which of those login attempts is correct? To put it
another way, how can a recipient know which Provider is *approved* by the
owner of the WebID?

#### The Solution

When presented with WebID-OIDC credentials in the form of bearer tokens,
the Resource Server MUST confirm that the Identity Provider (the value in the
`iss`uer claim) is authorized by the holder of the WebID, by doing the
following:

1. (Common case) If the server that issued the ID Token is the same entity that
  hosts the WebID profile, then it is considered the authorized OIDC provider
  for that WebID (short-circuiting the Provider Confirmation process), and no
  further steps need to be taken. Specifically, one of the following must be
  true:
    - The [origin](https://developer.mozilla.org/en-US/docs/Web/API/URL/origin)
      of the WebID URI is the same as the origin of the URI in the
      `iss`uer claim. (For example, `iss: 'https://example.com'` and the
      WebID URI is `https://example.com/profile#me`).
    - The WebID URI is a *subdomain* of the issuer of the ID Token.
      For example, `iss: 'https://example.com'` and the WebID URI is
      `https://alice.example.com/profile#me`.
   If neither of the above is the case (and the WebID is hosted on a security
   realm different than that of its OIDC provider), further steps need to be
   taken for confirmation.
2. Determine the **authorized OIDC provider** URI for that WebID, by performing
   [Authorized OIDC Issuer Discovery](#authorized-oidc-issuer-discovery).
3. If the Provider URI is not discoverable (either from the header or the body
   of the [WebID
   Profile](https://github.com/solid/solid-spec#webid-profile-documents), the Resource Server MUST reject the credentials or authentication attempt.
4. If the Provider URI is discovered, it MUST match the Issuer URI in the ID
   Token (the `iss` claim), reject the credentials otherwise.

## Authorized OIDC Issuer Discovery

During the Provider Selection or [Provider
Confirmation](#webid-provider-confirmation) steps it is necessary to discover,
for a given WebID, the URI of the authorized OIDC provider for that WebID.

1. First, attempt to [discover from link
   headers](#issuer-discovery-from-link-header)
2. If not found, proceed to [discover from the WebID
   Profile](#issuer-discovery-from-webid-profile)

Note that this procedure is different from the classic [OpenID Provider Issuer
Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html#IssuerDiscovery)
process, since that spec is concerned with discovering the issuer URI *from a
user's email address*, using the WebFinger protocol. Whereas this spec needs to
derive the issuer URI from a WebID URI (which is often hosted on a different
domain than the issuer).

#### Issuer Discovery From Link Header

**Note: this feature is at risk of being removed.
Please [join the discussion](https://github.com/solid/webid-oidc-spec/issues/18).
Code depending on this will still work for now.**

To discover the authorized OIDC Issuer for a given WebID from Link rel headers:

1. Make an HTTP OPTION request to the WebID URI.
2. Parse the `Link:` header, and check for the value of the
  `http://openid.net/specs/connect/1.0/issuer` link relation. If
  present, use the value of that link relation as the authorized Provider
  URI. For example: `Link: <https://provider.example.com>; rel="http://openid.net/specs/connect/1.0/issuer"` means that `https://provider.example.com` is the authorized OIDC provider for that URI.
3. If the Link header is not present (or does not contain the relevant link
   relation), proceed to discovering the issuer from the WebID profile contents.

#### Issuer Discovery From WebID Profile

To discover the authorized OIDC Issuer for a given WebID from the WebID Profile
contents (this requires Turtle/RDF parsing capability):

1. Dereference the WebID URI (make an HTTP GET request) and fetch the contents
  of the WebID Profile (typically in Turtle or JSON-LD format or some other RDF
  serialization).
2. Parse the RDF, and query for the object of the statement containing the
  `<http://www.w3.org/ns/solid/terms#oidcIssuer>` predicate.

For example, if Alice (with the WebID of `https://alice.example.com/profile#me`)
wanted to specify `https://provider.com` as the authorized OIDC provider for
that profile, she would add the following triple to her profile:

```ttl
@prefix solid: <http://www.w3.org/ns/solid/terms#>.

# ...

<#me> solid:oidcIssuer <https://provider.com> .
```

## Detailed Sign In Workflow Example

To walk through a more detailed example for WebID-OIDC login, refer to the
[Example WebID-OIDC Workflow](example-workflow.md) doc.

## Detailed Application Workflow Example

For a detailed example of how an application/agent can access resources in a
pod on behalf of a given user, refer to the
[Example Application OIDC Workflow](application-workflow.md).

## Decentralized Authentication Glossary

In order to discuss decentralized authentication protocol details, it would be
helpful to familiarize ourselves with the terminology that is frequently used
by various decentralized protocol specs (such as OAuth2, OpenID Connect).

##### User
Human user. If the user is an app or service (that has its own WebID Profile),
this can be generalized to `Agent`. Also called `Resource Owner`. In the
following examples, Alice and Bob are Users.

##### User-Agent
A formal name for a `Browser`. Note that this is often separate from a Client
application (in many cases, client apps written in Javascript run *inside* the
browser).

##### Identity Provider (OP)
An OpenID Connect Identity `Provider` (called `OP` in most OIDC specs). Also
sometimes referred to as `Issuer`. This can be either a POD (see below) or an
external OIDC provider such as
[Google](https://developers.google.com/identity/protocols/OpenIDConnect). In
the spec, Alice's POD, `alice.example`, will mostly play the role of a Provider.

##### Resource Server (RS)
A server hosting resources that the user wants to access, such as HTML, images,
Linked Data / RDF sources, and so on. In the spec, `bob.example` will be used as
the `Resource Server` (that is, Alice will be requesting resources on Bob's
server). *Note:* In the traditional federated social sign on context, a
provider (such as Facebook) serves as *both* an Identity Provider *and* a
Resource Server.

##### Relying Party (RP)
A `Relying Party` is a POD or a client app that has to *rely* on an ID Token
that's issued by a Provider. In the spec, when Alice tries to access a resource
on `bob.example`, Bob's POD acts as the Relying Party, in that interaction.
And correspondingly, Alice's POD, `alice.example`, will serve as the Identity
Provider, again for that interaction.

Incidentally, when Alice tries to access a resource on her *own* POD,
`alice.example` plays all of the roles -- it's both the Provider and a Relying
Party (as well as the Resource Server).

##### POD
A Personal Online Datastore (POD for short). It plays several roles -- firstly,
it stores a user's data (and so acts as a `Resource Server`). In many cases, it
also hosts the user's WebID Profile, and implements the API endpoints that allow
it to act as a WebID-OIDC Identity Provider (OP). Lastly, when users requests
resources from it, the POD also acts as a Relying Party (a recipient of those
users' ID Tokens).
In this spec, `alice.example` and `bob.example` are both PODs.

##### Home POD vs Other POD
A user's Home POD is one that hosts their WebID Profile, and also acts as that
user's Identity Provider. We use the term *Other POD* in this spec to denote
some other WebID-OIDC compliant POD, acting as a Resource Server and Relying
Party, that a user is trying to access using the WebID URI and Profile of their
Home POD.

When Alice tries to access a resource on Bob's POD, `alice.example` is her Home
POD, and `bob.example` plays the role of the Other POD.

##### Public Client vs Confidential Client
Public - in-browser, mobile or desktop app, cannot be trusted with securely
storing secrets (private key material, secret client IDs, etc).
Confidential - server-side app, can be trusted with secrets.

##### Presenter
A public client app that is trying to *present* a user's credentials from their
home POD to some other POD. For example, Bob is trying to access, via a client
app, a shared file on Alice's `alice.example` POD, logging in using his own
`bob.example` POD/provider. In this example, `bob.example` is a Provider, `alice.example` is
a Relying Party, and the client app (say, a browser-based HTML editor) is a
Presenter.
