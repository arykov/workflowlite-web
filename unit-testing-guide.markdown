---
layout: page
title: Unit Testing Gude
---

# Unit Testing Guide

This document is a guide to unit testing. It is mostly aimed at
developers using WorkFlow Lite to test new workflows and migrated JPDs
but most of the instructions are equally applicable to any Java code.


## Unit Testing Defined

Unit testing can still be an ambiguous term to some people. For the
purposes of this document, unit testing is defined as the _automated
testing of a single software component_. For Java, "single software
component" typically means a single Java class. By "automated" we mean
the sequence of steps to perform a test is an executable computer
program.

Typically, unit tests are written to be very fast so that they can be
run frequently. This helps detect errors soon after they have been
created, when their cause is fresh in the minds of developers.

Unit tests are usually _white-box tests_, written with detailed
knowledge of the internal implementation of the component.

It's helpful to contrast unit testing with other types of testing.
_Integration testing_ tests the interaction of multiple components
together, while _system testing_ tests the system as deployed to a
particular environment. _Developer integration testing_ ad-hoc testing of an
application by the developer. All of these tests are usually _black-box
tests_, written against functional specifications rather than against
the internal implementation.


## The Importance of Unit Testing

The authors feel that unit testing is crucial to writing reliable
software. Since unit tests are typically the only white-box tests in
use, they are the only way to exercise all paths within the code.
Without unit tests backed up by a code coverage tool, applications would
be deployed to production with code that has literally never been run
before. It could take a very long time before bugs on infrequently taken
code paths are discovered.

Often, even a trivial unit test will discover bugs. A common first test for a
method is to pass it a number of illegal arguments--nulls, zeroes,
negative numbers--and ensure it handles them gracefully. Without unit
tests, this type of non-happy-path handling is often forgotten by developers.

Having developers focus on unit test code coverage also forces them to
tighten up code. It's often easier to simplify a complex boolean
expression than to write unit tests to test its various branches, or to
factor boilerplate code into re-usable methods than to re-test it in
each place it's used.  Developers will be less tempted to add
superfluous functionality "just in case someone needs it some day."

Finally, armed with a comprehensive suite of unit tests, developers are
free to refactor and improve code without the fear of breaking existing
functionality.


## Unit Testing Tools

The rise of Test-Driven Development (TDD), part of the Agile movement,
has produced an explosion in the number and quality of unit test tools:

- _JUnit_ is perhaps the original unit test tool for Java. We continue
  to support JUnit 3 and 4.

- _EasyMock_ is a Java library for creating mock objects. A mock object
  is a pseudo-implementation of some collaborator of the class under
  test. It's often easier to work with mock objects than to create stub
  implementations for testing purposes.

- _Hamcrest_ is a Java library of matcher objects, allowing developers
  to assert conditions while providing helpful, descriptive error
  messages when assertions fail.

- _Cobertura_ is a Java code coverage tool. that automatically produces a code
  coverage report each time the tests are executed.


## Basic Unit Testing

Consider the following class, our _class-under-test_.

    public class Adder {
        public static int add(int x, int y) {
            return x + y;
        }
    }

Let's write some simple tests for this class.  Under JUnit 3, unit tests
are encapsulated in Java classes that extend the JUnit `TestCase` class.
Individual tests are packaged as methods in the class that start with
the word `test`. By convention, the test class is named after the
class-under-test by appending `Test`.

    public class AdderTest extends TestCase {
        public void testAdd() {
            assertEquals(2, Adder.add(1, 1));
        }
    }

At its core, each test invokes some function of the class-under-test and
asserts the results. Here we're using JUnit's `assertEquals` method. Now
suppose our Adder had a bug, and `add(1, 1)` returned 3. The unit test
would fail with the following error:

    junit.framework.AssertionFailedError: expected:<2> but was:<3>

This is quite helpful, as it told us what to expect and what we got
instead. However, we have to be careful with `assertEquals` that we
specify the arguments in the correct order, that being the expected
vaule _then_ the actual value. If we got the order wrong our unit test
will still detect the bug, but the error message would be quite
misleading.


## Setup and Teardown

