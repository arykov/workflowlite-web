---
layout: page
title: JPD Migration Gude
---

#WLI Migration Guide

This document is a guide for developers needing to migrate WLI workflows
to the new WorkFlow.

WLI represents workflows in files with a `.jpd` extension, which we
refer to as *JPD*s. A JPD is essentially a Java file, with some extra Javadoc
tags that define the process shape. The WorkFlow Lite is able to execute these
processes with minimal modifications. WorkFlow Lite also has support for
reading the process shape directly from the JPD XML.

The basic steps to migrating a JPD are as follows:

- Create a package in the applications `-wli` project to hold the new
  workflow artifacts and copy the JPD file there.

- Extract the XML representing the process shape.

- Rename the JPD with a `.java` extension and edit it to remove WLI
  references and to convert controls and collaborators.

- Create a unit test to test the process.

- Complete Spring configuration


## Initial Steps

The first step in migration is to create a new java module to hold
the migrated workflows. Subsequently an EAR should be created that
includes newly created java module. This application depends on the
following jars: wf-lite.jar, xstream-1.3.1.jar, spring-2.5.6.jar,
commons-logging-1.0.3.jar, wf-lite.jar

Next, we need to copy the `.jpd` files we want to migrate from the
original WLI application into the new package we've created into the
newely created java module.

WorkFlow Lite can parse the XML that WLI uses in its JPDs to define
process shape; however, it can't read the XML directly from the JPD
file. Instead, there is an Ant task to extract the XML
from a JPD file into an appropriately named XML file. Extract the XML
from the JPD from the root of the java module as follows:

   `java -jar wflite-tools.jar -src <location of your original WLI app> -dst <destination for the new application>`

In this case, the XML would be extracted to a file called
`path/to/SomeProcessImpl.xml`. This will match the name of the process
manager we will create shortly.


## Migrating the JPD

We are now in a position to migrate the JPD to a Java class. Start by
renaming the JPD to have a `.java` extension and loading the class in
Eclipse. You will notice a number of compile errors. Our first job is to
delete the `implements com.bea.jpd.ProcessDefinition` portion of the
class definition.  Then in Eclipse do *Source >Organize Imports* (Shift-Ctrl-O).
This should locate the WorkFlow Lite replacements for these controls.

Any remaining compile-time errors will probably be due to other controls
that need to be migrated. Further, you will have to fix up any
collaborators before the converted workflow can even be superficially
unit tested. These are described in more detail in the next section.

### Controls

Controls are Java components that can be re-used across WLI workflows.
Controls may have per-process state, and may also access collaborators
on behalf of their workflow. How a control is migrated depends on what
kind of control it is.

#### Process controls

Process controls, i.e. the interface extends `com.bea.control.ProcessControl`,
are used by parent processes to interact with child processes.
Convert such a control as follows:

- Copy the `.jcx` file to the new workflow package and rename it to
  `.java`.

- Remove the `extends` clause in the interface definition, which
  references WLI classes.

- Add the `@Location` annotation, specifying the fully
  qualified class name of the converted process in its uri.

For example, the original JCX for our hypothetical sub process might
have looked like this:

    /**
    * @jc:location uri="/MyApp/com/myproject/MySubProcess.jpd"
    * @editor-info:link autogen-style="pCtrl" source="Untitled1.jpd" autogen="true"
    */
    public interface MySubProcessControl
            extends com.bea.control.ProcessControl,
            com.bea.control.ControlExtension
    {
      public interface Callback
      {
        /**
         * @common:message-buffer enable="true"
         */
          public void clientResponse ( java.lang.String resultData );
      }

      /**
       * @jc:conversation phase="start"
       */
      public void clientRequest(java.lang.String data);
      /**
       * @jc:conversation phase="continue"
       */
      public void nextStep(java.lang.String moreData);
    }

The converted interface would look like this:

    @Location(uri = "com.myproject.MySubProcess")
    public interface MySubProcessPControl {
        public interface Callback
        {
            /**
             * @common:message-buffer enable="true"
             */
            public void clientResponse(java.lang.String resultData);
        }
        @Conversation(Phase.START)
        public void clientRequest(java.lang.String data);
        @Conversation(Phase.CONTINUE)
        public void nextStep(java.lang.String moreData);
    }

