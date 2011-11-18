---
layout: page
title: WorkFlow Lite Design
---

#Design

*WorkFlow Lite* is a custom, lightweight approach to managing
long-running business processes without the need for a commercial BPM
tool. It provides the following features:

- a library for defining process shape and stepping through the process,
  based on Petri nets

- a framework for managing the process life cycle, managing persistence,
  error handling, and interaction with external systems

- a framework for supporting existing WLI workflows

This document details the design for each of these components.



## Process Management ##

Most workflows are concerned with coordinating people or external
systems throughout the lifetime of the workflow. To do this, the
workflow sends a request to a person or an external system, then at some
point in the future the person or external system sends a response
signalling the workflow to resume at a certain point. While Petri nets
are good at describing when to send the request and what to do when the
response arrives, we need other components to help with the following
problems:

- When sending a request to an external system, the workflow must
  identify itself so that the external system can correctly route the
  response back to the right workflow instance.

- Once the workflow has nothing to do but wait for responses from
  external systems, its state must be saved to the database.

- The external system needs some mechanism by which to send a response
  to a workflow. This mechanism must locate a workflow instance, load it
  from the database, and resume its execution.

- If an error occurs while processing a response, it should be logged
  for review by system administrators. Some automatic retry mechanism
  should exist to recover from infrastructure errors such as a network
  outage. However, retries should not go on forever; there should be a
  way to freeze a response that has exhausted its allowable retries, and
  to manually re-try frozen responses once the error conditions have
  been addressed.

- There should be sufficient data stored in the database to understand
  the progress of each workflow, and to gather statistics across
  workflow instances.

These concerns are handled by a component called the *process manager*
and a few other components provided by WorkFlow Lite.


### Overview ###

Each workflow is implemented by a Java class called the *process class*,
which encapsulates the process's state and business logic, and by a
corresponding XML file that defines the process's shape.  The lifecycle
of process class instances is managed by a Spring bean called the
*process manager*.

<div class="figure">
  <img src="images/doc/process-overview.png" alt="Process Overview">
  <span class="caption">Process Overview</span>
</div>

To start a workflow, the application calls the process manager's `start`
method (1), passing an instance of the process class. This method
locates the XML for the process based on the process class name and
builds a Petri net from the XML. It then executes the net (2), which in turn
calls methods on the process instance (3).

Typically, the process instance makes one or more requests to external
systems during its initial execution (4). In each case, the workflow
passes a `CallbackInfo` object identifying the process instance itself
and the point at which to resume when the external service responds.
External systems are accessed via a *service adapter*, a local Spring
bean that knows how to interface with the external system, and also how
to deal with callback events and the dispatch service.

The `start` method returns when the net either completes or is waiting
for one or more events to occur.  The `start` method persists the
process instance and Petri net state to the database (5) before
returning.

To respond to a process instance, an external service formulates a
`CallbackEvent` object and calls the `DispatchService.sendEvent` method
(6), which in turn posts the `CallbackEvent` to the dispatch queue (7),
a per-application JMS queue.  The `DispatchHandler` then de-queues the
message (8) and calls the process manager's `resume(CallbackEvent)`
method (9).

The `resume` method loads the process instance from the database (10) and
resumes executing the net at the point indicated by the `CallbackEvent`.
Again, the net begins executing, calling process instance methods.
Finally, when the net is either complete or waiting for more events, the
`resume` method saves the process instance and Petri net state back to
the database (11).  This cycle of calling workflow-aware services and
waiting for their responses repeats as many times as is required.



### Data Model ###

The data model for the process management component is shown below.

<div class="figure">
  <img src="images/doc/data-model.png" alt="Data Model">
  <span class="caption">Data Model</span>
</div>

`ProcessInfo` is a persistent class that stores the process instance,
the state of its Petri net, and associated status information.

`ProcessLogEntry` is a persistent class that forms an audit log of the
process, logging process start and complete events, as well as events
related to callback message handling.