Sometimes we want to run a series of tests on a class that requires some
non-trivial setup. We could pack that all the tests into a single test
method, doing the initialization at the top of the test, but that's
less than optimal. For example, the first assertion to fail would
terminate the whole test. We'd rather break each test out into its own
method so they can all succeed or fail independently.

JUnit helps out here by providing the methods `setUp` and `tearDown`.
Any initialization code put in `setUp` will be executed before _each_
test method. Similarly, the code in `tearDown` will be run after each
test method.

    public class MyClassTest extends TestCase {

        private MyClass myClass;

        @Override
        protected void setUp() throws Exception {
            myClass = new MyClass();
            // perform other initialization
        }

        public void testFoo() {
            assertEquals(1, myClass.foo());
        }

        public void testBar() {
            assertEquals(2, myClass.bar());
        }
    }

It's important to emphasize that `setUp` is called before _both_
`testFoo` and `testBar`. Your individual test cases should be
independent of each other.


## Improved Error Messages with Hamcrest

One challenge that arises as unit tests get more sophisticated is that
error messages can be difficult to decipher. This is even more important
when the test fails years after it was written and the developer has
long moved on. For example, suppose we wanted to test our `Widget`
class:

    public static class Widget {

        String colour;
        String size;

        public Widget(String colour, String size) {
            this.colour = colour;
            this.size = size;
        }

        public String getColour() {
            return colour;
        }

        public String getSize() {
            return size;
        }
    }

Suppose further that we wanted to test that some method returns a widget
with colour "red" and size "large". With regular JUnit assertions we
might write a test like this:

    public void testWithJUnitAssertions() {
        Widget actual = someMethod();
        assertTrue(actual.getColour().equals("red") && actual.getSize().equals("large"));
    }

While this would detect a widget that didn't match, the resulting error
message would be:

    junit.framework.AssertionFailedError

Upon receiving this error we would have to look at the unit test source
to determine what the error was, and even then would wouldn't know
whether the colour or size, or both, didn't match.

We can write a unit test that returns a much more informative message by
using a small library called Hamcrest. Hamcrest expectations are built
as a structure of `Matcher` objects. Because the matchers are aware of
their structure, they can accurately report exactly where the assertion
failed:

    public void testWithHamcrestMatchers() {
        Widget actual = someMethod();
        assertThat(actual, allOf(hasProperty("colour", equalTo("red")), hasProperty("size", equalTo("large"))));
    }

The functions `assertThat`, `allOf`, `hasProperty`, and `equalTo` are
static methods in the Hamcrest `Matchers` class, statically imported
into our test class. The latter three methods return matchers.
`assertThat` takes an actual value from our test and a matcher to match
it against. If this test fails, the error message is as follows:

    java.lang.AssertionError:
    Expected: (hasProperty("colour", "red") and hasProperty("size", "large"))
         but: hasProperty("size", "large") property "size" was "medium"

The error message tells in detail what was expected, and precisely what
part failed.


### Hamcrest Property Matching

