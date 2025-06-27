---
title: "Auction Hub"
draft: false
showtoc: false
weight: 93
---

> [**Source Code**](https://github.com/VanshajSaxena/auction-system)

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
│   │   │               ├── controller/     # API Controllers (delegates to generated interfaces)
│   │   │               ├── entity/         # JPA entities
│   │   │               ├── exception/      # Custom exception handling
│   │   │               ├── filter/         # Request filters (e.g. JWT authentication filter)
│   │   │               ├── mapper/         # MapStruct mappers
│   │   │               ├── repository/     # Spring Data JPA repositories
│   │   │               ├── security/       # Security related components
│   │   │               ├── service/        # Business logic interfaces
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
│                       ├── service/
│                       │   └── impl/       # Service related tests
│                       └── testutil/       # Test utility classes
├── mvnw                                    # Maven wrapper executable (Linux/MacOS)
├── mvnw.cmd                                # Maven wrapper executable (Windows)
└── pom.xml                                 # Maven Project Object Model
```

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

---

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
[SecurityConfig.java](https://github.com/VanshajSaxena/auction-system/blob/master/src/main/java/com/auction/system/config/SecurityConfig.java)
and
[OAuth2SecurityConfig](https://github.com/VanshajSaxena/auction-system/blob/master/src/main/java/com/auction/system/config/OAuth2SecurityConfig.java)
in the
[com.auction.system.config](https://github.com/VanshajSaxena/auction-system/tree/master/src/main/java/com/auction/system/config)
package.

## Database and ORM

The application uses an in-memory H2 database for development, but it can be
configured to use any database of the choice (e.g. MySQL).

```yaml
# Example configuration for a local MySQL server
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/auction_system
    username: test
    password: password
```

Along with this the proper JDBC driver for the particular database needs to be
on the class path for the application to connect to the data source
successfully.

```xml
<!-- in project object model (pom.xml) -->
  <dependencies>
    <dependency>
      <groupId>com.mysql</groupId>
      <artifactId>mysql-connector-j</artifactId>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
```

The actual ORM (Object Relational Mapping) framework used in the application is
[Hibernate](https://hibernate.org/), along with [Spring Data
JPA](https://spring.io/projects/spring-data-jpa) to reduce effort of directly
interacting with Hibernate interface such as `Session` and `EntityManager`.

```java
// Entity class
@Entity
@Table(name = "users", uniqueConstraints = {
    @UniqueConstraint(columnNames = "username"),
    @UniqueConstraint(columnNames = "email")
})
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@Builder
public class UserEntity { // User entity from the application source code

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false)
  private String firstName;

  @Column(nullable = false)
  private String lastName;

  @Column(nullable = false)
  private String username;

  @Column(nullable = false)

  // ...more fields
}
```

Using Spring Data JPA it is now easy to generate SQL just by defining a
`UserRepository` that extends a `JpaRepository`.

```java
// Repository interface
@Repository
public interface UserRepository extends JpaRepository<UserEntity, Long> {

  Optional<UserEntity> findByUsername(String username);

  Optional<UserEntity> findByEmail(String email);

}
```

Now the repository can be used like this:

```java
// Service class
@Service
@RequiredArgsConstructor
public class DefaultUserService implements UserService {

  // Inject using constructor injection
  private final UserRepository userRepository;

  private final UserMapper userMapper;

  private final PasswordEncoder passwordEncoder;

  //...methods
}
```