`ProcessMessage` records each callback message sent to the process. Due
to retries and freezing/thawing messages, there may be multiple log
entries associated with each message.

`CallbackInfo` uniquely identifies a particular process instance and
resume point (a.k.a. *control name*). It is used when calling external
services to let them know where response messages should be directed.

`CallbackEvent` encapsulates a response from a workflow-aware service to
a particular workflow.

Finally, `Correlation` is a persistent class that associates `CallbackInfo`
objects with a service-specific correlation identifier. This is useful
for services that can't attach the `CallbackInfo` object directly to
their request. For example, the Fax service tracks faxes by a
numeric fax ID. One can use correlations to build a workflow-aware
wrapper around such a service.


### Component Model ###

The component model for the process management component is shown below.

<div class="figure">
  <img src="images/doc/component-model.png" alt="Component Model">
  <span class="caption">Component Model</span>
</div>

`JPDProcessManagerImpl` is the process manager implementation. It is the
core of the framework, defining the workflow life cycle.

Persistence concerns are delegated to a `ProcessDao`, allowing us to
test process managers without a container.

The `DispatchService` accepts callback events from workflow-aware
services, posting them tot the application's the dispatch queue.

The `DispatchHandler` is a message bean that de-queues messages from
the dispatch queue and forwards them to the process manager.

`TimerService` is a simple, workflow-aware service that implements
timeouts.

Finally, the `CorrelationDao` is used when implementing workflow-aware
wrappers to existing services. It maps service identifiers to
`CallbackInfo` objects required to call back into processes.


### Callback Event Handling ###

The heart of the process management system is its callback handling
mechanism. The process is illustrated below.

<div class="figure">
  <div class="wsd">
    <pre>
participant "Queue" as queue
participant "DispatchHandler" as dispatcher
participant "Process\nManager" as pm
participant "Process\nDAO" as dao
participant "Petri\nNet" as net
participant "Process\nInstance" as instance

queue->dispatcher: onMessage
activate dispatcher
dispatcher->pm: resume
activate pm
pm->dao: findAndLockProcess
alt new message
    pm->dao: insertProcessMessage
else existing message
    pm->dao: updateProcessMessageAndLogNewTx
end
pm->net: fire
activate net
net->instance: call
activate instance
deactivate instance
net->instance: call
activate instance
deactivate instance
net->instance: call
activate instance
deactivate instance
deactivate net
alt normal completion
    pm->dao: updateProcessMessageAndLogNewTx
    pm->dao: updateProcess
else fire method threw exception
    pm->dao: updateProcessMessageAndLogNewTx
    pm->pm: re-throw exception
end
deactivate pm
deactivate dispatcher

    </pre>
  </div>
  <span class="caption">Callback Event Handling</span>
</div>


### Transactions ###

The `DispatchHandler` de-queues messages from the dispatch queue within
a transaction created by Spring. This transaction is propagated to the
process manager, and in turn to any business logic invoked during that
execution of the Petri net. Any exceptions thrown by the business logic
are allowed to propagate back up to the `DispatchHandler`, rolling back
both the business logic and the process state and causing the message to
be re-queued.

Updates to the `ProcessMessage` and `ProcessLogEntry` tables are handled
in new transactions via the `ProcessDao.updateProcessMessageAndLogNewTx`
method, so they are not rolled back even when the message fails.

There is no explicit support for transaction demarcation within the
Petri net structure; however, business logic may invoke components that
acquire and commit a new transaction within the scope of the outer
transaction.


### Concurrency ###

Workflows can be waiting for multiple events at the same time. It's
quite possible that two events could be delivered to the same process
instance within a short time period. Assuming the application is running
on a cluster and the dispatch queue is a distributed queue, these events
could even be delivered simultaneously on different cluster node.

To ensure that this scenario is handled smoothly, the process manager
performs a `select ... for update nowait` on the `ProcessInfo` table, placing a
row-level write lock on the record for the process. Should two messages
arrive simultaneously, the first one to lock the row will proceed, while
the second will fail with an exception. The transaction for the second
message will be rolled back, the message will be re-queued, and will be
re-delivered after the redelivery timeout. Effectively, the message
handling has been *serialized* for a particular workflow instance.