Methods annotated with @Conversation(Phase.START), which is default and can be omitted,
start a process and  have to be the first one to be invoked by
the parent process. This invocation can be followed by zero
or more calls to methods annotated with @Conversation(Phase.CONTINUE)
- continuation methods.

In WorkFlow Lite, continuation methods are handled via the dispatch
service like regular callback events; however, unlike regular callback
events, there is no associated control name.

For example, suppose our hypothetical sub-process provided the method
`nextStep` to allow callers to terminate it prematurely. This would be
declared in the process control JCX as a method with the
`@jc:conversation phase="continue"` Javadoc tag. The JPD would have an
equivalent method called `nextStep`.

#### Subscription controls

If the control is a *subscription control*, the migration involves
changing the Spring context to add a JMS listener. Please refer to the
Subscription Controls section in the [design document](design.html) for
more details.

#### Data Transformation controls

If the control is a `.dtf` control, migrate it as follows. First copy
the `.dtf` file into the new workflow package and rename it to `.java`.
Open the file in Eclipse and change it to a non-abstract class that
extends `WliControl`. Remove the `implements TransformSource` portion.
You must then implement each individual method in the class.

`.dtf` controls generally retrieve some data from an XML document based
on an XPath. In some cases, the XPath is declared in the control itself,
in others the control references a separate file with a `.xq` extension.
The XML itself is represented by object of type `XmlObject`.

To implement the control's methods, you must first change any
`XmlObject` arguments to `String`s. This change may ripple into the
JPD's own fields. Next, you must implement the methods. A convenient way
to do this is to use the `DomQuery` class introduced with WorkFlow Lite.
For example, consider the `FaxOrphanDocumentTransformation` control.
The original implementation is as follows.

    public abstract class FaxOrphanDocumentTransformation implements TransformSource {

        /**
         * Original xpath = "/fax_response/fax/document_id/text()"
         * @dtf:transform xquery="string( $xmlInput/self::fax_response/fax/document_id/text())"
         *
         */
        public abstract String xpathFaxDocumentId(XmlObject xmlInput);

        /**
         * Original xpath = "/fax_response/fax/event_status_code/text()"
         * @dtf:transform xquery="string( $xmlInput/self::fax_response/fax/event_status_code/text())"
         *
         */
        public abstract String xpathFaxEventStatusCode(XmlObject xmlInput);

    }

The migrated control looks like this.

    public class FaxOrphanDocumentTransformation extends WliControl {

        public String xpathFaxDocumentId(String xmlInput) {
            return DomQuery.parse(xmlInput).select("/fax_response/fax/document_id").text();
        }

        public String xpathFaxEventStatusCode(String xmlInput) {
            return DomQuery.parse(xmlInput).select("/fax_response/fax/event_status_code").text();
        }
    }

If the control is of some other type, refer to the [design
document](design.html) for generic instructions on migrating controls.


### Collaborators

Collaborators are stateless components in the application that are invoked
by the workflow. Traditionally, these have been packaged as session EJBs in
application's `-wli` module or in common modules. Since accessing an EJB
requires some clunky boilerplate code to perform a JNDI lookup and
invoke the EJB's "Home" interface, this access is often wrapped with a
facade consisting of a singleton Java object or a utility class with a
number of static methods.

Aside from the added complexity of the facade, this approach also makes
unit testing difficult. Not only must we stub out or mock any
collaborators, but we must set up a fake JNDI registry.

In WorkFlow Lite we have emphasized a more lightweight approach based on
dependency injection (DI), an approach popularized by the Spring framework.
While the EJB approach requires callers to look up their collaborators,
in DI the caller simply declares a field to hold the callee, and some
other component such as the Spring context populates the field as
required. This eliminates the need for facades and makes unit testing
markedly easier: the test case can simply populate the class under test
with its own mocked collaborators.

There are two problems with using collaborators from within a JPD and its
controls. The first is that in general collaborators are *not*
serializable. For example, a component may use a JDBC `DataSource` for
persistence, and `DataSource`s are not serializable. However, code in a
JPD usually needs to access collaborators, and the JPD and its
components need to be serialized. The second problem is that JPD
instances come and go as required by the workflow life cycle, and
therefore cannot be declared in the Spring context. How then do the
collaborator fields in the JPD get populated?

