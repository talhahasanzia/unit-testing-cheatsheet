
# Unit Testing Guide
## What is Unit Testing?
- In its simplest term, unit testing is testing a small piece of code.
- A unit test usually targets a single method that perform a single task.
- Test ensures that the code is behaving as expected and that code has a deterministic outcome.

## When the Unit Tests are written/run?
There are generally 2 approaches when writing unit tests,
- One is test-driven-development, in which unit tests are written first, these tests specify requirements and at first they should fail because there is no code to produce required output, the code is then written around them to meet requirements and make tests pass.
- Other approach is retrospective testing, that is, after writing code, we write unit tests. This is more commonly used approach in practice (if there are unit tests written at all). We simply write code and then write unit tests after the code and validate code behaviours.
_With both approaches, it is a good practice to run unit tests as often as you can, after every code change, this will result in early identification of flaws in the system._

## Who writes Unit Tests?
- Developers. Period.

## Why should you write Unit Tests?
- Unit tests validate small pieces code for expected outcomes and report early issues.
- Unit tests also make sure that the code is giving errors where expected, this way a lot of corner cases are covered.
- Allows identification of logical bugs in the code when they are covering edge cases.
- If unit tests are written regularly, they make code resist to changes that breaks the code. If a change in the code fails the unit test, either the code has issue or test is not updated, both ways, its a warning sign for developer to put an extra effort to investigate the issue before closing the development cycle.
- It seems waste of time and extra effort at the beginning, and it is, but few days effort of maintaining the code with meaningful unit tests saves months of refactoring later.
- It makes code cleaner - since the tests require certain level of abstraction and loose coupling which make code more cleaner.

## Writing Unit Tests
In the following discussion I will be using Java as the example language and some references from Android Framework. Regardless of this fact, the general guidelines and good practices will apply to any OOP based framework.

### Intention
- The purpose of this writing is to promote the unit tests in software projects, specially those which are shaping the way we live, these projects can be related to finance, medicine, transport or online shopping.
- The bugs or problems in these softwares causes both customers and developers to lose their time and money.
- Promoting unit tests can save a lot of time in covering basic business logic validations and resulting a reliable software which can be fruitful for all the entities in the business.
- Writing unit test for the sake of unit tests or say achieving high code coverage is not a good idea. If the tests are not playing their part in verifying business flows, they are of no value.
- In programming, every line of code generates a business value, no one likes to lose business, unit tests verify that the business value that code aims to provide is on the spot.

### Writing Testable Code - A prerequisite
- Unit tests are written in controlled environment, this helps us provide certain conditions under which we expect certain outcomes. If we cannot control the agents or entities that are driving these conditions we cannot be sure about the outputs. This test environment is largely controlled by dependencies we provide to the test case.
- First and foremost thing is separating the dependencies from the code. For this you should be able to recognize what are the dependencies and what is the actual code that needs to be tested.
- In terms of Java and Android, if you are working in Android, your business layer should be testable without any Android dependencies.
- Continuing on Android as example, for the business layer (and unit tests themselves), most of the Android framework stuff will be dependencies for code. Like network library, Retrofit.

_Instead of:_
```
    public UserService(User user, Retrofit retrofit)
```
_Use some abstraction:_
```
    public UserService(User user, Request userRequest) 
```
_Using an abstraction layer of Request class to wrap retrofit in it so the code doesnt depend on Android Retrofit_

- Creation of objects should be delegated to upper level, like dependency manager. If a class that needs to be tested is creating its own objects inside, we cannot control data in those objects to test.

_Instead of using "new" keywords in code:_
```
    public UserService(User user){
      request = new Request();
      ...
    }
```
_Use dependency injection:_
```
    public UserService(User user, Request userRequest) 
```
_We have now control over creation. We can also use dependency managers like Dagger2 and "inject" this dependency here_


- Another important thing to mention here is _composition over inheritance_ will always be in favour of testable code. Since composition gives us more control over objects when we are using dependency injection, we can easily control these objects in test environment. In above example `UserService`, we can provide a `UserValidator` instead of implementing it. Just like `UserRepo` and `Request`, it can follow composition. But the abstraction is still maintained since no concrete references are used here.

