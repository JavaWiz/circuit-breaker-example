### Circuit Breaker
This guide walks us through the process of applying circuit breakers to potentially failing method calls by using the Netflix Hystrix fault tolerance library.

### What We Will Build
We will build a microservice application that uses the circuit breaker pattern to gracefully degrade functionality when a method call fails. Use of the Circuit Breaker pattern can let a microservice continue operating when a related service fails, preventing the failure from cascading and giving the failing service time to recover.

### Set up a Server Microservice Application
The Bookstore service will have a single endpoint. It will be accessible at /recommended and will (for simplicity) return a recommended reading list as a String.

We need to make our main class in BookstoreApplication.java. It should look like the following:

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@SpringBootApplication
public class CircuitBreakerBookstoreApplication {

	@GetMapping("/recommended")
	public String readingList() {
		return "Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)";
	}

	public static void main(String[] args) {
		SpringApplication.run(CircuitBreakerBookstoreApplication.class, args);
	}
}
```

The @RestController annotation indicates that BookstoreApplication is a REST controller class and ensures that any @GetMapping methods in this class behave as though annotated with @ResponseBody. That is, the return values of @GetMapping methods in this class are automatically and appropriately converted from their original types and are written directly to the response body.

We are going to run this application locally alongside an application with a consuming application. As a result, in src/main/resources/application.properties, we need to set server.port so that the Bookstore service cannot conflict with the consuming application when we get that running. The following listing (from bookstore/src/main/resources/application.properties) shows how to do so:

```
server.port=8090
```

### Set up a Client Microservice Application
The reading application will be your consumer (modeling visitors) for the bookstore application. We can view your reading list there at /to-read, and that reading list is retrieved from the bookstore service application.

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@EnableCircuitBreaker
@RestController
@SpringBootApplication
public class CircuitBreakerReadingApplication {

	@Autowired
	private BookService bookService;

	@Bean
	public RestTemplate rest(RestTemplateBuilder builder) {
		return builder.build();
	}

	@GetMapping("/to-read")
	public String toRead() {
		return bookService.readingList();
	}

	public static void main(String[] args) {
		SpringApplication.run(CircuitBreakerReadingApplication.class, args);
	}
}
```
To get the list from your bookstore, we can use Spring’s RestTemplate template class. RestTemplate makes an HTTP GET request to the bookstore service’s URL and returns the result as a String.

We now can access, in a browser, the /to-read endpoint of our reading application and see our reading list. However, since we rely on the bookstore application, if anything happens to it or if the reading application is unable to access Bookstore, we will have no list and our users will get a nasty HTTP 500 error message.

### Apply the Circuit Breaker Pattern
Netflix’s Hystrix library provides an implementation of the circuit breaker pattern. When we apply a circuit breaker to a method, Hystrix watches for failing calls to that method, and, if failures build up to a threshold, Hystrix opens the circuit so that subsequent calls automatically fail. While the circuit is open, Hystrix redirects calls to the method, and they are passed to your specified fallback method.

Spring Cloud Netflix Hystrix looks for any method annotated with the @HystrixCommand annotation and wraps that method in a proxy connected to a circuit breaker so that Hystrix can monitor it. This currently works only in a class marked with @Component or @Service. Therefore, in the circuit-breaker application, we need to add a new class (called BookService).

```
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.net.URI;

@Service
public class BookService {

	private final RestTemplate restTemplate;

	public BookService(RestTemplate rest) {
		this.restTemplate = rest;
	}

	@HystrixCommand(fallbackMethod = "reliable")
	public String readingList() {
		URI uri = URI.create("http://localhost:8090/recommended");

		return this.restTemplate.getForObject(uri, String.class);
	}

	public String reliable() {
		return "Cloud Native Java (O'Reilly)";
	}
}
```

The RestTemplate is injected into the constructor of the BookService when it is created.

We have applied @HystrixCommand to your original readingList() method. We also have a new method here: reliable(). The @HystrixCommand annotation has reliable as its fallbackMethod. If, for some reason, Hystrix opens the circuit on readingList(), we have an excellent (if short) placeholder reading list ready for your users.


In our main class, CircuitBreakerReadingApplication, we need to create a RestTemplate bean, inject the BookService, and call it for our reading list.

Now, to retrieve the list from the Bookstore service, we can call bookService.readingList(). We should also add one last annotation, @EnableCircuitBreaker. That annotation tells Spring Cloud that the reading application uses circuit breakers and to enable their monitoring, opening, and closing (behavior supplied, in our case, by Hystrix).

### Try It
To test your circuit breaker, run both the bookstore service and the reading service and then open a browser to the reading service, at localhost:8080/to-read. We should see the complete recommended reading list, as the following listing shows:

```
Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)
```

Now stop the bookstore application. Your list source is gone, but thanks to Hystrix and Spring Cloud Netflix, we have a reliable abbreviated list to stand in the gap. We should see the following:

```
Cloud Native Java (O'Reilly)
```

### The Hystrix Dashboard
A nice optional feature of Hystrix is the ability to monitor its status on a dashboard.

To enable it, we'll put spring-cloud-starter-hystrix-dashboard and spring-boot-starter-actuator in the pom.xml of our REST Consumer :

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>
		spring-cloud-starter-netflix-hystrix-dashboard
	</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

The former needs to be enabled via annotating a @Configuration with @EnableHystrixDashboard and the latter automatically enables the required metrics within our web-application.

After we've done restarting the application, we'll point our browser at [Hystrix Dashboard](http://localhost:8080/hystrix), Add http://localhost:8080/actuator/hystrix.stream in dashboard to get a meaningful dynamic visual representation of the circuit being monitored by the Hystrix component.

Monitoring a hystrix.stream is something fine, but if you have to watch multiple Hystrix-enabled applications, it will become inconvenient. For this purpose, Spring Cloud provides a tool called Turbine, which can aggregate streams to present in one Hystrix Dashboard.

### Summary
Congratulations! We have just developed a Spring application that uses the circuit breaker pattern to protect against cascading failures and to provide fallback behavior for potentially failing calls.