The solutions to both problems are related. We solve the first problem
by declaring collaborator fields in the JPD and any controls as
`transient`. This prevents the framework from attempting to serialize
these controls.

To solve the second problem process manager uses SpringCollaboratorInjector,
that is responsible for populating transient fields. When it encounters a
field that needs to be injected it tries to do it in the following order:

- By type. If only one collaborator of required type is defined in Spring
context it gets injected.

- By bean id specified within Spring @Qualifier(value="bean id") annotation
if such annotation is present

- By deep field name as Spring bean id. Consider the following example:
public class com.xyz.MyProcess{
  private transient Collaborator collab1;
  private Control control1;
}
public class Control{
  private transient Collaborator collab2;
}
Deep field name for collab1 will be com.xyz.MyProcess.collab1. For collab2 deep
field name is com.xyz.MyProcess.control1.collab2


To migrate collaborators, then, the developer must do the following:

- Create an interface mirroring the API provided by the facade or EJB.

- Create a `transient` field in the JPD or control matching of the type
  of the new interface. (No setter is required: the framework will
  populate these fields by reflection.)

### Creating a Unit Test

Since process managers in WorkFlow Lite are regular Spring beans they are
easily tested using standard Java unit testing tools (see our [Unit
Testing Guide](unit-testing-guide.html) for more information).

All process managers require a number of components to be configured:
`ProcessDao`, `DispatchService`, `CorrelationDao`, and `TimerService`.
Rather than configuring these in the test case for each workflow, test
cases can subclass `JpdProcessManagerTestCase`, which configures the
process manager with these collaborators. `JpdProcessManagerTestCase`
provides `DummyDispatchService` and `InMemoryProcessDao` for their
respective collaborators, so your test case need not set up stubs for
these. The other collaborators are EasyMock mocks.

You can create a unit test for your process manager as follows:

- Create a JUnit test case that extends `JpdProcessManagerTestCase`.


- In your `setUp` method, create mocks for any additional collaborators
  required by your process manager, and register them with collaboratorInjector,
  using `registerCollaborator`, that allows to register collaborators by name(s)
  and type. They will be injected by a process manager as needed as described in
  collaboration section.

- In your test case, after setting up expectations with your mocks, call
  the `start` or `startChild` method on your process manager. You then
  need to call `dispatchService.processEvents()` to pump any dispatch
  events through your test. Make any desired assertions.

- If your process is still active, fire events at it by calling
  `dispatchService.sendEvent(CallbackEvent)`. Be sure to call
  `dispatchService.processEvents()` each time, again to pump dispatch
  events through the system.


## Spring Context

Each application will have a context that defines its own components that relies on predefined base configurations of WorkFlow Lite.
Base configuration contains a number of predefined concrete and abstract beans.

#### Naming conventions

In order to avoid name clashes between beans defined in different spring contexts, each application defined bean is prefixed with "&lt;appname&gt;-". For example myapp-FaxCollaborator.
Existing prefix wf refers is used in workflow-context.xml

#### Beans in workflow-context.xml

* transactionManager - instance of spring transaction manager to use in all transactional components

* wf-AsynchDispatcherQueue - local JMS queue that is used for dispatching asynchronous events. Its default JNDI name is wf.DefaultAsynchDispatcherQueue, which will have to be changed using construct similar to this:

    <bean class="org.springframework.beans.factory.config.PropertyOverrideConfigurer">
      <property name="properties">
        <props>
          <prop key="wf-AsynchDispatcherQueue.jndiName">myApp.AsynchDispatcherQueue</prop>
        </props>
      </property>
    </bean>

* wf-ConnectionFactory - JMS connection factory to interact with wf-AsynchDispatcherQueue. JNDI: wf.XAConnectionFactory

* wf-DataSource - XA data source to access internal tables. JNDI: wf.TXDataSource

* wf-CorrelationDao - Correlation Data Access Object

* wf-ProcessDao - DAO to track process state and status, message status, log messages.

* wf-DispatchListener - MDP that listens on wf-AsynchDispatcherQueue.

* wf-DispatchService - preconfigured Dispatch service.

* wf-TimerService - preconfigured Timer Service.

* wf-ProcessManager - preconfigured instance of JpdProcessManager

