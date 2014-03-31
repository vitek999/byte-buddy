What is Byte Buddy?
-------------------

<img src="https://raw.githubusercontent.com/raphw/byte-buddy/gh-pages/images/logo-bg.png" alt="Byte Buddy logo" height="160px" align="right" />

Byte Buddy is a code generation library for creating Java classes during the runtime of a Java application and without
the help of a compiler. Other than the
code generation utilities that [ship with the Java Class Library](http://docs.oracle.com/javase/6/docs/api/java/lang/reflect/Proxy.html),
Byte Buddy allows the creation of arbitrary classes and is not limited to implementing interfaces for the creation of
runtime proxies.

In order to use Byte Buddy, one does not require an understanding of Java byte code or the
[class file format](http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html). In contrast, Byte Buddy’s API aims
for code that is concise and easy to understand for everybody. Nevertheless, Byte Buddy remains fully customizable
down to the possibility of defining custom byte code. Furthermore, the API was designed to be as non-intrusive as
possible and as a result, Byte Buddy does not leave any trace in the classes that were created by it. For this reason,
the generated classes can exist without requiring Byte Buddy on the class path. Because of this feature, Byte Buddy’s
mascot was chosen to be a ghost.

Byte Buddy is written in Java 6 but supports the generation of classes for any Java version. Byte Buddy is a light-weight
library and only depends on the visitor API of the Java byte code parser library [ASM](http://asm.ow2.org/) which does
itself [not require any further dependencies](http://search.maven.org/remotecontent?filepath=org/ow2/asm/asm/4.2/asm-4.2.pom).

At first sight, runtime code generation can appear to be some sort of black magic that should be avoided and only
few developers write applications that explicitly generate code during their runtime. However, this picture changes when
creating libraries that need to interact with arbitrary code and unknown type hierarchies. In this context,
a library implementer must often choose between either requiring a user to implement library-proprietary interfaces
or to generate code at runtime when the user’s type hierarchy becomes first known to the library. Many known libraries
such as for example *Spring* or *Hibernate* choose the latter approach which is popular among their users under the term
of using [*Plain Old Java Objects*](http://en.wikipedia.org/wiki/Plain_Old_Java_Object). As a result, code generation
has become an ubiquitous concept in the Java space. Byte Buddy is an attempt to innovate the runtime creation of Java
types in order to provide a better tool set to those relying on such functionality.

Hello World
-----------

Saying *Hello World* with Byte Buddy is as easy as it can get. Any creation of a Java class starts with an instance
of the `ByteBuddy` class which represents a configuration for creating new types:

```java
Class<?> dynamicType = new ByteBuddy()
  .subclass(Object.class) 
  .method(named("toString")).intercept(FixedValue.value("Hello World!"))
  .make()
  .load(getClass().getClassLoader(), ClassLoadingStrategy.Default.WRAPPER)
  .getLoaded();
assertThat(dynamicType.newInstance().toString(), is("Hello World!"));
```

The default `ByteBuddy` configuration which is used in the above example will create a Java class in the version of
the class file format that is related by the version of the processing Java virtual machine. As hopefully obvious from
the example code, the created type will extend the `Object` class and intercept its `toString` method which should
return a fixed value of `Hello World!`. The method to be intercepted is identified by a method matcher. In the
example, a predefined method matcher `named(String)` is used which identifies a method by its exact name. Byte Buddy
comes with numerous predefined and well-tested method matchers which are collected in the `MethodMatchers` class. The
creation of custom matchers is however as simple as implementing the
([functional](http://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html)) `MethodMatcher` interface.

For implementing the `toString` method, the `FixedValue` class defines a constant return value for the intercepted method.
Defining a constant value is only one example of many method interceptors that ship with Byte Buddy. By implementing the
`Instrumentation` interface, a method could however even be defined by custom byte code.

Finally, the described Java class is created and then loaded into the Java virtual machine. For this purpose, a target
class loader is required as well as a class loading strategy where we choose a wrapper strategy. The latter creates a new
child class loader which wraps the given class loader and only knows about the newly created dynamic type. Eventually, we
can convince ourselves of the result by calling the `toString` method on an instance of the created class and finding
the return value to represent the constant value we expected.

A more complex example
----------------------

Of course, a *Hello World example* is a too simple use case for evaluating the quality of a code generation library.
In reality, a user of such a library wants to perform more complex manipulations such as introducing additional
logic to a compiled Java program. Using Byte Buddy, doing so is however not much harder and the following example
will give a taste of how method calls can be intercepted.

For this demonstration, we will make up a simple pseudo domain where `Account` objects can be used for transferring money
to a given recipient where the latter is represented by a simple string. Furthermore, we want to express that the direct
transfer of money by calling the `transfer` method is somewhat unsafe which is why we annotate the method with `@Unsafe`.

```java
@Retention(RetentionPolicy.RUNTIME)
@interface Unsafe { }

@Retention(RetentionPolicy.RUNTIME)
@interface Secured { }

class Account {
  private int amount = 100;
  @Unsafe
  public String transfer(int amount, String recipient) {
    this.amount -= amount;
    return "transferred $" + amount + " to " + recipient;
  }
}
```

In order to make a transaction safe, we rather want to process it by some `Bank`. For this purpose, the bank
provides an obfuscation but logs the transaction details internally (and to your console). It will then conduct
the transaction for the customer. With the help of Byte Buddy, we will now create a subclass of `Account` that
processes all its transfers by using a `Bank`.

```java
class Bank {
  public static String obfuscate(@Argument(1) String recipient,
                                 @Argument(0) Integer amount,
                                 @Super Account zuper) {
    System.out.println("Transfer " + amount + " to " + recipient);
    return zuper.transfer(amount, recipient.substring(0, 3) + "XXX") + " (obfuscated)";
  }
}
```

Note the annotations on the `Bank`'s obfuscation method's parameters. The first both arguments are annotated
with `@Argument(n)` which will instruct Byte Buddy to inject the `n`-th argument of any intercepted method into
the annotated parameter. Further, note that the order of the parameters of the `Bank`'s obfuscation method
is opposite to the intercepted method in `Account`. Also note how Byte Buddy is capable of auto-boxing the `Integer`
value. The third parameter of the `Bank`'s obfuscation method is annotated with `@Super` which instructs
Byte Buddy to create a proxy that allows to call the non-intercepted (`super`) imlementations of the extended type.

For the given implementation of a `Bank`, we can now create a `BankAccount` using `ByteBuddy`:

```java
Class<? extends Account> dynamicType = new ByteBuddy()
  .subclass(Account.class)
  .name("BankAccount")
  .method(isAnnotatedBy(Unsafe.class)).intercept(MethodDelegation.to(Bank.class))
  .annotateType(new Secured() {
    @Override
    public Class<? extends Annotation> annotationType() {
      return Secured.class;
    }
  })
  .implement(Serializable.class)
  .make()
  .load(getClass().getClassLoader(), ClassLoadingStrategy.Default.WRAPPER)
  .getLoaded();
assertThat(dynamicType.getName(), is("BankAccount"));
assertThat(dynamicType.isAnnotationPresent(Secured.class), is(true));
assertThat(Serializable.class.isAssignableFrom(dynamicType), is(true));
assertThat(dynamicType.newInstance().transfer(26, "123456"),
    is("transferred $26 to 123XXX (obfuscated)"));
```

As obvious from the test results, the dynamically generated `BankAccount` class works as expected. We instructed
Byte Buddy to intercept any method call to methods that are annotated by the `Unsafe` annotation which only concerns
the `Account`'s `transfer` method. We then define to intercept the method call to be delegated by the
`MethodDelegation` instrumentation which will automatically detect the only `static` method of `Bank` as a possible
delegation target. The documentation of the `MethodDelegation` class gives details on how a method delegation is
discovered and how it can be specified in further detail.

Since all transactions will now be processed by a `Bank`, we can mark the new class as `Secured`. We can add this
annotation by simply handing over an instance of the annotation to add to the class. Since Java annotations are
nothing more than interfaces, this is straight-forward, even though it might appear a little strange at first. Finally,
we instruct the `Serializable` interface to be implemented by the created type.

The created type will be written directly in the Java class file format and in Java byte code and never exist as Java
source code or as the source code of another JVM language. However, if you had written a class like the one we just
created with Byte Buddy in Java, you would end up with the following Java source code:

```java
@Secured
class BankAccount extends Account implements Serializable {

  private class Proxy extends Account {

    @Override
    public String transfer(int amount, String recipientAccount) {
      return BankAccount.super.transfer(amount, recipientAccount);
    }

    // Omitted overridable methods of java.lang.Object which are also
    // implemented to call the super method implementations of the outer
    // BankAccount instance.
  }

  @Override
  public String transfer(int amount, String recipientAccount) {
    return Bank.obfuscate(recipientAccount, new Integer(amount), new Proxy());
  }
}
```

You can check out the documentation of the `MethodDelegation` class for more information. There are plenty of other
options for delegation such as delegating to a class or an instance member. And there are other instrumentations that
ship with ByteBuddy and were not yet mentioned. One of them allows the implementation of `StubMethod`s. The
`Exceptional` instrumentation allows to throw exceptions. One can coduct a `SuperMethodCall` or implement a
`FieldAccessor`. Or one can adapt Java Class Library proxies by using an `InvocationHandlerAdapter`. Just as for the
`MethodDelegation`, the Java documentation is a good place to getting started. Give Byte Buddy a try! You will like it.

Where to go from here?
----------------------

Byte Buddy is a comprehensive library that tackles the rather complex matter of code generation. In the two above
examples, we only scratched the surface of Byte Buddy's capabilities. However, for using some of the more low-level
features of this library, a basic understanding of Java byte code is required. A tutorial of this depth is better
suited for another format than a GitHub readme and is currently in work. This is why I am at the moment focusing on
writing a thorough documentation. However, Byte Buddy already comes with a
[detailed in-code documentation](http://bytebuddy.net/javadoc/v0_1/) and extensive test case coverae. If you
feel like using Byte Buddy in your project, feel free to do so even today. When doing so, note the information on
adding a dependency to Byte Buddy to your project below.

Project dependency
---------------------

Byte Buddy is written on top of [ASM](http://asm.ow2.org/), a library for processing compiled Java classes. The
authors of ASM
recommend users of their library to
[repackage the ASM dependency into a different name space](http://asm.ow2.org/doc/faq.html#Q15) when building a project,
in case that the latter is meant to be used within other Java projects. This is done in order to avoid version
conflicts. This is necessary since ASM cannot guarantee the backwards compatibility of its API as a result of
unpredictable future changes in the Java class file format. Since Byte Buddy allows for direct interaction with ASM,
a dependency to Byte Buddy should be repackaged in a similar manner. For this purpose, you can for example use the
[Shade plugin](http://maven.apache.org/plugins/maven-shade-plugin/examples/class-relocation.html) when using Maven.
With Gradle, a similar tool is the [Shadow plugin](https://github.com/johnrengelman/shadow).

License and development
-----------------------

Byte Buddy is licensed under the liberal and business-friendly
[*Apache Licence, Version 2.0*](http://www.apache.org/licenses/LICENSE-2.0.html) and is freely available on this GitHub
page. Byte Buddy will be released on Maven Central once the mentioned tutorial is finished. The project is built using
[Gradle](http://www.gradle.org/). You can use the Gradle wrapper that comes with the project such that you do not need
a Gradle installation of your own. Simply call

```shell
git clone https://github.com/raphw/byte-buddy.git
cd byte-buddy
gradlew build
```

from your shell and Byte Buddy is cloned and built on your machine. Byte Buddy is currently tested for the
[*OpenJDK*](http://openjdk.java.net/) versions 6,7 and 8 using Travic CI. Please use GitHub's
[issue tracker](https://github.com/raphw/byte-buddy/issues) for reporting bugs. When committing code, please
provide test cases that prove the functionality of your features or that demonstrate a bug fix. Furthermore, make sure
you are not breaking any existing test cases. If possible, please take the time to write some documentation. For feature
requests or general feedback, you can also use the issue tracker.

[![Build Status](https://travis-ci.org/raphw/byte-buddy.png)](https://travis-ci.org/raphw/byte-buddy)
