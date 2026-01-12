+++
title = "Testing in Spring Boot With Junit and Mockito"
date = "2025-10-10"
tags = [
    "java",
    "springboot",
    "testing",
]
+++

In October 2018 and March 2019, Boeing's 737 MAX ✈️ was involved in two fatal crashes resulting in the loss of 346 lives as well as billions in fines, lawsuits and a lasting damage to Boeing's reputation. The [cause of these crashes](https://www.youtube.com/watch?v=H2tuKiiznsY): faulty sensor data caused the MCAS (Maneuvering Characteristics Augmentation System) to activate inappropriately. The software wasn't tested adequately against single-sensor failure.

It's important to note that unit testing alone would not have saved the 737 MAX but had the MCAS system been more rigorously tested, its assumptions may have been questioned with more scrutiny and raised flags earlier.

Now it's fair to say most of us won't be writing software that controls highly sensitive avionics systems but that doesn't mean our code can't cause some chaos when things go wrong (looking at you, production bugs). In the world of Spring Boot, tools like JUnit and Mockito help us write focused tests to mock out dependencies, verify behavior and make sure our code doesn't panic when it sees unexpected input.

## Getting Started

![Spring Beans meme](/images/testing-spring-boot-junit-mockito/meme.jpg)

Today, we'll explore how to add tests to an existing Spring Boot application. Feel free to adapt the examples to fit your own projects. For this walkthrough, I'll be adding tests to a small authentication microservice 🔒 I've been developing.

To get started, we'll add the Spring Boot Starter Test dependency to our project.

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

This starter bundle includes:

- JUnit 5 (Jupiter)
- Mockito
- Spring Test
- AssertJ
- Hamcrest

Here's a breakdown of what each package does individually:

### 1\. JUnit 5 (Jupiter)

This is the most well known testing framework in the Java ecosystem. It provides annotations like _@Test_, _@BeforeEach_, _@AfterEach_ and the core engine to run unit tests. Here we are asserting that 1+1 equals 2:

```java
@Test
void myTest() {
  assertEquals(2, 1 + 1);
}
```

### 2\. Mockito

Mockito is a mocking framework which allows you to mock objects to replace real dependencies.

```java
List<String> mockList = Mockito.mock(List.class);
Mockito.when(mockList.size()).thenReturn(5);
assertEquals(5, mockList.size());
```

Whenever size is called on the list, 5 is returned even though the list is empty. This is called stubbing.

### 3\. Spring Test

Spring Test allows you to test Spring components like controllers and services by starting a Spring application context (all your beans, configurations, etc). Spring Test also includes utilities to mock HTTP requests such as MockMVC.

```java
@Autowired
private MockMvc mockMvc;

@Test
void testController() throws Exception {
  mockMvc.perform(get("/ping"))
    .andExpect(status().isOk())
    .andExpect(content().string("pong"));
}
```

### 4\. AssertJ

Makes assertions more readable and expressive compared to JUnit's built in assertions.

Here's the difference:

```java
List<String> names = List.of("Alice", "Bob");
assertEquals(2, names.size());
assertTrue(names.contains("Alice"));
```

The above JUnit assertions work fine but can get verbose and the order of arguments (expected, actual) sometimes confuses people.

```java
List<String> names = List.of("Alice", "Bob");
assertThat(names)
  .hasSize(2)
  .contains("Alice")
  .doesNotContain("Eve");
```

With AssertJ, the assertions API is more fluent and reads easily like natural language as well as supporting chaining of multiple assertions.

### 5\. Hamcrest

Hamcrest serves a similar purpose to AssertJ making assertions more expressive and readable than the default JUnit ones.

```java
// JUnit
assertEquals(5, result);
// Hamcrest
assertThat(result, is(5));
```

The Hamcrest assertion reads more like a sentence:  "assert that result is 5".

## Writing Tests

Now that we understand the core libraries needed to write tests for our Java applications and their purposes, let’s put them into action. You can find the full code for the authentication microservice I’ll be working with [here](https://github.com/myl117/authservice).

Specifically, we’ll be testing the signup functionality, which is primarily implemented in two files: _SignupService.java_ and _SignupController.java_.

Before writing any tests, we need to establish the folder and file structure. A best practice is to mirror the structure of the main application in the test directory. This means for every class under _src/main/java/..._, there should be a corresponding test class under _src/test/java/..._ in the same package path.

### Example

Main code:

```swift
src/main/java/com/example/app/
├── controller/
│   └── UserController.java
├── service/
│   └── UserService.java
├── repository/
│   └── UserRepository.java
└── model/
    └── User.java
```

Tests (mirroring the structure):

```swift
src/test/java/com/example/app/
├── controller/
│   └── UserControllerTest.java
├── service/
│   └── UserServiceTest.java
├── repository/
│   └── UserRepositoryTest.java
└── integration/
    └── UserFlowIntegrationTest.java
```

Below is the structure of my initial tests folder (before adding more tests):

```swift
src/test/java/com/myl117/authservice/authservice/
├── controller/
│   └── SignupControllerTests.java
└── service/
    └── SignupServiceTests.java
```

With our test files in place, it’s time to start writing tests.

```java
package com.myl117.authservice.authservice.service;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.jdbc.core.JdbcTemplate;

import com.myl117.authservice.authservice.dto.SignupRequest;
import com.myl117.authservice.authservice.exception.UserAlreadyExistsException;

@ExtendWith(MockitoExtension.class)
@DisplayName("SignupService Unit Tests")
public class SignupServiceTests {
  @Mock
  private JdbcTemplate jdbcTemplate;

  @Mock
  private EmailService emailService;

  private JwtService jwtService;

  private SignupService signupService;

   @BeforeEach
    void setUp() {
      jwtService = new JwtService();
      signupService = new SignupService(jdbcTemplate, jwtService, emailService);
    }

  @Test
  @DisplayName("Should successfully signup a new user with hashed password")
  void shouldSignupUserSuccessfully() {
    SignupRequest req = new SignupRequest();
    req.setName("John");
    req.setEmail("john.doe@example.com");
    req.setPassword("totallysecurepassword");

    when(jdbcTemplate.queryForObject(anyString(), eq(Integer.class), any())).thenReturn(0);

    signupService.Signup(req);

    verify(jdbcTemplate).update(anyString(), eq("John"), eq("john.doe@example.com"), anyString(), eq("PENDING_VERIFICATION"));
    verify(emailService).sendVerificationEmail(eq("John"),  eq("john.doe@example.com"), anyString());
  }

  @Test
  @DisplayName("Should throw UserAlreadyExistsException if email is already in use")
  void shouldThrowIfUserAlreadyExists() {
    SignupRequest req = new SignupRequest();
    req.setEmail("exists@example.com");

    when(jdbcTemplate.queryForObject(anyString(), eq(Integer.class), any())).thenReturn(1);

    assertThatThrownBy(() -> signupService.Signup(req))
      .isInstanceOf(UserAlreadyExistsException.class)
      .hasMessage("Email is already in use");
  }
}
```

That was a lot of code. Let’s now walk through what it does.

The first few lines import external testing libraries as well as custom project classes such as SignupRequest and UserAlreadyExistsException.

### Annotations

Throughout our code, you’ll see some annotations that tell JUnit/Mockito how to behave.

**@ExtendWith(MockitoExtension.class)** - Tells JUnit to enable Mockito’s mocking system for this test.

**@DisplayName("...")** - Gives a human-readable name to a test or class when printed in reports.

**@Mock** - Creates a fake version of that object (no real database, no real email sending).

**@BeforeEach** - Runs the _setUp()_ method before every test which is good for setting up fresh test data.

**@Test** - Marks a method as an actual test case.

### Test 1: Successful Signup

```java
@Test
@DisplayName("Should successfully signup a new user with hashed password")
void shouldSignupUserSuccessfully() {
  SignupRequest req = new SignupRequest();
  req.setName("John");
  req.setEmail("john.doe@example.com");
  req.setPassword("totallysecurepassword");
```

Here we make a fake signup request. Then:

```java
when(jdbcTemplate.queryForObject(anyString(), eq(Integer.class), any())).thenReturn(0);
```

This means: _“Pretend that when the database checks if the email exists it finds 0 users”_

```java
signupService.Signup(req);
```

Finally, we verify that the service tried to insert the user into the database and told the email system to send a verification email.

```java
verify(jdbcTemplate).update(
  anyString(),
  eq("John"),
  eq("john.doe@example.com"),
  anyString(),
  eq("PENDING_VERIFICATION")
);

verify(emailService).sendVerificationEmail(
  eq("John"),
  eq("john.doe@example.com"),
  anyString()
);
```

That confirms the signup logic behaves correctly when a new user signs up.

### Test 2: Email Already Exists

```java
@Test
@DisplayName("Should throw UserAlreadyExistsException if email is already in use")
void shouldThrowIfUserAlreadyExists() {
  SignupRequest req = new SignupRequest();
  req.setEmail("exists@example.com");

  when(jdbcTemplate.queryForObject(anyString(), eq(Integer.class), any())).thenReturn(1);
```

This time, we fake the database saying “1 user already exists with that email” and then we expect the signup to fail:

```java
assertThatThrownBy(() -> signupService.Signup(req))
  .isInstanceOf(UserAlreadyExistsException.class)
  .hasMessage("Email is already in use");
```

So if the _Signup()_ method throws the correct exception, the test passes.

## Web Layer Testing

Now that we’ve tested our service and confirmed it works as expected, let’s move on to testing the web layer 🌐 of our signup system, specifically the HTTP aspects: routes, status codes and responses.

We’ll use Spring’s MockMvc, which we discussed earlier, to simulate real HTTP requests without starting an actual server.

### Setting up MockMvc:

```java
MockitoAnnotations.openMocks(this);
signupController = new SignupController(signupService);
mockMvc = MockMvcBuilders.standaloneSetup(signupController).build();
objectMapper = new ObjectMapper();
```

- MockMvc - lets you fake HTTP requests to your controller.
- SignupService - mocked, so no real DB or logic runs.
- ObjectMapper - converts Java objects to JSON (for the request body).

```java
doNothing().when(signupService).Signup(req);
```

Pretend the signup call works fine — no exceptions. Then:

```java
mockMvc.perform(post("/api/auth/signup")
.contentType(MediaType.APPLICATION_JSON)
.content(objectMapper.writeValueAsString(req)))
.andExpect(status().isOk())
.andExpect(content().string("Successfully created user"));
```

This sends a POST request to /api/auth/signup and expects a 200 OK response with the message “Successfully created user”

Finally, we can run our tests with Maven using the following command:

```bash
mvn test
```

This command will compile your project, execute all the tests and display the results in the console. It’s a quick way to verify that your tests pass helping you catch issues early and keep your Spring application reliable as it grows.

That concludes our overview of unit and web layer testing in Spring. While there’s much more to explore, this guide provides a solid foundation. With MockMvc and a few simple mocks, you can confidently ensure your endpoints behave as expected without running the full application.

Thanks for reading!