While Hamcrest matchers provide us with better error messages, they can
be a little verbose to write. Suppose we wanted to check all the
properties on a fairly complex Java bean (we'll use Widget, but let's
pretend it's a lot more complicated!). Luckily, Hamcrest has a matcher
for just such an occasion:

    public void testWithHamcrestMatchers2() {
        Widget actual = someMethod();
        Widget expected = new Widget("red", "large");
        assertThat(actual, samePropertyValuesAs(expected));
    }

When this test fails, the resulting error message is as follows:

    java.lang.AssertionError:
    Expected: same property values as Widget [colour: "red", size: "large"]
         but: size was "medium"

Now suppose our Widget class changes to be able to track the timestamp
when it was created, and that this value is set by `someMethod`. We
might start with a unit test that looks like this:

    public void testWithHamcrestMatchers2() {
        Widget actual = someMethod();
        Widget expected = new Widget("red", "large");
        expected.setCreatedTimestamp(new Date());
        assertThat(actual, samePropertyValuesAs(expected));
    }

However, it's likely that the assertion will fail, since the timestamps
for `actual` and `expected` will likely vary by a few milliseconds.
Here, we want a "sloppy" match, say that the created timestamps match
within 500 milliseconds. The WorkFlow Lite library provides an abstract
Hamcrest matcher called the `MultiMatcher`, which is handy for creating
your own composite matchers. In this case, we might create a matcher
specifically for `Widget`s:

    public class WidgetMatcher extends MultiMatcher<Widget> {
        protected WidgetMatcher(Widget expected) {
            super("Widget");
            add(propertyEquals(expected, "colour"));
            add(propertyEquals(expected, "size"));
            add(propertyMatches("createdTimestamp", near(expected.getCreatedTimestamp(), 500)));
        }
    }

Here, `near` returns a Hamcrest matcher that matches if the given date
is within 500 milliseconds of the actual value.

To improve the readability of our unit tests, we often wrap matcher
classes in static methods, either in a utility class called
`HamcrestMatchers` for widely used matchers, or in the unit test itself
for matchers that are not so widely used:

    public Matcher<Widget> matchesWidget(Widget expected) {
        return new WidgetMatcher(expected);
    }

Our improved unit test now looks like this:

    public void testWithHamcrestMatchers2() {
        Widget actual = someMethod();
        Widget expected = new Widget("red", "large");
        expected.setCreatedTimestamp(new Date());
        assertThat(actual, matchesWidget(expected));
    }


## Mocking Collaborators with EasyMock

A non-trivial application will involve a number of objects communicating
with one another. This can present a challenge when trying to isolate
one of the objects for testing.

For example, suppose we have a `WidgetService` that encapsulates the
business logic surrounding widgets. A typical service will need to
persist and query for widgets in a relational database.

    public interface WidgetService {
        public Widget createWidget(String colour, String size);
    }

    public class WidgetServiceImpl implements WidgetService {

        private DataSource dataSource;

        public Widget createWidget(String colour, String size) {
            Widget w = new Widget(colour, size);
            new JdbcTemplate(dataSource).update(...);
            return w;
        }
    }

`WidgetServiceImpl` is difficult to test, since in the unit test
environment there is no database to which the data source can connect. A
common solution is to factor the database access code into a separate
collaborator, `WidgetDao`:

    public interface WidgetDao {
        public void insert(Widget widget);
    }

    public class WidgetDaoImpl implements WidgetDao {

        private DataSource dataSource;

        public void insert(Widget widget) {
            new JdbcTemplate(dataSource).update(...);
        }
    }

`WidgetServiceImpl` now looks like this:

    public class WidgetServiceImpl implements WidgetService {

        private WidgetDao widgetDao;

        public Widget createWidget(String colour, String size) {
            Widget w = new Widget(colour, size);
            widgetDao.insert(w);
            return w;
        }
    }

(`createWidget` looks a little wimpy here. Real-world services typically
have more interesting business logic to test.)

Now we can unit test `WidgetServiceImpl` by creating an alternative
implementation of `WidgetDao` that doesn't depend on the database.
However, creating such a replacement can be difficult, especially if it
has a lot of methods, and especially we want our unit test to assert
which methods were called when, with what data.

The solution is to use _mock objects_. Mock objects are classes
_generated_ from an interface as needed for testing. In this section we
look at using the EasyMock library to create mock objects.

Let's create a unit test for `WidgetServiceImpl`, using the
`WidgetMatcher` we created in the previous section:

    public class WidgetServiceImplTest extends TestCase {

        public void testCreateWidget() {

            WidgetServiceImpl service = new WidgetServiceImpl();

            Widget returned = service.createWidget("red", "large");

            assertThat(returned, matchesWidget(new Widget("red", "large)));
        }
    }

This unit test will fail with a null pointer exception, when the service
attempts to access the DAO. Our solution is to create a _mock_ of the
DAO:

    WidgetDao dao = createMock(WidgetDao.class);
    service.setWidgetDao(dao);

The `createMock` method is a static method on the `EasyMock` class that
we've imported statically into our test class. It creates a special
implementation of the given interface called a _mock object_ or simply a
_mock_. When a mock is first created it is in _record mode_. We can tell
the mock what methods will be invoked when we run the test by simply
calling them in this mode. Once we are done recording the expected, we
use the EasyMock `replay` method to put the mock in _replay mode_, then
we run our tests. Finally we run our test. Here's our modified test:

    public void testCreateWidget() {

        WidgetServiceImpl service = new WidgetServiceImpl();

        WidgetDao dao = createMock(WidgetDao.class);
        service.setWidgetDao(dao);

        // We expect our service to call this
        dao.insert(anyObject(Widget.class));

        replay(dao);

        Widget returned = service.createWidget("red", "large");

        assertThat(returned, matchesWidget(new Widget("red", "large)));

        verify(dao);
    }

Note the strange syntax when used when we set up the expected call to
the DAO's `insert` method. When calling mock methods in record mode, we
pass _argument matchers_ to the method instead of argument methods
directly. `anyObject` is another EasyMock static method that creates a
matcher matching any object of the class `Widget`. If our test resulted
in a call to `insert` with a different type of argument (not exactly
possible here, but let's pretend), the test would fail. EasyMock offers
a number of convenient matchers, and you can write your own. This is
explained in further detail in the [EasyMock
documentation](http://easymock.org/EasyMock3_0_Documentation.html).

Note also the call to `verify` at the end of the test. This confirms
that all of the expected methods were called. If our code-under-test
failed to ever call `insert`, the test would fail here.


### Complex Argument Matching

The test we created above passes. However, it would be nice to verify
that when our class-under-test called `insert` that it called it with
the right data, not just any old Widget object. For simple argument
types like primitives and `String`s, EasyMock provides matchers that can
validate the value passed:

    mock.someMethod(eq(6));

In this case, if the code ended up calling `someMethod` with the
argument `6`, the test would pass. If the test code called `someMethod`
with the argument `5`, the test would fail with a message along the
lines of "you called someMethod(6), but I was expecting a call to
someMethod(5)". This is fine for simple arguments. But what if mock was
being passed an object with dozens of properties?

Our first problem with complex arguments is how to define a match. We
might create `equals` and `hashCode` methods in our complex bean that
take each property value into account, but this is not very satisfying.
For example, the object may have a property with a unique identifier and
an `equals` method built around that. Extending `equals` to include all
other properties may break existing code, and would make `equals` very
complicated and hard to unit test.

It seems like a Hamcrest property matcher might help, and indeed,
Hamcrest provides an adapter to turn a Hamcrest matcher into an EasyMock
matcher. Here we try this approach with our Widget test:

    public static <T> T argThat(Matcher<T> matcher) {
        EasyMock2Adapter.adapt(matcher);
        return null;
    }

    public void testCreateWidget() {

        //...

        Widget expected = new Widget("red", "large");
        dao.insert(argThat(samePropertyValuesAs(expected)));

        //...
    }

Unfortunately, if this test fails, the error message gives us no clue
why:

    java.lang.AssertionError:
      Unexpected method call insert(com.wflite.util.testdoc.Widget@3b8b49):
        insert(is same property values as Widget [colour: "red", size: "large"]): expected: 1, actual: 0

Recall that better error messages was the whole reason to use Hamcrest
in the first place!

To remedy this, we instead use the EasyMock `capture` argument matcher.
The `capture` method matches any argument when it's called, but it
stores the value away so that we can perform further checks on it once
the test has finished:

    public void testCreateWidget() {

        Capture<Widget> actual = new Capture<Widget>();

        WidgetServiceImpl service = new WidgetServiceImpl();

        WidgetDao dao = createMock(WidgetDao.class);
        service.setWidgetDao(dao);

        dao.insert(capture(actual));

        replay(dao);

        Widget returned = service.createWidget("red", "large");

        Widget expected = new Widget("red", "large");

        assertThat(actual.getValue(), samePropertyValuesAs(expected));
        assertThat(returned, samePropertyValuesAs(expected);

        verify(dao);
    }

We've added the variable `actual` of type `Capture<Widget>` that holds the value
passed to `insert` during our test. Our expected call setup has changed
to use the `capture` matcher to populate `actual`. Finally, we assert
`actual` against the expected value. A mismatch will produce an error
like this:

    java.lang.AssertionError:
    Expected: is same property values as Widget [colour: "red", size: "large"]
         but: size was "medium"