_Instead of:_
```
public class UserService implements UserValidator{
...
}
```
_We can do:_
```
 // constructor injection
 public UserService(User user, Request userRequest, UserValidator validator) 
```
- Separate creation logic from business logic, thats what dependency managers do, if this is not the case, we might miss some business logic to test which was ran during creation of objects. Since tests and business classes should not be responsible for creating objects involved in a business logic flow.

_Avoid writing business logic in constructor or anywhere in creation classes like dependency managers:_
```
    public UserService(User user, Request userRequest){
      if(userRequest !=null){
        this.userRequest= userRequest;   // This block is never tested since it is just creation block
      }
    }
```
- Unit tests usually targets business layer, so regardless of any framework you are, your code that needs to be tested must be using the core language libraries. A good way to realize this in Java is to look at `imports` section. If there is an import that is not coming from core Java it should raise alarm.
```
import android.os.Bundle; // <--- This is not good in business layer
import android.view.View; // <--- This is not good in business layer
import com.selfsol.app.models.user; // <--- This is ok as it is a POJO model
```
_First priority should be to avoid such dependencies, if not possible their creation should be delegated to dependency managers with proper abstractions. So when this class is tested, these dependencies can easily be mocked with dummy values or behavior and unit tests are not blocked._
- Its nice to have to follow an architectural pattern to manage code, whether it be MVVP, MVP, VIPER or any other, primary goal is to make code clean, maintainable and testable, which makes it helpful to work with such codebase in the longer run.
- The implementation of architecture doesnt really provide all the solutions, so there are still few things that needs to be decoupled, like networking and UI stuff. 

Following example illustrates how this abstraction can be achieved and how it is created in dependency manager layer:

```
// class for abstraction of network library
public interface Request<T>{
    void execute( T data);
}

// class having real network classes and implementation
public class UserRequest<User> implements Request{
    ...
}

// creation of this class
public class SomeDependencyProvider(){
    // notice interface reference in return type
    public Request<User> createRequestObject(){
        return new UserRequest();
    }
}
```

_Generally, we dont need network or UI when writing unit tests, so if there is a good abstraction layer between the business logic layers and UI or Network etc, it will help us mock these entities and validate the actual business logic in tests_

- Static methods makes it hard to test unless they are pure methods and donot use or create any dependency objects in their execution, moreover these static methods or objects cannot be mocked easily which are blocker in unit testing.

_Consider following class in which one method is acceptable but the other one creates trouble:_

```
public class Utils{
    // this wont be any trouble
    public static boolean isEmptyString( String string){
        return string !=null && !string.isEmpty();
    }
    // this will cause issue since this is using an object that is in its static scope and cannot be mocked
    public static Date formatDate(String pattern, long timeInMillis){
        DateFormatter formatter = new DateFormatter( pattern );
        return formatter.format( timeInMillis );
    }
    
}
```
_In other words, static classes cant be used with dependency injections so they lack this flexibility of being separating creation logic from actual business logic. This brings us to the point number 1, where we are unable to control the test environment._


- Tests are written for public methods. Because other classes or layers call public methods of classes that provide business logics. These methods in turn, call private methods which itself covers these private methods. DONT make private methods public for just writing TESTS.

```
public class UserService{

    public boolean validateUser(User user){
        validateAge(age);
    }
    
    private boolean validateAge(int age){
    }
}
```
_In above example, we will write test for validateUser(), which will also cover validateAge(), if we do want to test such methods individually we can introduce an interface that contains these signatures, in this way these methods will hide their implementation details and yet they will be accessible for testing from outside or mocking if required._ 

- When writing a test for a class, the target class should be instantiated as a concrete implementation, so the real code is ran, and all other participating objects are mocked. If the participating objects have abstraction (they are interface objects), mocking becomes  easy.

