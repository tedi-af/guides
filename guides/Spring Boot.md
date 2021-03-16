# Spring Boot

## Installation
- Download and install JDK at [adoptopenjdk.net](https://adoptopenjdk.net)
- Go to [Spring Initializr](https://start.spring.io)
- Click ADD DEPENDENCIES and choose Spring Web
- Change Project to Gradle Project
- Change Languange to Kotlin
- Click GENERATE
- Extract your downloaded Spring project

## Getting Started
1. Edit src/main/kotlin/com/example/demo/DemoApplication.kt
    ```
    ...

    @SpringBootApplication
    @RestController
    class DemoApplication {
        @GetMapping("/hello")
        fun hello(@RequestParam(value = "name", defaultValue = "World") name: String) = "Hello $name!"
    }
    
    ...
    ```
2. Execute `./gradlew bootRun` on your Terminal
3. Open your browser and navigate to **localhost:8080/hello** or **localhost:8080/hello?name=your-name**

## Proper Application
1. Create src/main/kotlin/com/example/demo/Greeting.kt
    ```
    ...

    class Greeting(
        val id: Long,
        val content: String
    )
    ```
2. Create src/main/kotlin/com/example/demo/GreetingController.kt
    ```
    ...

    @RestController
    class GreetingController {
        val counter = AtomicLong()

        @GetMapping("/greeting")
        fun greeting(@RequestParam(value = "name", defaultValue = "World") name: String) =
            Greeting(counter.incrementAndGet(), "Hello $name!")
    }
    ```
3. Open your browser and navigate to **localhost:8080/greeting** or **localhost:8080/greeting?name=your-name**

> Now your application returning data as JSON instead of plain text and separate your code from application entry point (DemoApplication). You can also build your application as executable JAR with `./gradlew build` and then execute `java -jar your-app.jar`

## Using Database
1. Prepare your application with these dependencies :
    - Spring Web
    - Spring Data JDBC
    - H2 Database
2. Create src/main/kotlin/com/example/demo/Message.kt
    ```
    ...

    @Table("MESSAGES")
    data class Message(
        @Id val id: String?,
        val text: String
    )
    ```
3. Create src/main/kotlin/com/example/demo/MessageRepository.kt
    ```
    ...

    interface MessageRepository : CrudRepository<Message, String> {
        @Query("select * from messages")
        fun findMessages(): List<Message>
    }
    ```
4. Create src/main/kotlin/com/example/demo/MessageService.kt
    ```
    ...

    @Service
    class MessageService {
        @Autowired
        lateinit var repository: MessageRepository

        fun findMessages() = repository.findMessages()

        fun post(message: Message) = repository.save(message)
    }
    ```
5. Create src/main/kotlin/com/example/demo/MessageController.kt
    ```
    ...

    @RestController
    class MessageController {
        @Autowired
        lateinit var service: MessageService

        @GetMapping
        fun index() = service.findMessages()

        @PostMapping
        fun post(@RequestBody message: Message) = service.post(message)
    }
    ```
6. Edit src/main/resources/application.properties
    ```
    spring.datasource.schema=classpath:sql/schema.sql
    spring.datasource.url=jdbc:h2:file:./data/mydb
    spring.datasource.initialization-mode=always
    ```
7. Create src/main/resources/sql/schema.sql
    ```
    create table if not exists messages (
        id varchar(36) default uuid() primary key,
        text varchar(100) not null
    );
    ```
8. Create requests.http
    ```
    POST http://localhost:8080
    Content-Type: application/json
    {
        "text": "Hello World"
    }

    GET http://localhost:8080
    ```
9. Run your application and try to use GET or POST request

## Building RESTful Web Service with MySQL
1. Prepare your application with these dependencies :
    - Spring Web
    - Spring Data JPA
    - MySQL Driver
    - Flyway Migration
2. Create src/main/kotlin/com/example/demo/User.kt
    ```
    ...

    @Entity
    @Table(name = "users")
    data class User(
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        var id: Int? = null,
        var name: String,
        var email: String
    )
    ```
3. Create src/main/kotlin/com/example/demo/UserRepository.kt
    ```
    ...

    interface UserRepository : JpaRepository<User, Int>
    ```
4. Create src/main/kotlin/com/example/demo/UserController.kt
    ```
    ...

    @RestController
    @RequestMapping("/users")
    class UserController {
        @Autowired
        lateinit var repository: UserRepository

        @GetMapping
        fun all(): List<User> = repository.findAll()

        @PostMapping
        fun create(@RequestParam name: String, @RequestParam email: String) =
            repository.save(User(name = name, email = email))

        @GetMapping("/{id}")
        fun find(@PathVariable id: Int): User =
            repository.findById(id).orElseThrow { UserNotFoundException(id) }

        @PutMapping("/{id}")
        fun update(@PathVariable id: Int, @RequestParam name: String, @RequestParam email: String): User  =
            repository.findById(id)
                .map {
                    it.name = name
                    it.email = email
                    repository.save(it)
                }
                .orElseGet { repository.save(User(id, name, email)) }

        @DeleteMapping("/{id}")
        fun delete(@PathVariable id: Int): String {
            repository.findById(id)
                .map {
                    repository.delete(it)
                }
                .orElseThrow { UserNotFoundException(id) }

            return "User id $id was deleted"
        }
    }
    ```
5. Create src/main/kotlin/com/example/demo/UserNotFoundException.kt
    ```
    ...

    class UserNotFoundException(id: Int) : RuntimeException("User id $id is not found")
    ```
6. Create src/main/kotlin/com/example/demo/UserNotFoundAdvice.kt
    ```
    ...

    @RestControllerAdvice
    class UserNotFoundAdvice {
        @ExceptionHandler(UserNotFoundException::class)
        @ResponseStatus(HttpStatus.NOT_FOUND)
        fun userNotFoundHandler(exception: UserNotFoundException) = exception.message
    }
    ```
7. Edit src/main/resources/application.properties
    ```
    spring.jpa.hibernate.ddl-auto=update
    spring.datasource.url=jdbc:mysql://localhost:3306/demo
    spring.datasource.username=user
    spring.datasource.password=password
    ```
8. Create src/main/resources/db/migration/V...__Table_Users.sql
    ```
    create table if not exists users (
        id int auto_increment primary key,
        name varchar(50) not null,
        email varchar(100) not null unique
    );
    ```
9. Create src/main/resources/db/migration/V...__Data_Users.sql
    ```
    insert into users (name, email) values ('test1', 'test1@example.com');
    insert into users (name, email) values ('test2', 'test2@example.com');
    insert into users (name, email) values ('test3', 'test3@example.com');
    ```
10. Edit build.gradle.kts
    ```
    ...

    plugins {
        ...
        id("org.flywaydb.flyway") version "7.7.0"
    }

    flyway {
        url = "jdbc:mysql://localhost:3306/demo
        user = "user"
        password = "password"
    }

    ...
    ```
11. Create requests.http
    ```
    DELETE http://localhost:8080/users/4
    
    PUT http://localhost:8080/users/1
    Content-Type: application/x-www-form-urlencoded
    name=test1update&email=test1@example.com
    
    GET http://localhost:8080/users/1
    
    POST http://localhost:8080/users
    Content-Type: application/x-www-form-urlencoded
    name=test4&email=test4@example.com
    
    GET http://localhost:8080/users
    ```
12. Run your application and try to use GET, POST, PUT or DELETE request

## Using Docker
1. Download and install these files from [download.docker.com](https://download.docker.com/linux/ubuntu/dists/focal/pool/stable/amd64)
    - containerd
    - docker-ce-cli
    - docker-ce
2. Create Dockerfile in your project
    ```
    FROM openjdk:latest
    COPY build/libs/demo-0.0.1-SNAPSHOT.jar app.jar
    ENTRYPOINT ["java", "-jar", "app.jar"]
    ```
3. Create docker-compose.yml
    ```
    services:
      app:
        container_name: demo
        build:
          context: .
        ports:
          - 8080:8080
    ```
    > You need docker-compose command to build your application. Follow these installation instructions [here](https://docs.docker.com/compose/install)
4. Create ~/demo.sh
    ```
    #!/bin/bash
    cd ~/demo
    git pull
    ./gradlew build
    sudo docker-compose up -d --build --force-recreate
    ```
5. Execute `./demo.sh` on your Terminal