# Vault Scala Library
[![Circle CI](https://circleci.com/gh/janstenpickle/scala-vault.svg?style=svg)](https://circleci.com/gh/janstenpickle/scala-vault)

Scala library for working with [Hashicorp Vault](https://www.vaultproject.io/).

This library has three modules:

|Name|Description|Download|
|---|---|---|
|**Core** | Basic client capable of obtaining a token using an App ID, supports getting and setting of secrets  | [![Download](https://api.bintray.com/packages/janstenpickle/maven/vault-core/images/download.svg)](https://bintray.com/janstenpickle/maven/vault-core/_latestVersion) |
| **Auth** | Functions to authenticate a user using userpass authentication and token verification | [![Download](https://api.bintray.com/packages/janstenpickle/maven/vault-auth/images/download.svg)](https://bintray.com/janstenpickle/maven/vault-auth/_latestVersion)|
| **Manage** | Functions for managing auth modules, mounts and policies | [![Download](https://api.bintray.com/packages/janstenpickle/maven/vault-manage/images/download.svg)](https://bintray.com/janstenpickle/maven/vault-manage/_latestVersion) |

## Install with SBT
Add the following to your sbt `project/plugins.sbt` file:
```scala
addSbtPlugin("me.lessis" % "bintray-sbt" % "0.3.0")
```
Then add the following to your `build.sbt`
```scala
resolvers += Resolver.bintrayRepo("janstenpickle", "maven")
libraryDependencies += "janstenpickle.vault" %% "vault-core" % "0.3.1"
libraryDependencies += "janstenpickle.vault" %% "vault-auth" % "0.3.1"
libraryDependencies += "janstenpickle.vault" %% "vault-manage" % "0.3.1"
```
## Usage
Simple setup:
```scala
import java.net.URL

import janstenpickle.vault.core.AppId
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.core.WSClient

val config = VaultConfig(WSClient(new URL("https://localhost:8200")), "token")

val appIdConfig = VaultConfig(WSClient(new URL("https://localhost:8200")), AppId("appId", "userId"))
```
### WSClient
This library uses the [Dispatch](http://dispatch.databinder.net/Dispatch.html), a lightweight async HTTP client to communicate with Vault.

### Responses
All responses from Vault are wrapped in an asynchronous [Result](http://github.com/albertpastrana/uscala). This allows any errors in the response are captured separately from the failure of the underlying future.

### Reading and writing secrets
```scala
import java.net.URL

import janstenpickle.vault.core.AppId
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.core.WSClient
import janstenpickle.vault.core.Secrets


val config = VaultConfig(WSClient(new URL("https://localhost:8200")), AppId("appId", "userId"))

val secrets = Secrets(config, "secret")
```
#### Getting a secret
```scala
val response = secrets.get("some_secret")
// Unsafely evaluate the Task
println(response.unsafePerformSyncAttempt)
```
#### Setting a secret
```scala
val response = secrets.set("some_secret", "some_value")
```
#### Setting a secret under a different sub key
```scala
val response = secrets.set("some_secret", "some_key", "some_value")
```
#### Setting a secret as a map
```scala
val response = secrets.set("some_secret", Map("k1" -> "v1", "k2" -> "v2"))
```
#### Getting a map of secrets
```scala
val response = secrets.getAll("some_secret")
```
#### Listing all secrets
```scala
val response = secrets.list
```

### Authenticating a username/password
```scala
import java.net.URL

import janstenpickle.vault.core.WSClient
import janstenpickle.vault.auth.UserPass


val userPass = UserPass(WSClient(new URL("https://localhost:8200")))

val ttl = 10 * 60
val response = userPass.authenticate("username", "password", ttl)
```
The [response](auth/src/main/scala/janstenpickle/vault/auth/UserPass.scala#L23:L27) will contain the fields as per the [Vault documentation](https://www.vaultproject.io/docs/auth/userpass.html).

#### Multitenant username/password auth
This requires that `userpass` authentication has been enabled on separate path to the default of `userpass`. Instructions of how to do this are documented below. By doing this credientials for different tenants may be stored separately within Vault.
```scala
val response = userPass.authenticate("username", "password", ttl, "clientId")
```

### Validating a Token
```scala
import java.net.URL

import janstenpickle.vault.core.AppId
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.core.WSClient
import janstenpickle.vault.auth.Token


val config = VaultConfig(WSClient(new URL("https://localhost:8200")), AppId("appId", "userId"))

val token = Token(config)

val response = token.validate("some_vault_token")
```
The [response](auth/src/main/scala/janstenpickle/vault/auth/Token.scala#L22-L30) will contain the fields as per the lookup response in the [Vault documentation](https://www.vaultproject.io/docs/auth/token.html)

The token response will also have optional `username` and `client` values which are provided as best effort. If the token was obtained by using `userpass` authentication then they will be present. The `client` value will first look in the metadata for a client key and fallback to a section of the `path` value, whereas the `username` value will just look in the metadata.

## Managing Vault
This library also provides some limited management functionality for Vault around authenctiation, mounts and policy.
### Authentication Management
```scala
import java.net.URL

import janstenpickle.vault.core.AppId
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.core.WSClient
import janstenpickle.vault.manage.Auth


val config = VaultConfig(WSClient(new URL("https://localhost:8200")), AppId("appId", "userId"))

val auth = Auth(config)

// enable an auth backend
val enable = auth.enable("auth_type")

// disable an auth backend
val disable = auth.disable("auth_type")
```
The enable function can also take an optional mount point and description, the mount point is useful when setting up multitenant `userpass` backend as the mount point will correspond to the client ID.
```scala
val response = auth.enable("auth_type", Some("client_id"), Some("description"))
```


# Example Usage - Multitenant Token Authentication Service
Using this library it is very simple to set up a token authentication service for ReST API authentication made up of three components:

* Vault
* Thin authentication endpoint
* API service

The sequence diagram below shows how this may be constructed:

![Auth Sequence](https://i.imgur.com/nu6Gs77.png)

### Code Examples for Token Authentication Service

The exmaples below show how clients can be set up, users authenticated and tokens validated:

#### Client Administration
```scala
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.manage.Auth

class ClientAuth(config: VaultConfig) {
  val auth = Auth(config)
  def create(clientId: String, clientName: String): AsyncResult[WSResponse] = auth.enable("userpass", Some(clientId), Some(clientName))
  def delete(clientId: String): AsyncResult[WSResponse] = auth.disable(clientId)
}
```
#### User Administration
```scala
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.manage.UserPass

class UserAdmin(config: VaultConfig, ttl: Int) {
  val userPass = UserPass(config)
  def create(username: String, password: String, clientId: String, policies: Option[List[String]] = None): AsyncResult[WSResponse] = 
    userPass.create(username, password, ttl, policies, clientId)
  def setPassword(username: String, password: String, clientId: String): AsyncResult[WSResponse] =
    userPass.setPassword(username, password, clientId)
  def setPoliciesUsername: String, policies: List[String], clientId: String): AsyncResult[WSResponse] = 
    userPass.setPolicies(username, policies, clientId)
  def delete(username, clientId: String): AsyncResult[WSResponse] = userPass.delete(username, clientId)
}
```
#### User Authentication
```scala
import janstenpickle.vault.core.WSClient
import janstenpickle.vault.manage.Auth

class UserAuth(wsClient: WSClient, ttl: Int) {
  val userPass = UserPass(wsClient)
  
  // returns only the token
  def auth(username: String, password: String, clientId: String): AsyncResult[String] =
    userPass.authenticate(username, password, ttl, clientId).map(_.client_token)
}
```
#### Token Validation
```scala
import janstenpickle.vault.core.VaultConfig
import janstenpickle.vault.manage.Auth

case class ValidationRepsonse(username: String, clientId: String, policies: Option[List[String]])

class TokenValidation(config: VaultConfig) {
  val token = Token(config)
  
  //only return the username, clientId and policies from the response
  def validate(userToken: String): AsyncResult[Option[ValidationResponse]] = 
    token.validate(userToken).map(tr => 
      for {
        username <- tr.username
        clientId <- tr.client
      } yield ValidationRepsonse(username, clientId, tr.policies)
    )
}
```