_Consider writing test for UserService:_
```
@RunWith(MockitoJUnitRunner.class)
public class UserServiceTest {
    // Class reference
    private UserService userService;
    
    // interface reference mocked
    @Mock
    Request request;

    // interface reference mocked
    @Mock
    UserRepo userRepo;
...
}
```
_We can later define how "request" behaves when its methods are called, same for "userRepo". Point is, this way, by making them abstract, decoupled and using dependency injection pattern, it is easier to control these participating objects and we can design our test by providing values and behaviours the way we like to._


### Writing Unit Tests
- We will use Java and its testing framework JUnit in these examples, moreover, we will refer Mockito for mocking objects. 
- We will focus on UserService (or UserManager) example, in which, we will validate creation of valid users in a system.
- We will not test classes that serves as dependencies in the UserService, these are UI classes referred as "View" in most of the architectures and Network classes like Request or Response. We will mock these dependencies.
- It is not our responsibility to test 3rd Party library classes like Retrofit, Gson, Picasso, OkHttp etc.
- Meaningful names of test methods should be used, these names can be longer as long as they are self explanatory.
- AAA Testing Pattern - Arrange, Act, Assert is the pattern we will use and it goes as follows:
  - Arrange: Setup objects, parameters and environment for the test
  - Act: Perform action, the actual code is ran on which test is to be done
  - Assert: Assertion or validation is made on the action that was performed if it is giving required results or not
  
  
See [`UserServiceTest`](https://github.com/talhahasanzia/unit-testing-cheatsheet/blob/master/UserServiceTest.java) for details
 
 ### Arrange
 _Lets declare objects first:_
 ```
    private UserService userService;

    @Mock
    Request request;

    @Mock
    UserRepo userRepo;

 ```
 _How mocks work?_
 ```
    // MockitoJunitRunner initializes mocks itself when tests starts
    @RunWith(MockitoJUnitRunner.class)
    public class UserServiceTest {
```
_And finally setting up the Test Class:_
```
userService = new UserService(request, userRepo);
```

The setup seems easy and simple because we already followed the practices in the code that are test friendly, here using injection pattern instead of creating dependencies in the object itself. We were able to provide `UserRepo` and `Request` from outside based on our requirement here.

### Act
_Now we perform our test:_
```
    @Test
    public void testValidAge_inputPositiveNumber_shouldValidateSuccess() {
        boolean result = userService.validateAge(18);
        ...
    }
```
This is how we right test in JUnit. Notice that the function name is like a sentence that tells the whole story. In source code, this convention may not be liked by many, but in tests, this is preferred. It explains a lot about the test. Since tests dont have that amount of documentation that code has (if any), this is important. The convention goes like this: `testMethod_conditions_outcomes()`

Then the _act_ happens. The method is called and the value is captured. 

### Assert

```
    assertTrue("Expected to validate positive number as valid age but failed", result);
```
The value captured in the _act_ is _asserted_ to be true or false. The `assertTrue` is a JUnit assertion method which checks the given condition alongwith option to provide a message in case of failure. We are not discussing implementation at this point just the approach. All the implementations are left blank.

There is a message _Expected to validate positive number as valid age but failed_ passed as 1st parameter. This is failure message. It is important to flag failures with meaningful messages. So when someone come across a failed test case in the log, s/he can identify what is the problem. _This is also a good practice._


In most of the cases, this assert flow works in tests. But sometimes the methods dont return values. Like in VIPER which uses callback pattern. In this case `view` is holding reference to `presenter` and a `view` reference is inside a `presenter`. `View` calls `presenter.fetchSomething()` this method does not return anything but after it fetches, it calls `view.onSuccessfulFetch()`. How can we test presenter since its methods dont return anything?

Similarly in case of `Request` when the `UserService` calls `createUser(User user)`, it doesnt do anything. It calls `request.execute(user, this)` and return nothing.

There are two things we can do to test the flows or method that dont return anything:
- If the method calls another method on any other object, this method call can be `verified` by Mockito framework. This is called testing interactions where we cannot test states (asserting values).
- Check if the code runs without any exception. This is not the only thing you should be doing, but in rare cases like these, this is important factor.


