# Development Automation with Atomist and a Reactive Spring and Kotlin-based Application

by [Josh Long](http://twitter.com/starbuxman)

Spring Framework 5, just released a few weeks ago, is the beginning of a new paradigm for the Spring ecosystem. It's a full throated embrace of the principles of reactive programming. Spring Framework 5 builds on project Reactor, extending the reactive metaphor to a new reactive web runtime and component model called Spring WebFlux. Reactive programming goes hand in hand with functional programming. Spring Framework 5 has a strong functional foundation, requiring a Java 8 baseline and shipping with Kotlin-language integrations for Spring WebFlux, bean configuration, and more.

Spring Framework 5 is only the beginning. it is the foundation for enhancements across the Spring portfolio. The just released Spring Data Kay ships with support for reactive NoSQL stores like Cassandra, Redis, Couchbase and MongoDB. We're nearing the final release of Spring Security 5.0, also built on Spring Framework 5, that supports authentication and authorization in a reactive world. All of this rolls up into [Spring Boot 2.0](http://start.spring.io), due in December, and Spring Cloud Finchley, due early 2018, as an integrated, reactive end-to-end experience.

In the example application, we introduce a Spring Boot 2.0.0 application, generated [from the Spring Initializr](http://start.spring.io). We selected the Kotlin language from the dropdown for languages, and then selected the checkboxes for `Reactive Web`, `Reactive MongoDB`, `Actuator`, `Reactive Security`. The application revolves around a MongoDB document, a `Movie`, which we annotate with the Spring Data annotation `@Document` and `@Id`.

```kotlin
@Document
data class Movie(@Id var id: String? = null, var title: String? = null)
```

We use a convention-based  Spring Data repository - just an interface that Spring Data implements for us - to manage the data management lifecycle of the entity. The interface implies methods for reading, writing and deleting instances of the `Movie` entity. This is a full interface declaration. We're just extending the `ReactiveMongoRepository`. We don't need to provide the redundant `{}` definition as we would in Java.

```kotlin
interface MovieRepository : ReactiveMongoRepository<Movie, String>
```

A simple service bean defines three methods that let us read all records from the database, find a record by its ID, and generate new `MovieEvent` events, gated one per second, in a continuous stream .

```kotlin
@Service
class MovieService(private val mr: MovieRepository) {

    fun all() = mr.findAll()

    fun byId(id: String) = mr.findById(id)

    fun events(id: String) = Flux
            .generate({ sink: SynchronousSink<MovieEvent> -> sink.next(MovieEvent(id, Date())) })
            .delayElements(Duration.ofSeconds(1L))
}
```

We    use the new  service (which Spring injects as a constructor argument and that Kotlin also turns into a JavaBean property) when defining the functional reactive web endpoints. This configuration results in three HTTP endpoints, `/movies`, `/movies/{id}`, and `/movies/{id}/events`. The example uses the Kotlin-language specific `router` DSL.

```kotlin
@Configuration
class WebConfiguration(val ms: MovieService) {

   
    @Bean
    fun routes() = router {
        GET("/movies") { ok().body<Movie>(ms.all()) }
        
        GET("/movies/{id}") { ok().body<Movie>(ms.byId(it.pathVariable("id"))) }
        
        GET("/movies/{id}/events") {
            ok().contentType(MediaType.TEXT_EVENT_STREAM).body<MovieEvent>(ms.events(it.pathVariable("id")))
        }
    }
}
```

This wouldn't be much more than a toy if we didn't address security. We use Spring Security 5.0 to add in HTTP Basic authentication and authorization for two users, `springrod` and `starbuxman`. Naturally, only `springrod` has `ADMIN` authority on this system! :) Try running the application and making a `curl` call to the `/movies` endpoint with `springrod`'s credentials: `curl -uspringrod:pw http://localhost:8080/movies`

```kotlin
@Configuration
@EnableWebFluxSecurity
class SecurityConfiguration {

    @Bean
    fun authentication() =
            MapUserDetailsRepository(
              User.withUsername("springrod").password("pw").roles("ADMIN", "USER").build(),
              User.withUsername("rob_winch").password("pw").roles("ADMIN", "USER").build(),
              User.withUsername("starbuxman").password("pw").roles("USER").build())

    @Bean
    fun authorization(http: HttpSecurity) =
            http
                    .httpBasic()
                    .and()
                    .authorizeExchange().anyExchange().hasRole("ADMIN")
                    .and()
                    .build()
}
```

If we're to have any hope of deploying this code into production, we'll want to support observability, too. The Spring Boot Actuator surfaces information about the service as HTTP endpoints rooted at `/application`. These endpoints include its `/health`, its HTTP `/mappings`, and many more. In order to enable the HTTP endpoints, we have to opt-in with a simple property in the `src/main/resources/application.properties` file: `endpoints.default.web.enabled=true`.

## Running the Project

First install Mongo db if you don't have it already.

Then, start the server with:

```
mvn spring-boot:run
```

Test by visiting `http://localhost:8080/movies`