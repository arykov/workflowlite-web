---
layout: page
title: WorkFlow Lite Overview
---

# Overview

WorkFlow Lite is a lightweight library for dealing with long-running
business processes, a.k.a. workflows. This document gives an overview
of the WorkFlow Lite approach.

## Whither Workflow Tools?

Why do we need a tool to help us with workflow management? The short
answer is that workflows may last minutes, hours, days, or longer, and
that typical programming languages are not well-suited to handle
processes of such lengths.

Consider a very simple workflow:

- send a fax to someone

- wait for the fax server to confirm the fax has been sent

- send an email to an employee to notify her that the fax has been sent

We might try solving this in Java with a simple polling loop:

    int faxId = faxService.sendRequest(...);
    while (true) {
        Thread.sleep("60000");
        if (faxService.isFaxSent(faxId)) {
            break;
        }
    }
    emailService.sendEmail(...);

This has a couple of serious problems. First, this code occupies the
thread for the entire time it takes for the fax to be sent, which could
take minutes. During that time, the thread is unavailable to perform
other work on the server. If there were hundreds of faxes outstanding at
the same time, the application server would quickly grind to a halt.

The second serious problem is that the process state is kept in the
memory of the application server. What would happen if that server
suddenly stopped working? All of those busy loops waiting for faxes
would go away, and no one waiting for them to be sent would ever be
notified.

So the slightly longer answer to why we need a workflow tool is that we
need a way to implement long-running processes that are able to survive
system restarts, and that don't consume valuable system resources such
as threads, memory, and database locks while they are waiting for events
to occur.

Another, more subtle reason is that people tend to think of a business
process instance as a thing in and of itself. It's convenient to think
in terms of the status of the process, of how far a process has
progressed and for which events it's waiting. It's handy to be able to
determine whether a process is "stuck", and to be able to restart it.

## An Action-Reaction Approach

A simple solution to managing long-running processes is as follows:

- Bundle up the process state into a persistent object and store it in a
  database.

- When an event such as the receipt of a fax confirmation occurs, load
  the state from the database, perform the next set of actions dictated
  by the workflow, and re-save the state.

This addresses our immediate concerns. Assuming we use something like a
JMS queue for receiving external events we don't tie up a thread
waiting for the response. And since our state is persisted, we can
survive a restart of the system, or a switch to a different server in
the cluster.

The problem with this approach is that it tends to break down for all
but the simplest of workflows.

Consider the following workflow:

<div class="petriNet">
  <img src="images/doc/overview-workflow.png" alt="Example Workflow">
</div>

