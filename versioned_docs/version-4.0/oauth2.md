---
title: OAuth 2.0 Authentication Backend
---
<!--
Copyright (c) 2007-2025 Broadcom. All Rights Reserved. The term "Broadcom" refers to Broadcom Inc. and/or its subsidiaries.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# OAuth 2.0 Authentication Backend

## Overview {#overview}

This [RabbitMQ authentication/authorisation backend](./access-control) plugin lets applications
(clients) and users authenticate and authorize using JWT-encoded OAuth 2.0 access tokens.

There's also a companion [troubleshooting guide for OAuth 2-specific problems](./troubleshooting-oauth2).

## Table of Contents

### [How to configure it](#how-to-configure-it)

* [Prerequisites](#prerequisites)
* [Configure OAuth 2.0 step by step](#configure-for-openid-connect)

### [How it works](#how-it-works)

* [Authorization Flow](#authorization-flow)
* [Variables configurable in rabbitmq.conf](#variables-configurable)
* [Token validation](#token-validation)
* [Token expiration and refresh](#token-expiration)
* [Scope-to-Permission translation](#scope-translation)
* [Topic Exchange scopes](#topic-exchange-scopes)
* [Scope and tags](#scope-and-tags)

### [Basic usage](#basic-usage)

* [Configure OAuth 2.0 provider's issuer](#configure-issuer)
* [Configure signing keys](#configure-signing-keys)
* [Use a different token field for the scope](#use-different-token-field)
* [Preferred username claims](#preferred-username-claims)
* [Rich Authorization Request](#rich-authorization-request)

### [Advanced usage](#advanced-usage)

* [Use default OAuth 2.0 provider](#use-oauth-provider)
* [Configure OAuth 2.0 provider's end_session_endpoint](#configure-end-session-endpoint)
* [Configure multiple resource servers](#multiple-resource-servers)
* [Configure multiple OAuth 2.0 providers](#multiple-oauth-providers)

### Examples for Specific Identity Providers

* How to [set up RabbitMQ with OAuth 2: examples](#examples)

## How to configure it {#how-to-configure-it}

This is a step-by-step guide for how to configure OAuth 2.0 authentication for the majority of
use-cases, such as when:

- You are using an identity provider that is compliant with OpenId Connect
- You want RabbitMQ to trigger OAuth 2.0 authentication in the management UI. This is known as
  *Service Provider initiated logon*. As opposed to, users must come to RabbitMQ already with a
  token obtained by other means.

### Prerequisites {#prerequisites}

#### Step 1: Enable the plugin

The OAuth 2 plugin must be activated (or [pre-configured](./plugins#enabled-plugins-file)) before it
can be used, like all other plugins. There is a file commonly found at
`/etc/rabbitmq/enabled_plugins`. Add the OAuth 2 plugin as follows:

```bash
[rabbitmq_management,rabbitmq_shovel,rabbitmq_auth_backend_oauth2].
```

#### Step 2: Configure OAuth 2.0 as one of the authentication backends

Edit your `rabbitmq.conf` file, commonly found at `/etc/rabbitmq` and add OAuth 2.0 backend. For
example, the following configuration adds OAuth 2.0 as the only authentication backend:

```ini
auth_backends.1 = oauth2
```

### Configure OAuth 2.0 step by step{#configure-for-openid-connect}

Nowadays, the majority of the identity providers are compliant with OpenId Connect. At minimum, this
means that they expose an HTTP endpoint called *Discovery endpoint* mounted at
`/.well-known/openid-configuration`. Through this endpoint, RabbitMQ discovers all the necessary
endpoints for a successful OAuth 2.0 authentication. The following identity providers are compliant
with OpenId Connect:

- UAA
- Keycloak
- Auth0
- Entra ID (a.k.a Azure)
- Okta

#### Step 1: Configure the OAuth 2.0 resource name associated to RabbitMQ {#step1}

```ìni
auth_oauth2.resource_server_id = rabbitmq
```

:::warning
OAuth 2.0 tokens have an attribute called `audience`. This attribute lists all the OAuth 2.0
resources the token is meant for. RabbitMQ expects to find its `resource_server_id` in one of the
`audience` values. If it is not found, the token is rejected and the authentication fails.

Although it is not recommended, you can disable audience validation by adding the following
configuration.

```ini
auth_oauth2.verify_aud = false
```

:::

#### Step 2: Configure root URL of the Identity Provider {#step2}

```ini
auth_oauth2.issuer = https://keycloak:8443/realms/test
```

:::info
It must be an HTTPS URL. If your identity provider uses a certificate signed by a self-signed CA
certificate, you can configure the CA certificate by adding the following line to your configuration:

```ini
auth_oauth2.https.cacertfile = <path_to_your_ca_cert>
```

:::

#### Step 3: Configure which token's attributes contain the user name {#step3}

RabbitMQ needs to know the user name for logging purposes and also to display it in the management
UI. By default, RabbitMQ looks up the user name in the token's attribute `sub`, and if not found it
uses `client_id`.

Here you are configuring RabbitMQ to use `user_name` first and, if not found, `preferred_username`
instead.

```ini
auth_oauth2.preferred_username_claims.1 = user_name
auth_oauth2.preferred_username_claims.2 = preferred_username
```

#### Step 4: Configure which token's attribute contains the scopes (a.k.a. permissions) {#step4}

By default, RabbitMQ always looks for scopes in the token's attribute `scope`. If your scopes are in
a different attribute you can configure the name of that attribute.

Given the following configuration, RabbitMQ looks for scopes in the attributes `scope` and
`extra_scope`. The attribute's value can be a string of space-separated scopes or a JSON list of
scopes.

```ini
auth_oauth2.additional_scopes_key = extra_scope
```

:::info
If you need to specify more than one attribute or the scopes are under a nested structure like shown
below, you need to upgrade to [RabbitMQ 4.1](/docs/oauth2#step4).

```json
{ ...
 "resource_access": {
    "account": {
      "roles": [
        "manage-account",
        "manage-account-links",
        "view-profile"
      ]
    }
  }
  ...
}
```

:::

#### Step 5: Ensure your scopes are recognized RabbitMQ scopes {#step5}

Ensure that the scopes contained in the token are recognized RabbitMQ scopes. Here are examples of
[recognized RabbitMQ scopes](#scope-translation):

* `rabbitmq.tag:administrator` Here `rabbitmq` is the name of the `resource_server_id` you configured
  in the first step. `tag:administrator` grants the administrator role to the management UI.
* `rabbitmq.configure:*/*` This scope grants the `configure` permission. The first wildcard
  corresponds to the vhost and the second wildcard to the queue/exchange.

They all start with a prefix that, by default, must match the name of the resource you configured
in [step 1)](#step1).

If your scopes are recognized RabbitMQ scopes, however, they start with a different prefix. You can
configure your custom prefix as follows:

```ini
auth_oauth2.scope_prefix = api://
```

In the above example, you are saying that your scopes are like `api://tag:administrator`.

:::info
More information about scope prefix, check out [Scope Prefix](#scope-prefix).
:::

#### Step 6: Map custom permissions/scopes onto RabbitMQ scopes {#step6}

This step is mandatory if the scopes in the token are not recognized RabbitMQ scopes. In the
previous step you saw what recognized RabbitMQ scopes look like.

Here you are configuring two custom scopes, `admin` and `developer`, on to RabbitMQ scopes:

```ini
auth_oauth2.scope_aliases.admin = rabbitmq.tag:administrator rabbitmq.read:*/
auth_oauth2.scope_aliases.developer = rabbitmq.tag:management rabbitmq.read:*/* rabbitmq.write:*/* rabbitmq.configure:*/*
```

:::info
For more information about mapping scopes see [Scope Aliases](oauth2-examples#using-scope-aliases).
:::

#### Step 7: OAuth 2.0 authentication completed for messaging protocols {#step7}

RabbitMQ is ready to authenticate applications connecting through one of the supported messaging
protocols. If you additionally want to enable OAuth 2.0 in the management UI carry on with following
steps.

#### Step 8: Enable OAuth 2.0 on the management UI {#step8}

The following steps configure RabbitMQ to use OAuth 2.0 to authenticate users with their identity
provider and get a token. This is known as
[service-provider initiated logon](oauth2-examples#service-provider-initiated-logon).

```ini
management.oauth_enabled = true
```

:::info
After you enable OAuth 2.0 in the management UI, basic authentication is disabled unless you add the
following configuration.

```ini
management.oauth_disable_basic_auth = false
```

For more information, check out [Allow Basic and OAuth 2 authentication for management UI](management#allow-basic-auth-for-mgt-ui).
:::

#### Step 9: Configure OAuth client_id {#step9}

```ini
management.oauth_client_id = <Your_Client_Id_from_your_identity_provider>
```

:::warning
The identity provider `UAA` requires a `client_secret` in addition to a `client_id`.
This is not necessary in the rest of the identity providers. The OAuth Client configured for
RabbitMQ in your identity provider should be of type `public web application` and not `confidential`.
The latter requires `client_secret`.
:::

#### Step 10: Configure which scopes the management UI requests from your identity provider {#step10}

When RabbitMQ requests a token from the identity provider, it has to specify which scopes should
contain the token. Through the configuration variable `management.oauth_scopes` you provide a
space-separated list of scopes.

You always include the first two scopes `openid` and `profile` followed by any RabbitMQ scope.

```ini
management.oauth_scopes = openid profile rabbitmq.tag:management
```

:::info
Some identity providers can include more scopes than the requested scopes. It depends on the actual
identity provider implementation.
:::

## How it works {#how-it-works}

## Prerequisites {#prerequisites}

To use the OAuth 2 plugin, all RabbitMQ nodes must be

1. [Configured to use the rabbit_auth_backend_oauth2 backend](./access-control).
2. [Configured with the resource service ID](#resource-server-id) (`resource_server_id`). The
   RabbitMQ cluster becomes an OAuth 2.0 resource and this is its identifier.
3. [Configured with issuer URL](#configure-issuer) of the OAuth 2.0 provider, or the JWKS URL, or
   directly with the signing keys that the OAuth 2.0 provider uses to sign tokens.

Here is the minimal configuration to support OAuth 2.0 authentication:

:::info
To activate it in the Management plugin you need [additional configuration](./management#minimum-configuration).
:::

```ini
auth_oauth2.resource_server_id = new_resource_server_id
auth_oauth2.issuer = https://my-oauth2-provider.com/realm/rabbitmq
```

Based on the previous configuration, JWT Tokens presented to RabbitMQ for authentication must:

- Be digitally signed.
- Have a value in the `aud` field that matches the `resource_server_id` value.
- Have scopes that must match the `resource_server_id` value. For example,
  `new_resource_server_id.read:*/*`.

Also, the `https://my-oauth2-provider.com/realm/rabbitmq/.well-known/openid-configuration` endpoint
must return the OpenID Provider Configuration which includes the JKWS URL to download the signing
keys.

:::info
*.well-known/openid-configuration* is the OpenID standard path for the OpenID Provider Configuration
endpoint
:::

More detail is included in the next section about what happens during the authentication and how to
configure OAuth 2.0 beyond the basic configuration shown previously.

## Authorization Flow {#authorization-flow}

This plugin does not communicate with any OAuth 2.0 provider in order to authenticate the user and
grant access. Instead, it decodes an access token provided by the client and authorizes a user based
on the scopes found in the token.

Tokens must be digitally signed otherwise they are not accepted. RabbitMQ must have the signing key
to validate the signature. You can either configure the signing keys the OAuth 2.0 provider will
use, or configure RabbitMQ with one of the following two endpoints:

* **JWKS endpoint**: This is the HTTP endpoint that returns the signing keys used to digitally sign
  the tokens.
* **OpenID Provider Configuration endpoint**: This endpoint returns the provider's configuration
  including all of its endpoints, most importantly the **JWKS endpoint**.

When you configure RabbitMQ with one of two previous endpoints, RabbitMQ must make a HTTP request
(or two, if we specify the latter endpoint) to download the signing keys. This is an operation that
occurs once for any signing key not downloaded yet.

When the OAuth 2.0 provider rotates the signing keys, newer tokens refer to a new signing key which
RabbitMQ does not have yet which triggers another download of the newer signing keys.

The token can be any [JWT token](https://jwt.io/introduction/) which contains the `scope` and `aud`
fields.

In chronological order, here is the sequence of events that occur when a client application wants to
connect to one of the RabbitMQ's messaging protocols, such as AMQP:

1. The Client application requests an **access_token** from the OAuth 2.0 provider.
2. The **access token** must include **scopes** supported by RabbitMQ in the `scope` field (it is
   possible to use a different field for the **scopes** by setting the name of the new field in
   `auth_oauth2.additional_scopes_key`).
3. The Client application passes the token as password when connecting to RabbitMQ's messaging
   protocol. **The username field is ignored**.
4. RabbitMQ validates the token's signature. To validate it, RabbitMQ must have the signing keys or
   download them from the JWKS endpoint as explained in earlier sections.
5. RabbitMQ validates that the token has the **audience** claim and whose value matches the
   `resource_server_id` (this operation can be deactivated by setting `auth_oauth2.verify_aud` to
   `false`).
6. RabbitMQ translates the **scopes** found in the token into RabbitMQ **permissions** (the same
   permissions used in the RabbitMQ's internal database).

## Variables configurable in rabbitmq.conf {#variables-configurable}

| Key                                        | Documentation
|--------------------------------------------|-----------
| `auth_oauth2.resource_server_id`           | The [Resource Server ID](#resource-server-id)
| `auth_oauth2.resource_server_type`         | The Resource Server Type required when using [Rich Authorization Request](#rich-authorization-request) token format
| `auth_oauth2.additional_scopes_key`        | Configure the plugin to look for scopes in other fields (maps to `additional_rabbitmq_scopes` in the old format). |
| `auth_oauth2.scope_prefix`                 | [Configure the prefix for all scopes](#scope-prefix). The default value is `auth_oauth2.resource_server_id` followed by the dot `.` character. |
| `auth_oauth2.preferred_username_claims`    | [List of the JWT claims](#preferred-username-claims) to look for the username associated with the token.
| `auth_oauth2.default_key`                  | ID of the default signing key.
| `auth_oauth2.signing_keys`                 | Paths to the [signing key files](#signing-key-files).
| `auth_oauth2.issuer`                       | The [issuer URL](#configure-issuer) of the authorization server that is used to discover endpoints such as `jwks_uri` and others (https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata).
| `auth_oauth2.jwks_url`                     | The URL of the [JWKS endpoint](#jwks-endpoint). According to the [JWT Specification](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.2), the endpoint URL must be https.
| `auth_oauth2.token_endpoint`               | The URL of the OAuth 2.0 token endpoint.
| `auth_oauth2.https.cacertfile`             | Path to a file containing PEM-encoded CA certificates. The CA certificates are used to connect to any of these endpoints: `jwks_url`, `token_endpoint`, or the `issuer`.
| `auth_oauth2.https.depth`                  | The maximum number of non-self-issued intermediate certificates that may follow the peer certificate in a valid [certification path](ssl#peer-verification-depth). The default value is 10.
| `auth_oauth2.https.peer_verification`      | Configures [peer verification](ssl#peer-verification). Available values: `verify_none`, `verify_peer`. The default value is `verify_peer` if there are trusted CA installed in the OS or `auth_oauth2.https.cacertfile` is set. <p/> **Deprecated**: This variable will be soon replaced by `auth_oauth2.https.verify`. Users should stop using this variable.
| `auth_oauth2.https.fail_if_no_peer_cert`   | Used together with `auth_oauth2.https.peer_verification = verify_peer`. When set to `true`, TLS connection will be rejected if the client fails to provide a certificate. The default value is `false`.
| `auth_oauth2.https.hostname_verification`  | Enable wildcard-aware hostname verification for key server. Available values: `wildcard`, `none`. The default value is `none`.
| `auth_oauth2.https.crl_check`              | [Perform CRL verification](https://www.erlang.org/doc/man/ssl#type-crl_check) (Certificate Revocation List) verification. Default value is false.
| `auth_oauth2.algorithms`                   | Restrict [the usable algorithms](https://github.com/potatosalad/erlang-jose#algorithm-support).
| `auth_oauth2.verify_aud`                   | Whether to verify the [token's `aud`](#token-validation) field or not. The default value is `true`.
| `auth_oauth2.resource_servers`             | [Multiple OAuth 2.0 resources configuration](#multiple-resource-servers-configuration).
| `auth_oauth2.oauth_providers`              | [Multiple OAuth 2.0 providers configuration](#multiple-oauth-providers-configuration).
| `auth_oauth2.default_oauth_provider`       | ID of the OAuth 2.0 provider used for the `auth_oauth2.resource_servers`, that did not specify any (via the variable `oauth_provider_id`) or when `auth_oauth2.jwks_url` and `auth_oauth2.issuer` are both missing.

## Resource Server ID {#resource-server-id}

A RabbitMQ cluster must have at least one resource server identifier configured. If it has just one
resource, this is configured in the `auth_oauth2.resource_server_id` variable and it is
**mandatory**. If the RabbitMQ cluster has more than one OAuth resource then they are configured
under `auth_oauth2.resource_servers.<index>` and in this case `auth_oauth2.resource_server_id`
variable is not mandatory.

RabbitMQ uses the resource server identity for these two purposes:

- To validate the token's audience (`aud`) whose value must contain the resource server identifier.
  This validation can be disabled though.
- To initiate the OAuth 2.0 Authorization Code flow in the Management UI. This is the flow used to
  authenticate a user and to get its access token. RabbitMQ must include the resource server
  identifier in the request's attribute called `resource`.

## Scope Prefix {#scope-prefix}

OAuth 2.0 tokens use scopes to communicate what set of permissions particular client are granted.
The scopes are free form strings.

By default, `resource_server_id` followed by the dot (`.`) character is the prefix used for scopes
to avoid scope collisions (or unintended overlap). However, in some environments, it is not possible
to use `resource_server_id` as the prefix for all scopes. For these environments, there is a new
variable called `scope_prefix` which overrides the default scope prefix.

Given the below configuration, the scope associated with the permission `read:*/*` is
`api://read:*/*`:

```ini
...
auth_oauth2.scope_prefix = api://
...
```

To use an empty string as prefix, use this configuration:

```ini
...
auth_oauth2.scope_prefix = ''
...
```

## Signing keys files {#signing-key-files}

The following configuration declares two signing keys and configures the kid of the default signing
key. For more information check the section [Configure Signing keys](#configure-signing-keys).

```ini
auth_oauth2.resource_server_id = new_resource_server_id
auth_oauth2.additional_scopes_key = my_custom_scope_key
auth_oauth2.preferred_username_claims.1 = username
auth_oauth2.preferred_username_claims.2 = user_name
auth_oauth2.default_key = id1
auth_oauth2.signing_keys.id1 = test/config_schema_SUITE_data/certs/key.pem
auth_oauth2.signing_keys.id2 = test/config_schema_SUITE_data/certs/cert.pem
auth_oauth2.algorithms.1 = HS256
auth_oauth2.algorithms.2 = RS256
```

## JWKS endpoint {#jwks-endpoint}

The following configuration sets the JWKS endpoint from which RabbitMQ downloads the signing keys
using the configured CA certificate and TLS variables.

```ini
auth_oauth2.resource_server_id = new_resource_server_id
auth_oauth2.jwks_url = https://my-jwt-issuer/jwks.json
auth_oauth2.https.cacertfile = test/config_schema_SUITE_data/certs/cacert.pem
auth_oauth2.https.peer_verification = verify_peer
auth_oauth2.https.depth = 5
auth_oauth2.https.fail_if_no_peer_cert = true
auth_oauth2.https.hostname_verification = wildcard
auth_oauth2.algorithms.1 = HS256
auth_oauth2.algorithms.2 = RS256
```

## Multiple Resource Servers configuration {#multiple-resource-servers-configuration}

Each `auth_oauth2.resource_servers.<id/index>.` entry has the following variables shown in the table
below. Except for the variables `id` and `oauth_provider_id`, if a resource does not configure a
variable, RabbitMQ uses the variable configured at the root level.

For example, if the resource `auth_oauth2.resource_servers.prod` does not configure
`preferred_username_claims` variable, RabbitMQ uses the value configured in
`auth_oauth2.preferred_username_claims` for the resource `prod`.

| Key                          | Documentation
|------------------------------|-----------
| `id`                         | The [Resource Server ID](#resource-server-id)
| `resource_server_type`       | The Resource Server Type required when using [Rich Authorization Request](#rich-authorization-request) token format.
| `additional_scopes_key`      | Configure the plugin to look for scopes in other fields (maps to `additional_rabbitmq_scopes` in the old format).
| `scope_prefix`               | [Configure the prefix for all scopes](#scope-prefix). The default value is `auth_oauth2.resource_server_id` followed by the dot `.` character.
| `preferred_username_claims`  | [List of the JWT claims](#preferred-username-claims) to look for the username associated with the token separated by commas.
| `oauth_provider_id`          | The identifier of the OAuth Provider associated to this resource. RabbitMQ uses the signing keys issued by this OAuth Provider to validate tokens whose audience matches this resource's id.

All available configurable parameters for each OAuth 2 provider is documented
[in a separate section](#multiple-oauth-providers-configuration).

Usually, a numeric value is used as `index`. For example,
`auth_oauth2.resource_servers.1.id = rabbit_prod`. However, it can be any string, such as
`auth_oauth2.resource_servers.rabbit_prod.jwks_url = http://some_url`. By default, the `index` is
the resource server's ID. However, you can override it via the `id` variable like in
`auth_oauth2.resource_servers.1.id = rabbit_prod`.

Here is an example which configures two resources (`prod` and `dev`) which are used by the users and
clients managed by the same identity provider whose issuer URL is `https://my-idp.com/`:

```ini
auth_oauth2.issuer = https://my-idp.com/
auth_oauth2.resource_servers.1.id = prod
auth_oauth2.resource_servers.2.id = dev
```

See the advanced usage section called [Multiple Resource Servers](#multiple-resource-servers) for
more information on how to configure them.

#### Multiple OAuth Providers configuration {#multiple-oauth-providers-configuration}

Each `auth_oauth2.oauth_providers.{id/index}` entry has the following sub-keys.

| Key                          | Documentation
|------------------------------|-----------
| `issuer`                     | URL of OAuth Provider. RabbitMQ uses this URL to build the OpenId Connect Discovery endpoint by appending the path `.well-known/openid-configuration` to this URL
| `token_endpoint`             | The URL of the OAuth 2.0 token endpoint.
| `jwks_uri`                   | The URL of the [JWKS endpoint](#jwks-endpoint). According to the [JWT Specification](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.2), the endpoint URL must use "https" for schema. <p/>**Warning**: RabbitMQ uses for each OAuth Provider the variable name `jwks_uri` used by the OpenId Connect Discovery Specification rather than `jwks_url`.
| `https.cacertfile`           | Path to a file containing PEM-encoded CA certificates used to connect `issuer` and/or `jwks_uri` URLs
| `https.depth`                | The maximum number of non-self-issued intermediate certificates that may follow the peer certificate in a valid [certification path](ssl#peer-verification-depth). The default value is 10.
| `https.verify`               | Configures [peer verification](ssl#peer-verification). Available values: `verify_none`, `verify_peer`. The default value is `verify_peer` if there are trusted CA installed in the OS or `auth_oauth2.https.cacertfile` is set.
| `https.fail_if_no_peer_cert` | Used together with `auth_oauth2.https.peer_verification = verify_peer`. When set to `true`, TLS connection will be rejected if the client fails to provide a certificate. The default value is `false`.
| `https.hostname_verification` | Enable wildcard-aware hostname verification for key server. Available values: `wildcard`, `none`. The default value is `none`.
| `https.crl_check`              | [Perform CRL verification](https://www.erlang.org/doc/man/ssl#type-crl_check) (Certificate Revocation List) verification. Default value is false.
| `signing_keys`               | Local filesystem paths to the [signing key files](#signing-key-files). The files must exist and be readable
| `default_key`                | ID of the default signing key
| `algorithms`                 | Used to restrict [the list of enabled algorithms](https://github.com/potatosalad/erlang-jose#algorithm-support)

Here is an example that configures two resources (`prod` and `dev`) where each resource is managed
by two distinct identity providers:

```ini
auth_oauth2.scope_prefix = rabbitmq.
auth_oauth2.resource_servers.1.id = prod
auth_oauth2.resource_servers.1.oauth_provider_id = idp_prod
auth_oauth2.resource_servers.2.id = dev
auth_oauth2.resource_servers.2.oauth_provider_id = idp_dev
auth_oauth2.oauth_providers.idp_prod.issuer = https://idp_prod.com
auth_oauth2.oauth_providers.idp_dev.issuer = https://idp_dev.com
```

See the advanced usage section called [Multiple OAuth Providers](#multiple-oauth-providers) for more
information on how to configure them.

### Token validation {#token-validation}

When RabbitMQ receives a JWT token, it validates it before accepting it.

#### Must be digitally signed

The token must carry a digital signature and optionally a `kid` header attribute which identifies
the key RabbitMQ should use to validate the signature.

#### Must not be expired

RabbitMQ uses this field `exp` ([exp](https://tools.ietf.org/html/rfc7519#page-9)) to validate the
token if present. It contains the expiration time after which the JWT MUST NOT be accepted for
processing.

#### Audience must have/match the resource_server_id

The `aud` ([Audience](https://tools.ietf.org/html/rfc7519#page-9)) identifies the recipients and/or
resource_server of the JWT.

By default, **RabbitMQ uses this field to validate the token**. This validation can be disabled by
setting the `auth_oauth2.verify_aud` setting set to `false`. When verification is enabled, this
`aud` field must either match the `resource_server_id` value or, in case of a list, it must contain
the `resource_server_id` value.

### Token expiration and refresh {#token-expiration}

For AMQP 0.9.1, when a token expires on an existing connection, the broker refuses further
operations after a limited time, but does not disconnect the client. To refresh the token, the
client can use the AMQP 0.9.1 [update-secret](./extensions/) method if supported by the client. For
an example, see the
[Java client documentation](/client-libraries/java-api-guide#oauth2-refreshing-token). If the client
does not support [`update-secret`](./extensions/), it must disconnect and reconnect with a new
token.

### Scope-to-Permission translation {#scope-translation}

Scopes are translated into permission grants to RabbitMQ resources for the provided token.

The current scope format is `<permission>:<vhost_pattern>/<name_pattern>[/<routing_key_pattern>]`
where

* `<permission>` is an access permission (`configure`, `read`, or `write`)
* `<vhost_pattern>` is a wildcard pattern for vhosts token has access to.
* `<name_pattern>` is a wildcard pattern for resource name
* `<routing_key_pattern>` is a wildcard pattern for routing key in topic authorization

Wildcard patterns are strings with optional wildcard symbols `*` that match any sequence of
characters.

Wildcard patterns match as following:

* `*` matches any string
* `foo*` matches any string starting with a `foo`
* `*foo` matches any string ending with a `foo`
* `foo*bar` matches any string starting with a `foo` and ending with a `bar`

There can be multiple wildcards in a pattern:

* `start*middle*end`
* `*before*after*`

**To use special characters like `*`, `%`, or `/` in a wildcard pattern, the pattern must be
[URL-encoded](https://en.wikipedia.org/wiki/Percent-encoding).**

These are the usually permissions examples:

- `read:*/*`(`read:*/*/*`) - read permissions to any resource on any vhost
- `write:*/*`(`write:*/*/*`) - write permissions to any resource on any vhost
- `read:vhost1/*`(`read:vhost1/*/*`) - read permissions to any resource on the `vhost1` vhost
- `read:vhost1/some*` - read permissions to all the resources, starting with `some` on the `vhost1`
  vhost
- `write:vhost1/some*/routing*` - topic write permissions to publish to an exchange starting with
  `some` with a routing key starting with `routing`
- `read:*/*/*` and `write:*/*/*` - queue binding permissions required to bind a queue on a topic
  exchange with any routing key

See the
[wildcard matching test suite](https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_auth_backend_oauth2/test/wildcard_match_SUITE.erl)
and
[scopes test suite](https://github.com/rabbitmq/rabbitmq-server/blob/main/deps/rabbitmq_auth_backend_oauth2/test/scope_SUITE.erl)
for more examples.

Scopes, by default, are prefixed with `resource_server_id` followed by the dot (`.`) character if
`scope_prefix` is not configured. For example, if `resource_server_id` is "my_rabbit", a scope to
enable read from any vhost will be `my_rabbit.read:*/*`.

If `scope_prefix` is configured then scopes are prefixed as follows: `<scope_prefix><permission>`.
For example, if `scope_prefix` is `api://` and the permission is `read:*/*` the scope would be
`api://read:*/*`

### Topic Exchange scopes {#topic-exchange-scopes}

The [previous](#scope-translation) section explained, in detail, how permissions are mapped to
scopes. This section explains more specifically what scopes you need in order to operate on **Topic
Exchanges**.

To bind and/or unbind a queue to/from a **Topic Exchange**, you need to have the following scopes:

- **write** permission on the queue and routing key -> `rabbitmq.write:<vhost>/<queue>/<routingkey>`
> for example `rabbitmq.write:*/*/*`

- **read** permission on the exchange and routing key -> `rabbitmq.write:<vhost>/<exchange>/<routingkey>`
> for example `rabbitmq.read:*/*/*`

To publish to a **Topic Exchange**, you need to have the following scope:

- **write** permission on the exchange and routing key -> `rabbitmq.write:<vhost>/<exchange>/<routingkey>`
> for example `rabbitmq.write:*/*/*`

OAuth 2.0 authorization backend supports variable expansion when checking permission on topics. It
supports JWT claims whose value is a plain string, plus the `vhost` variable.

For example, a user connected with the token below to the vhost `prod` should have a write
permission on all exchanges starting with `x-prod-`, and any routing key starting with `u-bob-`:

```json
{
  "sub" : "bob",
  "scope" : [ "rabbitmq.write:*/x-{vhost}-*/u-{sub}-*" ]
}
```

### Scope and tags {#scope-and-tags}

Users in RabbitMQ can have [tags associated with them](./access-control#user-tags). Tags are used to
[control access to the management plugin](./management#permissions).

In the OAuth context, tags can be added as part of the scope, using a format like
`<resource_server_id>.tag:<tag>`. For example, if the `resource_server_id` is "my_rabbit", a scope
to grant access to the management plugin with the `monitoring` tag will be
`my_rabbit.tag:monitoring`.

## Basic usage {#basic-usage}

### Configure OAuth 2.0 provider's issuer {#configure-issuer}

Before RabbitMQ 3.13, users had to either configure the JWKS endpoint (that is
`auth_oauth2.jwks_url` variable) or statically
[configure the signing keys](#configure-signing-keys). Now, users only need to configure the OpenID
Provider's **issuer** URL and from this URL RabbitMQ downloads the OpenID Provider configuration
which includes the JWKS endpoint in addition to other endpoints which will be useful in other
contexts.

Usually, this **issuer** URL is the same URL configured in the management plugin
(`management.oauth_provider_url`). From now on, you only need to configure a single URL, specified
by the `auth_oauth2.issuer` variable. Except in edge cases where the **issuer** URL does not host
the login page. In that cases, the user configures the login page in the
`management.oauth_provider_url` variable.

Sample configuration using issuer:

```ini
auth_oauth2.resource_server_id = my_rabbit_server
auth_oauth2.issuer = https://my-idp-provider/somerealm
```

Sample configuration which configures the jwks_url rather than the issuer:

```ini
auth_oauth2.resource_server_id = my_rabbit_server
auth_oauth2.jwks_url = "https://my-jwt-issuer/jwks.json
```

:::info
If you have both endpoints configured, RabbitMQ uses `jwks_url` because it does not need to discover
it via the `issuer` url.
:::

:::info
**Note about TLS variables for the `jwks_url` or the `issuer` url**: TLS variable such as the
`cacertfile` are configured as follows regardless which url we are using:
:::

```ini
...
auth_oauth2.https.cacertfile = /opts/certs/cacert.pem
...
```

**VERY IMPORTANT**: Since RabbitMQ 3.13, if `auth_oauth2.https.peer_verification` variable is not
set, RabbitMQ sets it to `verify_peer` as long as there are trusted certificates installed in the OS
or the user configured `auth_oauth2.https.cacertfile`.

### Configure signing keys {#configure-signing-keys}

Currently, it is very rare you configure RabbitMQ with signing keys, when RabbitMQ can automatically
download them as explained in the previous section. However, RabbitMQ supports those edge cases
where you need to statically configure the signing keys, or when you need to support symmetric
signing keys as opposed to the most widely used asymmetric keys.

The following example uses [Cloud Foundry UAA](https://github.com/cloudfoundry/uaa) as the OAuth 2.0
provider.

To get the signing key from the [OAuth 2.0 provider UAA](https://github.com/cloudfoundry/uaa), use
the
[token_key endpoint](https://docs.cloudfoundry.org/api/uaa/version/4.6.0/index.html#token-key-s) or
[uaac](https://github.com/cloudfoundry/cf-uaac) (the `uaac signing key` command).

The following fields are required: `kty`, `value`, `alg`, and `kid`.

Assuming UAA reports the following signing key information:

```erlang
uaac signing key
  kty: RSA
  e: AQAB
  use: sig
  kid: a-key-ID
  alg: RS256
  value: -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2dP+vRn+Kj+S/oGd49kq
6+CKNAduCC1raLfTH7B3qjmZYm45yDl+XmgK9CNmHXkho9qvmhdksdzDVsdeDlhK
IdcIWadhqDzdtn1hj/22iUwrhH0bd475hlKcsiZ+oy/sdgGgAzvmmTQmdMqEXqV2
B9q9KFBmo4Ahh/6+d4wM1rH9kxl0RvMAKLe+daoIHIjok8hCO4cKQQEw/ErBe4SF
2cr3wQwCfF1qVu4eAVNVfxfy/uEvG3Q7x005P3TcK+QcYgJxav3lictSi5dyWLgG
QAvkknWitpRK8KVLypEj5WKej6CF8nq30utn15FQg0JkHoqzwiCqqeen8GIPteI7
VwIDAQAB
-----END PUBLIC KEY-----
  n: ANnT_r0Z_io_kv6BnePZKuvgijQHbggta2i30x-wd6o5mWJuOcg5fl5oCvQjZh15IaPar5oXZLHcw1bHXg5YSiHXCFmnYag83bZ9YY_9tolMK4R9G3eO-YZSnLImfqMv7HYBoAM75pk0JnTKhF6ldgfavShQZqOAIYf-vneMDNax_ZMZdEbzACi3vnWqCByI6JPIQju
      HCkEBMPxKwXuEhdnK98EMAnxdalbuHgFTVX8X8v7hLxt0O8dNOT903CvkHGICcWr95YnLUouXcli4BkAL5JJ1oraUSvClS8qRI-Vino-ghfJ6t9LrZ9eRUINCZB6Ks8Igqqnnp_BiD7XiO1c
```

it translates into the following configuration (in the [advanced RabbitMQ config format](./configure)):

```ini
auth_oauth2.resource_server_id = my_rabbit_server
auth_oauth2.signing_keys.a-key-ID = /path-to-signing-key-pem-file
```

If a symmetric key is used, the configuration looks like this:

```erlang
[
  {rabbitmq_auth_backend_oauth2, [
    {resource_server_id, <<"my_rabbit_server">>},
    {key_config, [
      {signing_keys, #{
        <<"a-key-ID">> => {map, #{<<"kty">> => <<"MAC">>,
                                  <<"alg">> => <<"HS256">>,
                                  <<"value">> => <<"my_signing_key">>}}
      }}
    ]}
  ]},
].
```

### Use a different token field for the scope {#use-different-token-field}

By default the plugin looks for the `scope` key in the token, you can configure the plugin to also
look in other fields using the `extra_scopes_source` variable. Values format accepted are scope as
**string** or **list**

```ini
auth_oauth2.resource_server_id = my_rabbit_server
auth_oauth2.additional_scopes_key = my_custom_scope_key
```

Token sample:

```ini
{
 "exp": 1618592626,
 "iat": 1618578226,
 "aud" : ["my_id"],
 ...
 "scope_as_string": "my_id.configure:*/* my_id.read:*/* my_id.write:*/*",
 "scope_as_list": ["my_id.configure:*/*", "my_id.read:*/*", "my_id.write:*/*"],
 ...
 }
```

### Preferred username claims {#preferred-username-claims}

The username associated with the token must be available to RabbitMQ so that this username is
displayed in the RabbitMQ Management UI. By default, RabbitMQ searches for the `sub` claim first,
and if it is not found, RabbitMQ uses the `client_id`.

Most authorization servers return the user's GUID in the `sub` claim instead of the user's username
or email address, anything the user can relate to. When the `sub` claim does not carry a
*user-friendly username*, you can configure one or several claims to extract the username from the
token.

Example `rabbitmq.conf` configuration:

``` ini
# ...
auth_oauth2.resource_server_id = rabbitmq
auth_oauth2.preferred_username_claims.1 = user_name
auth_oauth2.preferred_username_claims.2 = email
# ...
```

In the example configuration, RabbitMQ searches for the `user_name` claim first and if it is not
found, RabbitMQ searches for the `email`. If these are not found, RabbitMQ uses its default lookup
mechanism which first looks for `sub` and then `client_id`.

### Rich Authorization Request {#rich-authorization-request}

The [Rich Authorization Request](https://oauth.net/2/rich-authorization-requests/) extension
provides a way for OAuth clients to request fine-grained permissions during an authorization
request. It moves away from the concept of scopes that are text labels and instead defines a more
sophisticated permission model.

RabbitMQ supports JWT tokens compliant with the extension. Below is a sample example section of JWT
token:

```javascript
{
  "authorization_details": [
    {
      "type" : "rabbitmq",
      "locations": ["cluster:finance/vhost:production-*"],
      "actions": [ "read", "write", "configure"  ]
    },
    {
      "type" : "rabbitmq",
      "locations": ["cluster:finance", "cluster:inventory" ],
      "actions": ["administrator" ]
    }
  ]
}
```

The token above contains two permissions under the attribute `authorization_details`. Both
permissions are meant for RabbitMQ servers with `resource_server_type` set to `rabbitmq`. This field
identifies RabbitMQ-specific permissions.

The first permission grants `read`, `write` and `configure` permissions to any queue and/or exchange
on any virtual host whose name matches the pattern `production-*`, and that reside in clusters whose
`resource_server_id` contains the string `finance`. The `cluster` attribute's value is also a
regular expression. To match exactly the string `finance`, use `^finance$`.

The second permission grants the `administrator` user tag in two clusters, `finance` and
`inventory`. Other supported user tags as `management`, `policymaker` and `monitoring`.

#### Type field

In order for a RabbitMQ node to accept a permission, its value must match that node's
`resource_server_type` variable's value. A JWT token may have permissions for multiple resource
types.

#### Locations field

The `locations` field can be either a string containing a single location or a Json array containing
zero or many locations.

A location consists of a list of key-value pairs separated by forward slash `/` character. Here is
the format:

```bash
cluster:<resource_server_id_pattern>[/vhost:<vhost_pattern>][/queue:<queue_name_pattern>|/exchange:<exchange_name_pattern>][/routing-key:<routing_key_pattern>]
```

Any string separated by `/` which does not conform to `<key>:<value>` is ignored. For instance, if
your locations start with a prefix, for example `vrn/cluster:rabbitmq`, the `vrn` pattern part is
ignored.

The supported location's attributed are:

- `cluster`: This is the only mandatory attribute. It is a wildcard pattern which must match
  RabbitMQ's `resource_server_id` otherwise the location is ignored.
- `vhost`: This is the virtual host you are granting access to. It also a wildcard pattern. If not
  specified, `*` will be used.
- `queue`|`exchange`: queue or exchange name pattern. The location grants the permission to a set of
  queues (or exchanges) that match it. One location can only specify either `queue` or `exchange`
  but not both. If not specified, `*` will be used
- `routing-key`: this is the routing key pattern the location grants the permission to. If not
  specified, `*` will be used

For more information about wildcard patterns, check the section
[Scope-to-Permission Translation](#scope-translation).

#### Actions field

The `actions` field can be either a string containing a single action or a JSON array containing
zero or many actions.

The supported actions map to either [RabbitMQ permissions](./access-control#authorisation):

- `configure`
- `read`
- `write`

Or RabbitMQ user tags:

- `administrator`
- `monitoring`
- `management`
- `policymaker`

#### Rich-Permission to scope translation

Rich Authorization Request permissions are translated into JWT token scopes that use the
aforementioned convention using the following algorithm:

For each location found in the `locations` where the `cluster` attribute matches the current
RabbitMQ server's `resource_server_id`:

- For each location found in the `locations` field where the `cluster` attribute matches the current
  RabbitMQ node's `resource_server_id`, the plugin extracts the `vhost`, `queue` or `exchange` and
  `routing_key` attributes from the location. If the location does not have any of those attributes,
  the default value of `*` is assumed. Out of those values, the following scope suffix will be
  produced:

  ```ini
  scope_suffix = <vhost>/<queue>|<exchange>/<routing-key>
  ```

  - For each action found in the `actions` field:

    If the action is not a known user tag, the following scope is produced out of it:

    ```ini
      scope = <resource_server_id>.<action>:<scope_suffix>
    ```

    For known user tag actions, the following scope is produced:

    ```ini
      scope = <resource_server_id>.<action>
    ```

The plugin produces permutations of all `actions` by all `locations` that match the node's
configured `resource_server_id`.

In the following RAR example

```javascript
{
  "authorization_details": [
    { "type" : "rabbitmq",
      "locations": ["cluster:finance/vhost:primary-*"],
      "actions": [ "read", "write", "configure"  ]
    },
    { "type" : "rabbitmq",
      "locations": ["cluster:finance", "cluster:inventory" ],
      "actions": ["administrator" ]
    }
  ]
}
```

if RabbitMQ nodes `resource_server_id` is equal to `finance`, the plugin computes the following sets
of scopes:

- `finance.read:primary-*/*/*`
- `finance.write:primary-*/*/*`
- `finance.configure:primary-*/*/*`
- `finance.tag:administrator`

## Advanced usage {#advanced-usage}

### Use default OAuth 2.0 provider {#use-oauth-provider}

As long as you have only one OAuth 2.0 provider, you can skip this advanced usage although you can
use it.

Under the [basic usage](#configure-issuer) section, you configured the `issuer` URL or maybe the
`jwks_url` along with the TLS variables if needed. This advanced usage configures everything
relative to the OAuth provider into a dedicated configuration.

Here is an example configuration that uses `issuer` to configure the identity provider's URL:

```ini
auth_oauth2.resource_server_id = rabbitmq-prod
auth_oauth2.scope_prefix = rabbitmq.
auth_oauth2.issuer = https://prodkeycloak:8080/realm/prod
auth_oauth2.https.cacertfile = /opts/certs/prodcacert.pem
```

The equivalent configuration where the identity provider is configured under
`auth_oauth2.oauth_providers` variable is:

```ini
auth_oauth2.resource_server_id = rabbitmq-prod
auth_oauth2.scope_prefix = rabbitmq.
auth_oauth2.default_oauth_provider = prodkeycloak

auth_oauth2.oauth_providers.prodkeycloak.issuer = https://prodkeycloak:8080/realm/prod
auth_oauth2.oauth_providers.prodkeycloak.https.cacertfile = /opts/certs/prodcacert.pem
```

This latter configuration is more relevant when users present tokens which are issued or signed by
different OAuth 2.0 providers. However, one can still use it provided
`auth_oauth2.default_oauth_provider` is set.

### Configure OAuth 2.0 provider's end_session_endpoint {#configure-end-session-endpoint}

This advanced setting is only required when the
[OpenId Connect Discovery endpoint](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfigurationRequest)
does not return an `end_session_endpoint` and you want Single Logout functionality. In other words,
when the user logs out from the management UI it is also logged out from the OAuth Provider.

:::info
If the [OpenId Connect Discovery endpoint](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfigurationRequest)
response does include an `end_session_endpoint`, the management UI uses it over the configured
endpoint.
:::

Here is an example configuration that sets `end_session_endpoint`:

```ini
auth_oauth2.resource_server_id = rabbitmq-prod
auth_oauth2.scope_prefix = rabbitmq.
auth_oauth2.issuer = https://prodkeycloak:8080/realm/prod
auth_oauth2.end_session_endpoint = https://prodkeycloak:8080/realm/prod/logout
```

The equivalent configuration where the identity provider is configured under
`auth_oauth2.oauth_providers` variable is:

```ini
auth_oauth2.resource_server_id = rabbitmq-prod
auth_oauth2.scope_prefix = rabbitmq.
auth_oauth2.default_oauth_provider = prodkeycloak

auth_oauth2.oauth_providers.prodkeycloak.issuer = https://prodkeycloak:8080/realm/prod
auth_oauth2.oauth_providers.prodkeycloak.end_session_endpoint = https://prodkeycloak:8080/realm/prod/logout
```

### Configure multiple resource servers {#multiple-resource-servers}

Usually, all users that access a RabbitMQ cluster are registered within the same identity provider.
Likewise, all tokens targeting the same RabbitMQ cluster also carry the same *audience*. In other
words, all users reference a RabbitMQ cluster with the same resource name which must match the value
of the `auth_oauth2.resource_server_id` variable.

However, there are some use-cases where RabbitMQ is accessed by users registered in different
identity providers or tokens are issued for the same RabbitMQ installation but with different
*Audience*(s). For these use-cases, RabbitMQ OAuth 2.0 plugin and the Management plugin can be
configured with multiple OAuth 2.0 resources.

The following is the OAuth 2.0 plugin configuration for two resources with the ids: `rabbit_prod`
and `rabbit_dev`. Both resources (also known as *audience*) are managed by the same Identity
Provider whose issuer is `http//some_idp_url`.

```ini
auth_oauth2.issuer = http//some_idp_url
auth_oauth2.scope_prefix = rabbitmq.

auth_oauth2.resource_servers.1.id = rabbit_prod
auth_oauth2.resource_servers.2.id = rabbit_dev
auth_oauth2.resource_servers.2.scope_prefix = dev-rabbitmq.
```

All resource servers share the variables you set so far under `auth_oauth2.` such as `scope_prefix`.
However, they can override them. Here are the list of variables a resource server can override:

- `id` - This is the actual resource identifier carried in the `audience` field of a token. If
  omitted, the value is the index. For example, given the
  `auth_oauth2.resource_servers.prod.scope_prefix` variable, the `id` would be `prod`.
- `scope_prefix`
- `additional_scopes_key`
- `resource_server_type`
- `oauth_provider_id` - This is the identifier of the OAuth provider. It is configured in RabbitMQ.
  It provides all the variables to contact the authorization server and discover all its endpoints,
  such as the `jwks_uri` to download the signing keys to validate the token.

  If this variable is omitted, RabbitMQ looks up the default Authorization Provider's id in the
  variable `auth_oauth2.default_oauth_provider`, and if it is also omitted, RabbitMQ uses
  `auth_oauth2.issuer` or `auth_oauth2.jwks_url` to download the signings keys to validate the
  token.

The list of supported resource servers is the combination of `auth_oauth2.resource_servers` and
`auth_oauth2.resource_server_id`. You can use both or only one of them.

:::info

There is an [example](./oauth2-examples-multiresource) that demonstrate how to use multiple OAuth 2
resources.

:::

A list of [all the configurable variables](#multiple-resource-servers-configuration) for each OAuth
Provider is documented in a separate section.

### Configure multiple OAuth 2.0 providers {#multiple-oauth-providers}

It only makes sense to set multiple OAuth 2.0 providers if there are
[multiple resources configured](#multiple-resource-servers).

This is the configuration used in the previous section but modified to use multiple OAuth 2.0
providers:

```ini
auth_oauth2.scope_prefix = rabbitmq.

auth_oauth2.resource_servers.1.id = rabbit_prod
auth_oauth2.resource_servers.1.oauth_provider_id = prod
auth_oauth2.resource_servers.2.id = rabbit_dev
auth_oauth2.resource_servers.2.oauth_provider_id = dev
auth_oauth2.resource_servers.2.scope_prefix = dev-rabbitmq.

auth_oauth2.oauth_providers.prod.issuer = https://rabbit_prod:8080
auth_oauth2.oauth_providers.prod.https.cacertfile = /opts/certs/prod.pem
auth_oauth2.oauth_providers.dev.issuer = https://rabbit_dev:8080
auth_oauth2.oauth_providers.dev.https.cacertfile = /opts/certs/dev.pem
```

A list of [all the configurable variables](#multiple-resource-servers-configuration) for each OAuth
Provider is documented in a separate section.

## Examples {#examples}

The [RabbitMQ OAuth 2.0 Auth backend examples](./oauth2-examples) contain many example configuration
files, that can be used to set up several OAuth 2.0 providers, including UAA, Auth0, and Azure, and
issue tokens, which can be used to access RabbitMQ resources.