* wf-AbstractLocalJMSListener - abstract listener for events published on local JMS queues(JMS/MQ Event generator). Connection factory and transaction manager are injected already. To create concrete implementation, JMS destination and concrete instance of wf-AbstractSubscriptionService need to be injected. If waiting for event on non local JMS queue, connection factory needs to be overridden.

* wf-AbstractSubscriptionService - abstract base for "Service" part of subscription control replacements with injected dispatch service and correlation DAO. Subscription control needs to be injected to create concrete implementation.

### Creating application configuration

Objects that form backbone of WorkFlow applications are managed by Spring IoC container. These objects include: process managers, event listeners and queues, auxilary objects used by application. All standard objects and base beans are defined in base configuration. Only things unique to specific application need to be defined in application specific context.

#### Tweaking default configuration

If your application needs to change some environment dependent values in your beans, use [PropertyOverrideConfigurer](http://static.springsource.org/spring/docs/2.5.x/reference/beans.html#beans-factory-overrideconfigurer).
For example see the wf-AsynchDispatcherQueue.

#### Loading spring configuration in EAR

Spring configuration should be loaded using com.wflite.weblogic.SpringApplicationLifecycleListener. This class loads context with the name ear is deployed under and extension ".xml" from the classpath. For example application deployed as appName needs to have appName.xml packaged in it in the root package.
To avoid confusion always use ear name as deployment name. To configure SpringApplicationLifecycleListener, add the following to your weblogic-application.xml

    <listener>
        <listener-class>com.wflite.workflow.process.weblogic.SpringApplicationLifecycleListener
        </listener-class>
    </listener>

For more information refer to [Programming Application Life Cycle Events documentation](http://download.oracle.com/docs/cd/E12840_01/wls/docs103/programming/lifecycle.html).

#### Exposing remote Wrokflow services

In order to expose beans defined in the spring context as remote services, WebLogic SCA is being used.
For more information on WebLogic SCA refer to [WebLogic SCA documentation](http://download.oracle.com/docs/cd/E14571_01/web.1111/e15511/toc.htm).
In a nutshell, the following steps have to be performed for generic WebLogic SCA application:

- The following lines need to be added to weblogic-application.xml:

    <library-ref>
        <library-name>weblogic-sca</library-name>
        <context-root>weblogic-sca-ctx-root1</context-root>
    </library-ref>

- &lt;wls11-home&gt;\common\deployable-libraries\weblogic-sca-1.1.war    needs to be deployed as common library

- META-INF/jsca/spring-context.xml has to be on the application classpath. Suggestion: put spring-context.xml in APP-INF/classes/META-INF/jsca/spring-context.xml

In order to apply above steps to WorkFlow applications:

- Only define WebLogic SCA related configuration in spring-context.xml. All other configuration should be defined in &lt;applicationName&gt;.xml

- Do not include the main context from spring-context.xml, otherwise MDPs will not work

- Use com.wflite.spring.ApplicationContextFactoryBean to reference beans defined in the &lt;applicationName&gt;.xml. Property referenceBeanId should contain beanId of the bean you are interested in bringing into your spring-context.xml

Sample configuration:

    <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:sca="http://xmlns.oracle.com/weblogic/weblogic-sca"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
          http://xmlns.oracle.com/weblogic/weblogic-sca http://xmlns.oracle.com/weblogic/weblogic-sca/1.0/weblogic-sca.xsd">
      <sca:service name="ProcessManagerEJB"
          type="com.wflite.process.ProcessManager" target="processManager">
          <binding.ejb xmlns="http://xmlns.oracle.com/weblogic/weblogic-sca-binding"
              uri="<appname>.processManager" name="processManager_ejb" remote="false" />
      </sca:service>
      <!-- used to avoid double loading of the context as bean factory -->
      <bean name="processManager" class="com.wflite.spring.ApplicationContextFactoryBean">
          <property name="referenceBeanId" value="wf-ProcessManager"/>
      </bean>
    </beans>



### JMS wrapper
To enable WebLogic [JMS wrappers](http://download.oracle.com/docs/cd/E11035_01/wls100/jms/j2ee.html) use in Sping  use com.wflite.weblogic.PooledConnectionFactoryFactoryBean. It currently supports only foreign JMS providers bound into WebLogic JNDI tree. The only property is JndiName that should hold JNDI of the XA connection factory.