In this process we wait for the fax to be sent, but if it seems to be
taking too long we give up waiting. This timeout pattern is a common
one. Let's start by encapsulating the process's state and business logic into a class
called `MyProcess`, and implementing a DAO to manage its persistence:

    public class MyProcess {
	    // BEGIN DECLARING STATEFUL PART
        private int id;
        // other fields as required
  		// END DECLARING STATEFUL PART
	   	// BEGIN DECLARING INJECTABLE COLLABORATORS
		private transient FaxService faxService;
        private transient TimerService timerService;
        private transient MyProcessDao dao;
		// END DECLARING INJECTABLE COLLABORATORS
		
		private Logger logger = ...;

		// BEGIN BUSINESS METHODS
        public void logFaxComplete() {
            logger.log("Fax process " + id + " complete);
        }

        public void logFaxSent() {
            logger.log("Sent fax for process " + id);
        }

        public void logFaxTimeout() {
            logger.log("Fax timed out for process " + id);
        }

        public void onFaxSent() {
            MyProcess process = dao.load(id);
            logFaxSent();
            logFaxComplete();
            process.setStatus(MyProcess.STATUS_COMPLETE);
            dao.save(process);
        }

        public void onFaxTimeout() {
            logFaxTimeout();
            logFaxComplete();
        }

        public void sendFax() {
            faxService.sendFax(/* fax info */, id);
        }

        public void startTimer() {
            timerService.setTimer(60000, id);
        }

		//THIS METHOD STARTS the PROCESS
        public void start() {
            sendFax();
            startTimer();
        }
		// END BUSINESS METHODS

    }

    public interface MyProcessDao {
	    public MyProcess create();
        public MyProcess load(int id);
        public void save(MyProcess process);
    }


We assume that both the fax service and the timer service use JMS
queues to report back their events. We therefore need a pair of message
listeners to handle the events and notify the process manager. Note that
we're assuming that both services can store our process ID and report it
back when an interesting event occurs:

    public class FaxListener implements MessageListener {

	    private MyProcessDao dao;

        public void onMessage(Message message) {
            int processId = /* get process ID from message */
			MyProcess myProcess = dao.load(processId);
            myProcess.onFaxSent();
			dao.save(myProcess);
        }
    }

    public class TimerListener implements MessageListener {

        public MyProcessManager myProcessManager;

        public void onMessage(Message message) {
            int processId = /* get process ID from message */
			MyProcess myProcess = dao.load(processId);
            myProcess.onFaxTimeout();
			dao.save(myProcess);
        }
    }


While this arrangement would appear to work, there are still problems
with it.

First, the original "shape" of the workflow is not reflected in the
code. This can make maintenance difficult and error prone. For example,
note the call to `logFaxComplete` from both `onFaxSent` and
`onFaxTimeout`. Suppose at some time in the future, the business
requests another step after `logFaxComplete`. It would be too easy for a
developer to add the extra step to `onFaxSent` and completely miss that
it should also be added to `onFaxTimeout`.

Second, `FaxListener` and `TimerListener` are specific to
`MyProcess`. It would be preferable if these were reusable
components that could work with any process.

Third, only one branch of the flow should execute for a particular
process. Once one branch executes, the other must be prevented from
executing. In our implementation, the timer event may still occur after
the fax was sent. There are two ways to tackle this.

- We could actively cancel the timer from `onFaxSent`. This works for
  this simple workflow, but it might make it much more difficult in more
  complex workflows to figure out if there are other wait states in
  other parallel branches that need to be cancelled.

- We could have a flag in `MyProcess` that records whether the fax has
  been sent, and check this flag in `onFaxTimeout`. Of course, the
  events could happen in the opposite order, so we'd also have to keep
  track of whether the timeout had already occurred. Finally, if this
  process was part of a larger process that executed it in a loop, we'd
  have to take care to clear the state before entering this portion of
  the workflow.

Both of these approaches suffer from a more subtle problem: a race
condition. If these events occurred sufficiently close in time to one
another, *both* branches of the workflow would be executed. This is a
difficult problem, and one that we'd prefer to solve once, generically.

Fourth, we haven't done a very good job of error handling. Ideally,
`FaxListener` and `TimerListener` would catch any exceptions raised by
the business logic, properly log them, then re-throw them so that the
error message is re-queued and delivered again a little while later.
Ideally, though, we would set a retry limit, and a process exceeding the
retry limit would be frozen pending an investigation. Later, there
should be some mechanism for "thawing" the process once the underlying
problem had been addressed. It would be preferable to centralize this
error handling.

Finally, there is a degree of busy work involved in persisting and
loading the process state. Preferably we'd have some way of having this
happen automatically, without having to write that code in each method.

WorkFlow Lite addresses each of these concerns.


## The WorkFlow Lite Approach

WorkFlow Lite implements three features that address the shortcomings of
the plain action-reaction approach:

- a way to define the shape of a process

- a process lifecycle management, including persisting
  process state, logging significant events, catching and logging
  exceptions, and freezing and thawing processes

- a callback mechanism for decoupling workflow-aware services from the
  processes they notify

The core component responsible for net creation and life cycle management is ProcessManagerImpl.

### Process Shape

Process shape is described by a data structure of  of type `Net`. This
structure gets created by `createNet` method  in `ProcessManagerImpl`.

Nets describing the process shape are built using XML that gets translated
into `Net` structure using lower level constructs described in [design section](design.html).
A net definition for MyProcess will look as follows:

<process>
  <clientRequest method="start"/>
  <parallel name="Parallel" joinCondition="OR">
      <branch name="Branch FaxResponse">
        <controlReceive name="onSent" method="fax_onSent"/>
        <perform name="logFaxSent" method="logFaxSent"/>
      </branch>
     <branch name="Branch Timeout">
        <controlReceive name="onTimeout" method="timer_onTimeout"/>
		<perform name="logFaxTimeout" method="logFaxTimeout"/>
     </branch>
  </parallel>
</process>				

This xml creates a "shape" that allows a process that is started using start
method and then waits for(gets passivated and will get reactivated on) either
fax to be sent event or timeout event. Note that start of MyProcess calls
sendFax and startTimer directly. Otherwise we'd use perform nodes to invoke
each of them individually.

Nodes "controlReceive" define wait states that are crucial to WorkFlow Lite
processes. In our action-reaction implementation, we had to maintain flags
in our process state to determine whether the fax sent notification
occurred before the timeout, or vice-versa. Wait states are simply a
generalization of this idea, freeing us from having to think about how to
enforce the execution of only one branch each time it's encountered.


### Process Lifecycle

We start the process by calling the `ProcessManagerImpl.start` method,
passing canonical name of MyProcess class and parameters that
"MyProcess.start" method accepts. The process manager creates
a new process and executes every step it can until either the process
completes or all branches of the flow have entered a wait state. It then
saves the process state--a combination of our custom process state and a
list of all the active wait states--into its own table. Finally, it
returns to the caller and the process has started.

We can resume the process later by calling the
`ProcessManagerImpl.resume` method, passing the ID of the wait state at
which to resume. The process again continues until the process completes
or all branches have again reached a wait state. If the process's
business logic throws an exception, `resume` logs the exception and
decrements the process's retry count, flagging the process as frozen if
the remaining retries reaches zero.

Note that in practice, `resume` is called directly only from unit test
code, to simulate callbacks from external services. Application code
instead utilizes process callbacks, described next, to resume processes.

If a process becomes frozen, the `ProcessManagerImpl.thaw` mechanism can
be used to replay the last resume call, hopefully allowing the process
to continue on its way.


### Process Callbacks

One question still unanswered is, how do the fax service and timer
service resume the process at the appropriate wait state?

Let's look closely again at where we call the fax service. In the
action-reaction model we passed the process ID to the fax service, which
we used later to call back into the process manager via a custom message
listener when the fax was sent. The process developer had to develop
their own listener for fax response messages that understood the fax
message format in order to retrieve the process ID. The fax service
functionality is therefore poorly encapsulated and highly coupled with
the processes that use it.

WorkFlow Lite solves this more generically by implementing a framework
for process callbacks. In our workflow, we call `createCallback` and
pass the resulting `CallbackInfo` object to the fax service's `sendFax`
method. `CallbackInfo` encapsulates the following data:

- The Spring bean ID of the process manager.

- The WorkFlow Lite process ID.

- A callback *control name*, determined by the process designer (more on
  this later). In this case we simply use "fax".

The fax service stores this away for later use, either in its own
database (`CallbackInfo` objects can be easily converted to and from
human-readable strings), or by using the `CorrelationDao` to
cross-reference the fax ID to the `CallbackInfo` object.

The fax service also implements its own fax response message listener. When
the fax response listener receives a message it determines the related
`CallbackInfo` object, then calls the `InvocationService`, passing the
following arguments:

- The appropriate `CallbackInfo` object.

- An event name, defined by the service. In this case the event name is
  "onSent".

- Any data appropriate to the event.

The `InvocationService` routes the event through a JMS queue, where it
is picked up by the `AsyncDispatcher` and dispatched to the process
manager's `resume` method. The `AsyncDispatcher` ensures that the proper
transaction semantics are used, and that any failures can be re-tried
via JMS message re-delivery.

Finally, let's discuss what the control name is for. A workflow-aware
service might have multiple events that it can fire. For example, the
fax service might have the events `onSent`, `onBusy`, and `onFailed`.
Further, there may be separate points at which a workflow is waiting for
callbacks from the same service. For example, a workflow might send two
faxes to two separate parties, and wait for both to complete before
proceeding.

In the latter case, the two separate requests are differentiated by a
"control name", a string defined by the process designer so as to be
unique across the process. In our example, we use the control names
"fax" and "timer". The control name is part of the `CallbackInfo` object
we pass to workflow aware services.

When the workflow-aware service fires a callback to our process it
specifies the event name. The WorkFlow Lite callback mechanism determines
the process wait state at which to resume by concatenating the control
name and the event name with an underscore. In our example process, we
have a wait state with the ID "fax_onSent". The "fax" portion refers to
the control name we passed to the fax service, and the "onSent" refers
to the event name specified by the fax service.


### JPD Migration Support

JPD workflows can be migrated with a minimum of effort to the WorkFlow
Lite framework.

A JPD is simply a Java class that combines process state and business
logic, along with some embedded XML that defines the process shape. At a
high level, the steps to migrate a JPD to WorkFlow Lite are as follows:

- Rename the JPD with a `.java` extension and remove all references to
  WLI-specific classes.

- Create a process manager class that extends WorkFlow Lite's
  `JpdProcessManager` class. The process manager must implement a start
  method that creates an instance of the JPD class and treats it as the
  process's state.

- In the process manager, implement the `createNet` method and define a
  process net that matches the process shape defined by the XML in the
  JPD.

- In the converted JPD, identify fields that are WLI controls (as
  opposed to those that represent process state) and replace them with
  the equivalents from WorkFlow Lite.

- In the converted JPD, identify calls to any collaborators such as the
  application's EJBs. Convert these to dependency-injection style calls,
  with calls referencing a field with the collaborator's interface. The
  process manager will automatically populate these collaborators
  appropriate time in the JPD's lifecycle.

In most cases, the only changes to the JPD are import statements and
field declarations; all business logic is preserved. Furthermore, since
the process manager and converted JPD are plain Java objects following a
dependency-injection style, it's easy to test the workflow outside the
WLI container using familiar Java unit testing tools. This was simply
not possible with WLI workflows.

WorkFlow Lite provides a detailed [JPD Migration Guide](jpd-guide.html)
that gives more detail and provides step-by-step instructions for JPD
migration.


## Summary

When writing applications that deal with long-running business
processes, a workflow tool is very helpful. Attempting to manage
long-running processes with an ad-hoc action-reaction approach can leave
some difficult issues unresolved. The WorkFlow Lite framework addresses
these issues with a set of three interrelated features: process shape
definition, process lifecycle management, and a process callback
mechanism.


## FAQ

**Q: How does WorkFlow Lite handle transactions?**

A: The answer depends on whether you're starting or resuming a workflow.
The process manager's `start` method requires an active transaction. It
may inherit a transaction from the calling code, or it may have Spring
start a transaction via the `@Transactional` annotation. If this
transaction rolls back, the WorkFlow Lite database tables contain no
trace of the attempt.

The `resume` method executes within the transaction under which the
`AsyncDispatcher` retrieved the callback message. However, unlike
`start`, `resume` writes records to the `ProcessLogEntry` table in new
transactions when the resumed execution starts and finishes. If the
outer transaction rolls back, the message is queued for later delivery
by JMS. This is the basis for WorkFlow Lite's retry mechanism.

WorkFlow Lite does not support declarative transaction demarcation within
a process.


**Q: How does WorkFlow Lite handle exceptions?**

A: The answer depends on whether you're starting or resuming a workflow.
The `start` method simply throws any exceptions back at the caller. It's
up to the caller to log the exception.

The `resume` method catches any exceptions thrown by the workflow and
stores them in the `ProcessLogEntry` table for later diagnosis. It then
re-throws the exception so the message is re-queued on the dispatch
queue and later retried.

If the `resume` method determines that the process has exceeded its
retry count (by counting consecutive failures in the `ProcessLogEntry`
table), it deems the process to be frozen, and the resume request is
ignored.


**Q: What about scalability?**

A: WorkFlow Lite is very lightweight and should perform well. Each resume
involves a select and an update of the process's row in the
`ProcessInfo` table, and two inserts into the `ProcessLogEntry`. Unlike
WLI and BPEL, no remoting is involved, nor is any conversion to or from
XML.

To ensure `resume` calls on the same process happen sequentially,
WorkFlow Lite places a row-level write lock on the process's row in
`ProcessInfo` while `resume` is executing. No locks are held while the
process is in a wait state, and no locks are held that affect multiple
process instances.

Finally, assuming callback queues are distributed across the cluster,
WorkFlow Lite processes may execute on any node in the cluster. This
should allow the capacity for WorkFlow Lite processes to be scaled nearly
linearly with new cluster nodes, assuming scalability is not otherwise
compromised by application code.


**Q: How does WorkFlow Lite *guarantee* that only one branch of an
OR-join executes?**

A: Each process instance has a row in the `ProcessInfo` table. WorkFlow
Lite uses a row-level lock on the `ProcessInfo` table to ensure that
only one thread can work on a particular process instance at a time.

Suppose, in the example above, that a fax response and its timeout
notification arrive very close together on two separate nodes on the
cluster. Each thread will attempt to execute code like this:

    select *
    from ProcessInfo
    where id = :id
    for update

The first thread to actually make it through to the database will
execute this statement, which places a write lock on the process
instance's row. The second thread will be blocked when it attempts to
put a write lock on the same row.

The first thread then proceeds to execute its branch, which cancels
the wait state in the other branch as it completes. The first thread
eventually commits, releasing the lock on the row.

At this point the second thread proceeds. It loads the process net's
state but finds that the wait state it was about to fire has already
been cancelled by the first thread. It then simply returns silently
without executing the rest of the branch.
