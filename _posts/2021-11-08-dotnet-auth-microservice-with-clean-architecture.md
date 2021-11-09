---
layout: post
title:  "An Auth Microservice with Clean Architecture"
categories:  example
tags: aspnetcore dotnet c# microservice clean hexagonal architecture jwt
comments: true
---

The code of this project is available at [https://github.com/veglos/dotnet-auth-microservice](https://github.com/veglos/dotnet-auth-microservice)

## Table of contents
1. [A Clean Architecture](#a-clean-architecture)
2. [The Hexagonal Architecture](#the-hexagonal-architecture)
   1. [Use Cases](#use-cases)
3. [The Auth Microservice](#the-auth-microservice)
   1. [What about an API Gateway?](#what-about-an-api-gateway)
   2. [OAuth 2.0, OpenID Connect, and Json Web Token](#oauth-openid-jwt)
   3. [Access Token vs Refresh Token](#access-token-vs-refresh-token)
   4. [JWT, JWS, and JWE](#jwt-jws-and-jwe)
   5. [How do other microservices know the Access Token is legit?](#how-do-other-microservices-know-the-access-token-is-legit)
4. [The Project Structure](#the-project-structure)
   1. [Auth.Domain](#auth-domain)
   2. [Auth.Application](#auth-application)
   3. [Auth.Infrastructure](#auth-infrastructure)
   4. [Auth.API](#auth-api)
5. [Conclusion](#conclusion)

## A Clean Architecture <a name="a-clean-architecture"></a>

---

In 2012 Robert C. Martin published a blog article called [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) where he wrote about the common features of different systems, like independence of the framework, independence of the database, independence of the UI, etc. Then in 2017 Martin published the book [Clean Architecture](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164) that elaborate a further analysis on how a good architecture should be, but most importantly «why».

![/clean-architecture](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/clean-architecture.jpg)

_Figure 1.1: Robert C. Martin's Clean Architecture Diagram_

## The Hexagonal Architecture <a name="the-hexagonal-architecture"></a>

---


The Hexagonal Architecture was created by Alistair Cockburn in 2005. He has a complete explanation in [his site](https://alistair.cockburn.us/hexagonal-architecture/).

Also known as the Ports & Adapters architecture pattern, it focuses on the [Dependency Injection (DI) principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle) in order to

<!-- <style type="text/css">
    ol { list-style-type: lower-alpha; }
</style> -->

<ol type="a">
  <li>delay the decision of what technology implementation to use, and</li>
  <li>change technology implementations later easily</li>
</ol>

This pattern relies mainly on four concepts: primary port, primary adapter, secondary port, and secondary adapter. They can be found with different names in literature. See table 2.1.


| Primary Port | Primary Adapter  | Secondary Port | Secondary Adapter |
|---------------|------------------|----------------|-------------------|
| Incoming Port | Incoming Adapter | Outgoing Port  | Outgoing Adapter  |
| Driving Port | Driving Adapter  | Driven Port    | Driven Adapter    |
| Input Port | Input Adapter    | Output Port    | Output Adapter    |

_Table 2.1: Different names for port and adapters according to different literature_

**Ports** -both primary and secondary- are just the interfaces that declare what the implementation must do. Conversely, **Adapters** -both primary and secondary- are the respective implementations of the ports.

**Primary** represents a mean to «run» or «drive» the application. It could be a request to get/send data to/from the app or to start a background process. A primary port shows to the «outside world» how the application can be called. On the other hand, **Secondary** represents a mean for the application to interact with resources, like calling a web service or fetching data from a database.

In summary:

* **Primary Port:** Interface exposed by the application for it to be called.
* **Primary Adapter:** Implementation of the Primary Port to call the application.
* **Secondary Port:** Interface exposed by the application for external resources to be called.
* **Secondary Adapter:** Implementation of the Secondary Port to access resources.

I made the diagram shown in figure 2.1 to illustrate the difference.

![/hexagonal-architecture](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/hexagonal-architecture.jpg)

_Figure 2.1: Hexagonal Architecture diagram_

## The Design <a name="the-design"></a>

---

From the previous diagrams in figure 1.1 and figure 2.1, I designed the diagram displayed in figure 3.1. I kept the same color scheme to identify the boundaries.

![/clean-1](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/clean-1.jpg)

_Figure 3.1: Auth microservice architecture model_

The solution is composed of four projects:
* **Auth.Domain:** Contains the enterprise business rules in classes (mostly [POCO](https://en.wikipedia.org/wiki/Plain_old_CLR_object)). It doesn't have any dependencies except the .NET Framework itself.
* **Auth.Application:** Contains the application business rules, represented by the use cases. Depends only on the Auth.Domain.
* **Auth.Infrastructure:** Contains the specific implementation of services and repositories. Depends on the Auth.Application and Auth.Domain.
* **Auth.API:** Contains the controllers to be called by the user, and in turn, call the use cases. It also hosts the Microsoft Dependency Injection Container which explains why it depends not only on Auth.Domain and Auth.Application, but also Auth.Infrastructure *.

*_note_: It is possible to avoid the Auth.Infrastructure dependency by creating a DI Container project. I believe it's not worth the inconvenience in this situation, but it's totally possible.

![/clean-2](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/clean-2.jpg)

_Figure 3.2: Dependency relationship between projects_

### Use Cases <a name="use-cases"></a>

Use cases are the heart of the application, they execute the business rules. Here are some quotes from Uncle Bob:

> _Indeed, this is the first concern of the architect, and the first priority of the architecture. The architecture must support the use case_
>
>– Martin, R.C. (2017). In Chapter 16 Independence. Clean Architecture (p. 148). Prentice Hall.

> _The most important thing a good architecture can do to support behavior is to clarify and expose that behavior so that the intent of the system is visible at the architectural level_
>
>– Martin, R.C. (2017). In Chapter 16 Independence. Clean Architecture (p. 148). Prentice Hall.


> _[...], use cases are narrow vertical slices that cut through the horizontal layers of the system. Each use case uses some UI, [...] business rules [...], and some database functionality_
>
>– Martin, R.C. (2017). In Chapter 16 Independence. Clean Architecture (p. 152). Prentice Hall.

![/clean-3](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/clean-3.jpg)

_Figure 3.3: Use cases cutting through the horizontal layers_

Our Auth microservice has two main use cases: Login use case and Refresh Token use case.

Given the login credentials, the Login use case returns an Access Token and a Refresh Token. The Access Token can be used by the client application to access other microservices that trust the Auth microservice. The Refresh Token is needed to get a new Access Token. More on this later.

## The Auth Microservice <a name="the-auth-microservice"></a>

---


An Auth Microservice is a centralized authority that grants authentication and authorization (Auth for short) to a user to allow her/him access to a resource provided by other systems (i.e. other microservices) that trust said authority.

Nowadays it is imperative for most microservices to have authentication and authorization, and while it is possible to implement them in every microservice, it is far more convenient to rely on an Auth Microservice. We don't want to login (ask for credentials) in every single microservice. Let the microservice focus on the scope they were meant to handle, nothing more.

![/auth-microservice](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/auth-microservice.jpg)

*Figure 4.1: The Auth Microservice handles the authentication and authorization of the user/client*

### What about an API Gateway? <a name="what-about-an-api-gateway"></a>

[The API Gateway](https://microservices.io/patterns/apigateway.html) may or may not handle authentication and authorization, and it can have more responsibilities than that, like response caching, circuit breaker, load balancing, etc., but it's main purpose is to be a single entry point to the entire system.

### OAuth 2.0, OpenID Connect, and Json Web Token <a name="oauth-openid-jwt"></a>

There are Id Tokens and Access Tokens. Id Tokens hold information about who the user is (claims), and **the intended recipient is the client application** (i.e. to show "Welcome Carlos!", etc.). 
On the other hand, Access Tokens hold information about what can be done (scopes) in a resource (i.e. fetch the user's photos, etc.), therefore **the intended recipient of such token is the user's resource**. Figure 5.1 depicts how an Access Token works.

The Id Token is defined by the [OpenID Connect Specification](https://openid.net/connect/), whereas the Access Token is defined by the [OAuth 2.0 Specification](https://oauth.net/2/). The former must be sent in a [JSON Web Token (JWT)](https://jwt.io/) format, the latter can be any string, including JWT.

**It is not the scope of this article to deal with the Client App (or any front-end project for that matter), hence I dealt mostly with Access Tokens, not so much ID Tokens.**

![auth-sequence](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/auth-sequence.jpg)

*Figure 4.2: Sequence diagram of the process of authorization by Access Token*

### Access Token vs Refresh Token <a name="access-token-vs-refresh-token"></a>

There is a third and last type of token called the Refresh Token.

As previously mentioned, the Access Token allows the user to access a resource, however it has a short lifespan, and depending on the system, it could last between 5 minutes and 1 hour. This is important because if for some reason the Access Token get stolen, the attacker could only make use of it for a short time.

However, we don't want to keep asking the user for her/his credentials in order ot get a new Access Token every 5 minutes. That's why we issue a Refresh Token, which has a long lifespan, usually between 1 day and 1 week, and can be used by the client application to get a brand new Access Token without prompting the user with a login screen every time.

But could the Refresh Token also be stolen> Yes, but since a Refresh Token is assigned to a single user, it can be disabled and be forced to re-login again, preventing the attacker from getting a new Access Token.

Access Tokens cannot be disabled, unless we change the Private-Public key pairs, but that would disable every single token in circulation.

### JWT, JWS, and JWE <a name="jwt-jws-and-jwe"></a>

Prabath Siriwardena does a wonderful explanation in his article [JWT, JWS and JWE for Not So Dummies!](https://medium.facilelogin.com/jwt-jws-and-jwe-for-not-so-dummies-b63310d201a3). Basically, as the [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519) says, JWT is a mean of representing claims (or scopes) between two parties, but the actual implementation occurs as a JSON Web Signature (JWS) or a JSON Web Encryption (JWE), or a combination of both. JWS encodes and signs the payload, whereas JWE encrypts the payload.

For this project I used JWT with JWS, and while it is possible to implement your own JWT library, it is highly recommended to use an already tested and popular library like the ones listed on [https://jwt.io/libraries](https://jwt.io/libraries). I used the Microsoft's System.IdentityModel.Tokens.Jwt library.

### How do other microservices know the Access Token is legit? <a name="how-do-other-microservices-know-the-access-token-is-legit"></a>

There are two possible ways for a microservice to recognize that the Access Token received is actually from the Auth Microservice and not a malicious impostor (and/or that the payload has not been modified). The two ways are by a **shared secret key** or by a **public-private key pair**.

A shared secret key is used when the Auth Microservice uses **symmetric** cryptography to sign the payload. Another microservice would need to know the same secret key (hence a shared secret) in order to verify if the payload is true.

A public-private key pair is used when the Auth Microservice uses asymmetric cryptography to sign the payload, that is, it still requires a private key to sign it, but the verification can be done with just the public key, which as its name implies, it can be publicly distributed to the world without compromising the private key.

It is safer to keep the signing key (private key) in the Auth Microservice and only share the public key, no matter how much you trust the other microservices. That's why this project uses **asymmetric** cryptography.

![asymmetric](/assets/img/2021-11-08-dotnet-auth-microservice-with-clean-architecture/asymmetric.jpg)

_Figure 4.3: Simplified example of asymmetric cryptography_

## The Project Structure <a name="the-project-structure"></a>

---

As stated before, the solution is made of four projects.

### Auth.Domain <a name="auth-domain"></a>

```bash
Auth.Domain /
├─ User.cs
├─ RefreshToken.cs
├─ Claims.cs
```

The first and most important layer. It contains the enterprise business rules. It is quite simple because the RefreshToken class and the Claims class are part of the User class, so it could have been just a single .cs file.

The scope of the Auth Microservice is narrow. The domain classes are enough to encompass the authentication and authorization.

### Auth.Application <a name="auth-application"></a>

```bash
Auth.Application /
├─ Enums
├─ Exceptions
├─ Ports /
   ├─ Repositories
   ├─ Services
├─ UseCases
   ├─ CreateUser /
   ├─ Login /
   ├─ RefreshToken /
   ├─ SignOut /
   ├─ IUseCase.cs
   ├─ Request.cs
   ├─ Response.cs
```

> _Just as the plans for a house or a library scream about the use cases of those buildings, so should the architecture of a software application scream about the use cases of the application_
>
>– Martin, R.C. (2017). In Chapter 21 Screaming Architecture. Clean Architecture (p. 196). Prentice Hall.

Auth.Application holds the "screaming" part of the architecture that tells us what the system do. Just by looking at the UseCase folder, it is apparent what the application does: Create a user, Log in, refresh a token, and sign out.

```bash
Auth.Application /
├─ Ports /
   ├─ Repositories /
      ├─ IAuthRepository.cs
   ├─ Services /
      ├─ IAuthTokenService.cs
      ├─ ICryptographyService.cs
```

If we zoom in a little bit, the Ports folder declares the interfaces of the implementations that the use cases will need in order to be executed. That means **all ports here are secondary/output ports**.

### Auth.Infrastructure <a name="auth-infrastructure"></a>

```bash
Auth.Infrastructure /
├─ Repositories /
   ├─ MongoDB /
      ├─ Scripts /
      ├─ AuthRepository.cs
      ├─ MongoDbSettings.cs
├─ Services /
   ├─ Cryptography
      ├─ CryptographyService.cs
   ├─ Jwt
      ├─ JwtService.cs
      ├─ JwtSettings.cs

```

This project contains the secondary adapters. Basically, every external infrastructure that is not fundamental for the behavior of the application, like third-party services or databases. For example, changing from SQL Server to PostgreSQL or MongoDB does not change the behavior of the application (asi in, the use cases).

### Auth.API <a name="auth-api"></a>

```bash
Auth.API /
├─ Controllers /
   ├─ AuthController.cs
├─ Program.cs
├─ Startup.cs
├─ appsettings.json
```

This layer in particular has two roles. 

First, it is an HTTP API primary adapter: It receives HTTP requests and converts them into request objects that the application can process.

Second, it is a [Dependency Injection Container](https://en.wikipedia.org/wiki/Dependency_injection) that knows how to instantiate and configure objects in runtime in order to be injected in the application when it requires it. This project uses the Microsoft.Extensions.DependencyInjection library. It is common practice to do this in an ASP.NET Core Web API project, since most application will only have HTTP Requests as inputs, so it's easier to do this right here rather than creating another project for the DI Container. This is the reason why the Auth.API project depends on the Auth.Infrastructure project *.

_*note: If the DI Container where in another project, for instance and Auth.DIContainer project, then the Auth.API project wouldn't need to depend on the Auth.Infrastrutcure project. It's just a matter of convenience why I did it this way._

## Conclusion <a name="conclusion"></a>

---

_There's no silver bullet_. There are many ways to implement a clean architecture, and many more ways to keep improving it forever. The important thing here is to grasp and understand the concepts and know how to identify them. 

We must have a clear-cut boundary between layers and the direction of dependencies. The implementation of new features immediately _"smell weird"_ when the boundaries are not taken into consideration. For example, if the Auth.Application requires to import a third-party library to access an Excel document then there is a clear violation of the dependency rule (the application cannot depend on the infrastructure), and necessary measures are required to correct it.

