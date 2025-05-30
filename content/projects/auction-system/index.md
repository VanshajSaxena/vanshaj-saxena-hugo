---
title: "Auction Hub"
draft: false
showtoc: false
weight: 93
---

**Auction Hub** is a backend service that is designed to serve a system of online
auctions, basically the buying, selling, and bidding of auction items in real
time, with a comprehensive focus on security of the application and its scalability.

---

## Project Structure

Using [Spring Boot](https://spring.io/projects/spring-boot),
[OpenAPI](https://www.openapis.org/what-is-openapi) and
[eclipse.jdt.ls](https://github.com/eclipse-jdtls/eclipse.jdt.ls), the Auction
System is designed to be stable, secure and scalable. It follows an **API-first**
approach to allow for its client to know exactly what to expect from the API.

The project follows the following structure which allows it to be modular and
maintainable to reducing cost of refactoring in future along with a test suit
that tests the essential components of the application.

```sh
auction-system/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── auction/
│   │   │           └── system/     # Main application package
│   │   │               ├── config/         # Spring Security configuration classes
│   │   │               ├── controllers/    # API Controllers (delegates to generated interfaces)
│   │   │               ├── entities/       # JPA entities
│   │   │               ├── exception/      # Custom exception handling
│   │   │               ├── filters/        # Request filters (e.g. JWT authentication filter)
│   │   │               ├── mappers/        # MapStruct mappers
│   │   │               ├── repositories/   # Spring Data JPA repositories
│   │   │               ├── security/       # Security related components
│   │   │               ├── services/       # Business logic interfaces
│   │   │               │   └── impl/       # Interface implementations
│   │   │               └── AuctionSystemApplication.java # Spring Boot main class
│   │   └── resources/
│   │       ├── api/
│   │       │   └── openapi.api-description.yaml # API description file
│   │       ├── static/
│   │       ├── templates/
│   │       └── application.yaml            # Application properties
│   └── test/                               # Test sources
│       └── java/
│           └── com/
│               └── auction/
│                   └── system/
│                       ├── services/
│                       │   └── impl/       # Service related tests
│                       └── testutils/      # Test utility classes
├── mvnw                                    # Maven wrapper executable (Linux/MacOS)
├── mvnw.cmd                                # Maven wrapper executable (Windows)
└── pom.xml                                 # Maven Project Object Model
```

The [source-code](https://github.com/VanshajSaxena/auction-system) can be found
on my GitHub Profile.

---

## OpenAPI

It uses OpenAPI spec 3.0.3 to define its contract. The API contract can be
viewed interactively either by running the server offline and making a request
to the `swagger.html` endpoint, which redirects to a Swagger UI page running
locally. Or my directly inspecting the
[openapi.api-description.yaml](https://github.com/VanshajSaxena/auction-system/blob/master/src/main/resources/api/openapi.api-description.yaml)
fine in the project source code.

This approach allows the consumers of the API to beforehand know what to expect
from the API and even generate client stubs for mocking and prototyping,
saving time a cost of development.

The project workflow involves interacting with OpenAPI description file, if an
endpoint needs to be added, the API description file can be updated to reflect
that and the server stubs will be generated with
[openapi-generator](https://github.com/OpenAPITools/openapi-generator) during
the `generate-source` phase of Maven `default` lifecycle.

## Security

The API is secured using [Spring
Security](https://spring.io/projects/spring-security) with best practices in
mind. It uses stateless authentication using JWTs (`access_token` and
`id_token`). To enable SSO using third-party authorization and openid providers
there are callback endpoints like `/auth/{provider}/callback` that expects an
`Authorization: Bearer <token>` header in the `POST` request.

The security scheme for these endpoints in OpenAPI description file looks like
this:

```yaml
paths:
  /auth/google/callback:
    post:
      security:
        - oauth2google: []

  /auth/apple/callback:
    post:
      security:
        - oauth2apple: []

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      description: Authorization using JWT issued by the backend.
      bearerFormat: JWT
    oauth2google:
      type: http
      scheme: bearer
      description: Authentication using ID Tokens from the Google Authorization Server.
      bearerFormat: ID Token (Google)
    oauth2apple:
      type: http
      scheme: bearer
      description: Authentication using ID Tokens from the Apple Authorization Server.
      bearerFormat: ID Token (Apple)
security:
  - bearerAuth: []
```

To have a deeper look at the security scheme have a look at the
`SecurityConfig.java` and `OAuth2SecurityConfig` in the
`com.auction.system.config` package.
