---
layout: page
title: WorkFlow Lite Deployment
---

# Deployment guide

## Packaging

WorkFlow Lite is a lightweight JAR library designed to be embedded within
the applications that use it rather than being deployed separately. This
approach provides some opportunities to simplify the packaging and
deployment of applications while reducing infrastructure costs compared
to that of WLI.

WorkFlow Lite is typically deployed as part of an ear. In order to function
properly it needs access to process classes. This simply means that those
classes have to be part of a jar deployed packaged in the same EAR. For details
of application configuration refer to [Spring Context section of JPD Migration Gude](jpd-guide.html)



## Runtime Library Dependencies

WorkFlow Lite internally utilizes several libraries that have to be
available to it at runtime. They can be deployed within the same
application as wflite.jar, made available through the classpath or as
[shared libraries](http://download.oracle.com/docs/cd/E15051_01/wls/docs103/programming/libraries.html)
   <table class="normaltable">
     <tr>
       <th>Library</th>
       <th>Version</th>
     </tr>
     <tr>
       <td>spring</td>
       <td>2.5.6+</td>
     </tr>
     <tr>
       <td>log4j</td>
       <td>1.2+</td>
     </tr>
     <tr>
       <td>commons-logging</td>
       <td>1.1+</td>
     </tr>
     <tr>
       <td>commons-lang</td>
       <td>2.5+</td>
     </tr>
   </table>

## Application Servers

WorkFlow Lite currently supports RedHat JBoss, Oracle WebLogic and IBM Websphere.

## Database

WorkFlow Lite relies on relational database for storing process state,
correlation, message and logging information. To configure your database
create a DB schema and execute ddl script corresponding to the
database you are using. WorkFlow Lite interacts with the database using XA
datasource, that by default should be bound under JNDI name wf.TxDataSource.