### Logging and Error Handling ###

Since workflows execute in the background, there are no users around to
report errors. We therefore need to capture errors, store them for later
analysis, and allow administrators to re-submit the message when the
underlying problem has been resolved.

It may also be helpful to be able to have the system automatically re-try
messages a few times, to allow the system to recover from transient
infrastructure conditions such as network or database outages.

When an exception occurs during message processing (specifically, when
the Petri net's `fire` method aborts with an exception), the process
manager checks to see if the message's `deliveryAttempts` count is
greater or equal to the process manager's `maxRetries` property. If so,
the process manager freezes the message, else it is retried.

To freeze a message, the process manager updates the message in a new
transaction, setting its status to FROZEN. It then re-throws the
exception, causing the main transaction to roll back, reversing any
process state changes and causing the message to be re-queued. When the
process manager again receives the message, it notes that it is already
in the FROZEN state and silently consumes it without further processing.

To re-try a message, the process manager updates the message in a new
transaction, setting its status to FAILED. Like the freeze case, it
re-throws the exception and the message is re-queued. When the process
manager again receives the message, it tries to re-process it. If it
again fails, its `deliveryAttempts` count is incremented and it is
either frozen or re-tries. However, if it is successfully processed, its
status is updated to PROCESSED.



### Workflow-aware Services ###

In the above sections we've discussed the mechanics of process
callbacks. In this section we describe the conventions for implementing
*workflow-aware services* that can fire these callbacks.

A workflow-aware service is a Spring bean. Typically, this bean has one
or more methods that a workflow can call to request a service. If this
request will result in later callbacks to the process, the method must
take a `CallbackInfo` parameter as its initial argument. For example, a
service for raising EPI tasks might have the following method:

    public interface TaskService {
        public String createAndAssignTask(CallbackInfo ci, String taskType, String role);
    }

The workflow-aware service must document which events it post back to
the workflow, so that the workflow can create the appropriate wait
states. Our convention is that the service publish a Java interface
documenting the callbacks. The callback interface should be defined
within the service interface and should be named `Callback`. The name of
each method in the interface is the name of an event that can be fired,
and the arguments to the method represent the data that is passed with
the event. Each method must return `void`. Returning to our task service
example, the callbacks might be documented via the `TaskServiceCallback`
interface:

    public interface TaskService {

        public String createAndAssignTask(CallbackInfo ci, String taskType, String role);

        public interface Callback {
            public void onComplete(String result);
            public void onAbort();
            public void onTimeout();
        }
    }

When creating wait states in the process corresponding to these events,
it's important to note that there may be separate points at which the
process is waiting for callbacks from the same service. For example, a
workflow may send two faxes to separate parties, and wait for both to
complete before proceeding.

To keep these straight, we borrow the concept of a *control name* from
WLI. When the workflow creates a `CallbackInfo` object to pass to a
service, it must specify a control name that is unique throughout the
process. When the service sends an event to the process, the wait ID is
determined by concatenating the control name to the event name (i.e.
the name of the method in the callback interface) with an underscore.

We can now show an example of a process manager that makes use of this
task service:

    public class MyProcess {

        private TaskService taskService;

        @Override
        private Net createNet() {
            return Net.sequence(
                selfExec("createTask"),
                Net.wait("task_onComplete"),
                selfExec("taskCompleted"));
        }

        private void createTask(Context context) {
            taskService.createAndAssignTask(
                createCallbackInfo(context, "task"), "MyTask", "MyRole");
        }

        private void taskCompleted(Context context, String result) {
            System.out.println("Task completed with result " + result);
        }
    }

We can see in the `createTask` method that the workflow designer has
selected the control name `task` for this callback. Our wait state for
the `onComplete` callback event is therefore named `task_onComplete`.


### Firing Callback Events ###

Now let's look at how the task service would fire callbacks.  First, we
note that the EPI task table has a string column called `wliTask` that
was used to hold the WLI task ID, which is a perfect place to store the
`CallbackInfo` object flattened to a String:


     public String createAndAssignTask(CallbackInfo ci, String taskType, String role) {

        TaskInfoVO task = new TaskInfoVO();
        task.setWliTask(ci.toString());
        //...

        epiService.createAndAssignTask(task, role);

     }

The task service implements a `completeTask` method that is called by
the UI when a task has been completed. Here is where we fire the
callback.

    public void completeTask(TaskInfoVO task, String result) {
        CallbackInfo ci = CallbackInfo.parse(task.getWliTask());
        CallbackEvent event = ci.createEvent("onComplete", new Object[] { result });
        dispatchService.sendEvent(event);
    }

It would be nice to use our callback interface instead of coding
`"onComplete"` as a string. Luckily, `CallbackInfo` has a convenience
method that creates a proxy of our callback interface so we can do just
that:

    public void completeTask(TaskInfoVO task, String result) {
        CallbackInfo ci = CallbackInfo.parse(task.getWliTask());
        ci.createCallbackProxy(dispatchService, TaskService.Callback.class).onComplete(result);
    }



### Correlations ###

With task service we were a little lucky in that we could let EPI track
our callback in the `wliTask` field. In other cases we may not be so
lucky. Consider a workflow-aware fax service that works with the fax
service on the fax gateway. When a fax send is requested, the fax
gateway generates and returns a fax ID. Later, it sends JMS messages to
an application-specific fax queue reporting the status of the fax. Each
of these status messages has the fax ID returned by the initial request.
To forward these notifications to the workflow, we need some way to
obtain the `CallbackInfo` object that was sent from the workflow in the
original request.

To handle this use-case, WorkFlow Lite implements a correlation service
that can correlate application-specific data back to a particular
`CallbackInfo` object.

Suppose the fax gateway sends notifications for success, retry, and
fail fax sending. Our workflow-aware fax service would mirror these
events:

    public interface FaxService {

        public void sendFax(CallbackInfo ci, FaxSendRequest req) {

        public interface Callback {
            public void onSuccess(FaxStatus status);
            public void onRetry(FaxStatus status);
            public void onFail(FaxStatus status);
        }
    }

When the workflow initiates a fax send, the workflow-aware fax service
stores away the fax ID to correlation info mapping using the correlation
dao:

    public class FaxServiceImpl implements FaxService {

        private CorrelationDao correlationDao;

        public void sendFax(CallbackInfo ci, FaxSendRequest req) {
            FaxSendResponse resp = gwFaxService.send(req);
            correlationDao.insert(new Correlation(resp.getFaxId(), ci));
        }
    }

Our workflow-aware fax service doubles as a JMS listener on the
application's fax status queue:

    public void onMessage(Message message) {

        FaxStatus status = FaxStatus.parse(((TextMessage) message).getText());
        CallbackInfo cbi = correlationDao.selectByCorrelationId(status.getFaxId());
        FaxService.Callback callback = cbi.createCallbackProxy(dispatchService, FaxService.Callback.class);

        if (status.getStatus() == FaxStatus.SUCCESS) {
            callback.onSuccess(status);
        } else if (status.getStatus() == FaxStatus.RETRY) {
            callback.onRetry(status);
        } else if (status.getStatus() == FaxStatus.FAIL) {
            callback.onFail(status);
        } else {
            throw new IllegalStateException("Unrecognized status: " + status.getStatus());
        }
    }

How does the correlation table get cleaned up? The `ProcessManagerImpl`
class detects when the process completes and calls the
`CorrelationDao` to delete all the correlation events for that
process ID.


## JPD Support ##

WLI workflows are implemented in files with the extension `.jpd`, which
are actually Java source code files. The Java class defined in the JPD
captures both the state of the business process and the business logic
contained in its steps. The "shape" of the business process is defined
by XML in the Java class's Javadoc block.

JPDs can be re-used mostly as-is by treating the JPD itself as the
process class.  However, there are a few complications to this approach:

- The process manager must parse the JPD XML to produce a Petri net.

- JPD business logic normally accesses collaborators that cannot be
  serialized along with the JPD. We therefore need an approach to
  injecting any collaborators whenever the process state is loaded from
  the JPD.

- WLI implements the concept of re-usable *controls*. These controls may
  contain per-process state, and also have the problem of interacting
  with collaborators. We need an approach to implement the equivalent
  functionality in migrated JPDs.

- WLI supports the idea of sub-processes, with Java interfaces defining
  how the calling process can start the sub-process and how the
  sub-process can call back to their callers.


### Converting the JPD to Java ###

JPD files can be converted to Java by simply renaming them to have a
`.java` extension. However, such a file will have a number of
compilation errors outside of a WLI environment.

The first step to addressing these is to have the JPD class extend the
`JpdBase` class from WorkFlow Lite, rather than implementing the WLI
interface `ProcessDefinition`.

Each JPD has a field of type `JpdContext` that the WLI runtime
initializes, and from which JPD implementations can access the process's
instance ID and information about the last exception that occurred. To
replace this functionality, WorkFlow Lite provides a replacement
`JpdContext` class. `JpdBase` contains a protected field of this class called
`context` that the process manager initializes appropriately. The
`context` field provided in all JPD implementations must be deleted, so
that the JPD code accesses the `context` field in `JpdBase`.

Finally, collaborators and controls used by the JPD must be converted.
These topics are covered separately below.


### JPD Process Manager ###

JPD processes are managed by an instance of the `JpdProcessManager`
class, a subclass of `ProcessManagerImpl`. Aside from being a regular
process manager, `JpdProcessManager` performs the following JPD-specific
tasks:

- It loads the JPD's XML definition of the process shape and creates an
  equivalent Petri net.

- When a JPD instance is first created, it looks for and populates any
  fields that hold WLI control replacements.

- When a JPD instance is first created, and when it is re-loaded from
  the database, it populates any transient fields with its own
  collaborators.

These features are discussed below in their own sections.

Since the process manager is a Spring bean, it must be declared in the
application's Spring context and populated with any required
collaborators.


### Process Shape ###

Each JPD contains a block of XML that defines the process shape.
`JpdProcessManager` is able to parse this XML, but it must first be
extracted to an XML file in the same Java package as the process
manager. Our ant build script defines a target for this:

    ant extract-jpd-xml -Djpd=path/to/MyProcess_20100101.jpd

The above command creates the file `path/to/MyProcess_20100101.xml`.

The `JpdProcessManager.createNet` method looks on the classpath for an
XML file with the same name as the process class, and converts the XML
to the equivalent Petri net via the `JpdXmlReader` class.


### Collaborators ###

Collaborators in JPDs are typically exposed as singleton
Java classes, facade objects containing only static methods, or remote
EJBs. Collaborating in this fashion prevents us from easily writing unit
tests for the workflow. We therefore recommend that all collaborators be
refactored to use a dependency injection style:

- All interaction should be through a Java interface.

- The collaborator should be a field in the JPD, with its type being the
  collaborator's interface.

Further, the JPD field must be marked as `transient` to prevent the
system from attempting to serialize the collaborator when storing the
process state.

Any collaborators required by the JPD must also be declared in the Spring context.
When resuming a process, `JpdProcessManager` looks for any transient
fields in the target JPD and populates them from the Spring context.


### Controls ###

A WLI *control* is a component that a workflow designer can place in a
JPD. The control can encapsulate both logic and per-workflow state, can
expose properties to be set via the WebLogic Workshop UI, and can
perform asynchronous callbacks into the JPD.

There are two different ways to package a control. Controls with a
`.jcs` extension are custom controls that contain logic written in Java.
Controls with a `.jcx` or `.dtf` extension (hereafter called *JCX
controls*) are customizations of standard WLI controls.  JCX controls
are a little weird: they're implemented as interfaces, with
customizations specified in pseudo-attributes in Javadoc blocks.


WorkFlow Lite currently does not have out of the box replacements for
all WLI controls; however, the library of supported controls is constantly
growing and it is easy to create replacement objects that can be substituted
 for WLI controls, allowing JPD code to remain unchanged.

The first step in replacing a WLI control is determining whether a
control is even required. If the WLI control contains no state and makes
no callbacks into the process, it can be treated as a simple,
dependency-injected collaborator as described in the previous section.

If a control replacement is required, we must create a class extending
the `com.wflite.workflow.wli.controls.WliControl` class and
implementing the same method signatures as the control we're replacing.
The class must be serializable as it will be serialized along with the
containing JPD. Any control state can be held in regular fields, while
any collaborators must be held in transient fields. If the collaborators
are workflow-aware services, as they must be if the WLI control makes
callbacks into the process, the control logic can obtain the required
`CallbackInfo` object by calling `getCallbackInfo()` in the `WliControl`
base class.

`JpdProcessManager` has explicit support for control replacements as
follows:

- When the JPD is first instantiated, any null JPD fields whose type extends
  `WliControl` are populated by instantiating a control using the
  control type's default constructor. If a control requires some
  particular state, it must be populated from the JPD's constructor or
  `start` method.

- When the process is started, any JPD fields whose type extends
  `WliControl` have their `callbackInfo` property, defined in
  `WliControl` populated with an appropriate `CallbackInfo` object. This
  `CallbackInfo` has the correct process ID and bean ID, and its control
  name is taken from the field name of the control in the JPD.

- Whenever the JPD is loaded, any transient fields in any of its
  controls are populated using CollaboratorInjector injected into
  JPDProcessManager. Typically it is SpringCollaboratorInjector.


### Process Controls ###

Process controls are used to support interactions between processes and
subprocesses. Each JPD
that may be called as a sub-process has a corresponding JCX control
called a *process control*. Any JPD that calls the sub-process has a
field that contains an instance of the control, through which they can
start the sub-process and receive callbacks, e.g. when the process
completes.  WorkFlow Lite has special support sub-processes, saving
the developer from hand-crafting process controls in each case.

We start with the sub-process, which is converted as a regular JPD.

The JPD for the sub-process will contain an inner interface named
`Callback`, and a field of that type. This field will be populated
automatically by the sub-process's process manager.

The process control JCX for the sub-process must be copied to a
different package and renamed with a `.java` extension.  It then must be
given the annotation `@Location`, specifying the `uri` attribute with
the fully qualified Java class name of converted child process JPD.

When migrating the *calling* JPD, we change any process control fields
to use the newly created control migrated from the JCX.

Now that we've address the mechanics of migration, let's see how
`JpdProcessManager` pulls this all together. When the calling JPD is
instantiated, the process manager looks for any fields of whose type is
annotated with the `@Location` annotation and populates the
field with a dynamic proxy that implements the process control's
interface using the `ProcessControlInvocationHandler` class.

Similarly, when the sub-process JPD is instatiated, its process manager
looks for fields of the type `Callback`, where `Callback` is an inner
interface in the same JPD. If found, it populates the field with a
dynamic proxy that implements the `Callback` interface using the
`CallbackControlInvocationHandler`.

At some point in the calling process's execution it will call the start
method in the process control. The `ProcessControlInvocationHandler`
takes over, creating an instance of the child process class (as
specified by the `@Location` annotation) and calling `startChild` on the
process manager, passing the `CallbackInfo` for the calling process.
This `CallbackInfo` is stored in the sub-process's `ProcessInfo` record
for later use, and the sub-process is started.

When the sub-process wishes to notify the calling process, it calls a
method on its `callback` field. Here, the
`CallbackControlInvocationHandler` takes over, creating a
`CallbackEvent` from the calling process's `CallbackInfo` object and the
method parameters and posting it to the `DispatchService`. The parent
process is notified by the normal callback mechanism and the cycle is
complete.


### Subscription Controls ###

In WLI, a subscription control listens on a
WLI event channel that is most often wired to a JMS queue listener and
routes any messages back into the workflow. Because of their popularity,
WorkFlow Lite has explicit support for subscription controls that listen
to JMS queues.

Migrating a subscription control to WorkFlow Lite happens in two parts:
creating a control class to replace the JCX interface, and deploying a
corresponding message listener in the Spring context.

To convert the subscription JCX, copy the file to a Java package and
rename it with a `.java` extension. Then change it from an interface
into a class extending `SubscriptionControl`. Finally, provide an
annotation specifying the XPath from which to determine the correlation
ID and channel name.

For an example, consider the following fax response control.
`FaxResponseControl.jcx` looks like this:

    /**
     * Defines a new Subscription control.
     *
     * @jc:mb-subscription-control xquery="data($message/fax/id)" channel-name="/FaxResponseChannel"
     */
    public interface FaxResponseControl extends SubscriptionControl,com.bea.control.ControlExtension
    {
        /**
         * @jc:mb-subscription-method filter-value-match="{value}"
         */
        void subscribeWithFilterValue(String value);

        interface Callback extends SubscriptionControl.Callback {
            /**
             * @jc:mb-subscription-callback message-body="{message}"
             */
            void onMessage(com.bea.xml.XmlObject message);
        }
    }

We copy this to `FaxSubscriptionControl.java` in one of our Java
packages:

    @Subscription(xpath="//fax/id", channel="/FaxResponseChannel")
    public class FaxResponseControl extends SubscriptionControl {

        private static final long serialVersionUID = 1L;

    }

Note that the `xpath` and `channel` attributes are derived from the
similarly-named pseudo-attributes in the JCX, and that we've dropped the
inner `Callback` interface (all subscription controls in WorkFlow Lite
use the `SubscriptionCallback` interface defined by WorkFlow Lite).

The corresponding message listener can be configured solely in the
Spring context:

    <jee:jndi-lookup id="faxResponseQueue" jndi-name="FaxResponseQueue"/>

    <bean id="faxResponseService"
          class="com.wflite.workflow.process.SubscriptionServiceImpl">
      <property name="correlationDao" ref="correlationDao" />
      <property name="dispatchService" ref="dispatchService" />
      <property name="subscriptionControl"
                value="com.myapp.FaxResponseControl" />
    </bean>

    <bean id="faxResponseListener"
          class="org.springframework.jms.listener.DefaultMessageListenerContainer">
      <property name="connectionFactory" ref="connectionFactory" />
      <property name="destination" ref="faxResponseQueue" />
      <property name="messageListener" ref="faxResponseService" />
      <property name="transactionManager" ref="transactionManager" />
    </bean>

So how does this all work? A subscription is simply a WorkFlow Lite
*correlation* that maps some identifier (in this case, the ID of a
particular fax request) to a `CallbackInfo` object. When the JPD code
calls the `subscribeWithFilterValue` on our `FaxResponseControl`, it
registers the mapping with the correlation DAO.  When the configured
listener receives a message, it extracts the correlation ID from the
message, looks up the `CallbackInfo` from the correlation DAO, and
invokes a callback to the subscribed process.

For this to work correctly, the control and the listener must agree on
the correlation channel name. We do this by specifying the name in the
subscription control's `@Subscription` attribute, and by providing the
listener with the class name of the subscription control. At runtime,
the listener looks up the annotation on the control class and extracts
the channel name. It gets the XPath required to extract the correlation
ID from any received messages in the same way. While the control itself
does not need this XPath, it's convenient to specify it here as it is
provided by the original JCX control.


<!-- Loads the API for sequence diagrams. Please do not delete. -->
<script type="text/javascript" src="http://www.websequencediagrams.com/service.js">&nbsp;</script>
