# Proof of concept: Spring Boot and Vue application

## Executing the project

You need:

- (Required) Java 11
- (Optional) Maven 3

If you have Maven installed, on the root of the project run: 

```txt
mvn clean package
mvn -pl backend spring-boot:run
```

**If you don't have Maven installed**, you can use one of the provided wrappers for your OS. On Windows use mvnw.cmd and on macOS and Linux use mvnw. Example on macOS:

```txt
./mvnw clean package
./mvnw -pl spring-boot:run
```

Then open the project: <http://localhost:8080/>

## Application's behavior

Once the application is up and running, the following is true:

- Going to `localhost:8080` serves Vue's home page.
- Route navigation through page components (e.g. links) works.
- Route navigation by directly typing in browser nav bar works.
- The frontend can talk with the backend without CORS config.
  
Both the frontend and the backend exist in the same [origin](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#definition_of_an_origin).

## Setup overview

The project is a two-module Maven project; one sub-module for the backend, another one for the frontend.

The idea for having both the frontend and backend in the **same origin** boils down to having the backend serve the SPA's (Single Page Application) entry file (`index.html`) and let the SPA deal with everything frontend related (data fetching, state management, routing, etc.), whilst the backend merely forwards non-API requests to the SPA and attends API requests.

## How to make Spring Boot serve the SPA

By default, when the root URL is visited, Spring Boot serves the following file:

```txt
src/main/resources/public/index.html
```

By default, when a Vue application is built, it outputs the following directory:

```
dist
├── css/
├── img/
├── js/
├── favicon.ico
└── index.html
```

The idea is to copy the output of Vue's build into the place where Spring Boot expects to find the `index.html`. This is done with the [`maven-resources-plugin`](https://maven.apache.org/plugins/maven-resources-plugin/) (see the `pom.xml` of the backend sub-module). This plugin does the copying during the `generate-resources` phase, which comes before the `compile` phase. For reference on the Maven phases, see [Lifecycles Reference](https://maven.apache.org/ref/3.6.3/maven-core/lifecycles.html).

Additionally, how can the frontend be built by Maven? Otherwise, the frontend would be required to be built through a npm invocation before Maven can use the output. This is done with the [`frontend-maven-plugin`](https://github.com/eirslett/frontend-maven-plugin) (see the `pom.xml` of the frontend sub-module).

## Heroku configuration

Create the following config vars in the settings page of your Heroku app:

```
BASE_URL=<heroku-url-for-you-app>
MAVEN_CUSTOM_GOALS=clean package
```

This config vars are read as environment vars by the application.

* `BASE_URL` is used by the Spring configuration and the Vue router.
* `MAVEN_CUSTOM_GOALS` tells Heroku how to build the project.

In the root of the project, the [Procfile](./Procfile) defines the command used to execute the project. Notice that entire application can be executing through a JAR and that the `heroku` profile is being used.

Heroku assigns a port to the application to bind to, this is provided by Heroku through the env variable `PORT`. In the Spring configuration this env var is read to configure execution. 

## Other configurations

- **Vue's dev server**. With the setup described above, running the project requires Maven to re-build the frontend and re-compile the backend, which can be lengthy. To accelerate frontend development, a dev server is created at `localhost:8081` after running `npm run serve` in the frontend sub-module. **This does require CORS to talk to the backend**, but frontend reload is much faster. CORS is configured in Spring Boot for the dev server (see class [`WebConfiguration`](backend/src/main/java/hercerm/pocspringvue/configuration/WebConfiguration.java)).
- **Spring Boot's route forwarding**. Spring Boot automatically serves `src/main/resources/public/index.html` when the root URL is visited, however it does not automatically delegate routing to Vue. To allow Vue to handle all the routing, forwarding is required for every route except `/index.html` and `api/*` routes. See the class [`FrontendForwarderController`](backend/src/main/java/hercerm/pocspringvue/configuration/FrontendForwarderController.java).