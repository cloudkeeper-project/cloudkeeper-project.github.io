---
layout: structured
title: Runtime Systems based on CloudKeeper
permalink: /runtime
---

# Runtime Systems

All (practically useful) programming languages rely on some sort of runtime system, which provides the means for executing constructs defined in the respective language. Runtime systems typically provide additional services such as a debugger, profiling, or optimization, in order to facilitate quick development cycles. As an example, the Java language crucially relies on the Java Virtual Machine for executing Java programs (after translation into byte code). The JVM also simplifies life for the Java developer by providing advanced debugging facilities (even remotely), live monitoring (e.g., of system-resource usage), or the HotSpot optimizer.

CloudKeeper is a domain-specific language and runtime system for _dataflows_. Unlike the JVM, however, CloudKeeper does not have a “default” runtime system that is available as stand-alone executable. Instead, CloudKeeper is designed as a library that allows:

* easy embedding of a CloudKeeper runtime system in other JVM-based programs and
* easy composition of pre-defined (or user-defined) subsystems into very tailored CloudKeeper runtime systems.

As an example, the CloudKeeper core project currently includes three pre-defined staging-area implementations: in-memory, file-based, or S3-based. Similarly, there are three pre-defined options for simple-module executors: one that executes simple modules in tasks submitted to a Java `Executor` in the same JVM, an executor that forks a new JVM for each simple-module execution, and another one that wraps simple-module executions in a DRMAA job (for execution with a distributed resource manager such as Grid Engine).

It is worth noting that the JVM can be embedded, too (using the [Invocation API](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/invocation.html)). Still, the usual `java` command-line utility covers far more of the typical use cases than a dedicated CloudKeeper stand-alone executable ever could.

## The Java Interface

The basic interfaces for executing a dataflow and retrieving results are very simple.

* A [`WorkflowExecution`](https://github.com/cloudkeeper-project/cloudkeeper/blob/master/cloudkeeper-core/cloudkeeper-model/src/main/java/com/svbio/cloudkeeper/model/api/WorkflowExecution.java) represents a dataflow execution. Instances are used to query the status of a dataflow execution and to (asynchronously) retrieve results.
* A [`WorkflowExecutionBuilder`](https://github.com/cloudkeeper-project/cloudkeeper/blob/master/cloudkeeper-core/cloudkeeper-model/src/main/java/com/svbio/cloudkeeper/model/api/WorkflowExecutionBuilder.java) is used to start dataflow executions. Following the [builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) common in Java, the builder is used to take optional arguments. Specifically, these are:
  * inputs,
  * meta data, and
  * URIs identifying CloudKeeper dataflow libraries needed for the execution.
* A [`CloudKeeperEnvironment`](https://github.com/cloudkeeper-project/cloudkeeper/blob/master/cloudkeeper-core/cloudkeeper-model/src/main/java/com/svbio/cloudkeeper/model/api/CloudKeeperEnvironment.java) represents a CloudKeeper runtime system and is used to create the workflow execution builder for a given CloudKeeper module.

## Example

The genome-analysis dataflow presented as [Language example](/language#example) can be started used the following code (assuming an existing `CloudKeeperEnvironment`):

{% highlight java %}
GenomeAnalysisModule module = moduleFactory.create(GenomeAnalysisModule.class)
  .reads().fromValue("ACT\nTACTG\nGTAC");
WorkflowExecution execution = module
  .newPreconfiguredWorkflowExecutionBuilder(cloudKeeperEnvironment)
  .start();
String report = WorkflowExecutions
  .getOutputValue(execution, module.report(), 1, TimeUnit.SECONDS);
{% endhighlight %}


## CloudKeeper Modularity

The [CloudKeeper core project](https://github.com/cloudkeeper-project/cloudkeeper) contains a CloudKeeper environment implementation that uses a given class loader to load dataflow and module definitions, performs all computation in a given `Executor` instance, and keeps all intermediate results in a Java `Map`. Each of these three components (that is, the so-called “[runtime-state provider](https://github.com/cloudkeeper-project/cloudkeeper/blob/master/cloudkeeper-core/cloudkeeper-api/src/main/java/com/svbio/cloudkeeper/model/api/RuntimeStateProvider.java)”, the “[simple-module executor](https://github.com/cloudkeeper-project/cloudkeeper/blob/master/cloudkeeper-core/cloudkeeper-api/src/main/java/com/svbio/cloudkeeper/model/api/executor/SimpleModuleExecutor.java)”, and the “[staging area](https://github.com/cloudkeeper-project/cloudkeeper/blob/master/cloudkeeper-core/cloudkeeper-api/src/main/java/com/svbio/cloudkeeper/model/api/staging/StagingArea.java)”) are just Java interfaces that can be replaced with arbitrary implementations. 

The flexibility of combining individual components of the CloudKeeper library implies that the code dealing with configuration and wiring of components can be significant. Therefore, an “all-inclusive” abstraction layer exists in a [separate project `cloudkeeper-all-inclusive`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive) that spares different tools from having to write the same boilerplate code over and over again (necessarily at the expense of some flexibility).

While core CloudKeeper components are typically configured using builders, the “all-inclusive” layer relies exclusively on one or more configurations files that use the [HOCON ("Human-Optimized Config Object Notation") syntax](https://github.com/typesafehub/config/blob/master/HOCON.md) of the [Typesafe Config](https://github.com/typesafehub/config) library. Moreover, the [Dagger 2](http://google.github.io/dagger/) dependency-injection library is used in order to create instances as needed. In contrast, the CloudKeeper core project aims to have as few third-party dependencies as reasonably possible and requires manual instantiation.


# An “All-Inclusive” Runtime System 

The rest of this document describes the “all-inclusive” abstraction layer provided by project `cloudkeeper-all-inclusive`.

## Configuration

Corresponding to the architecture (see below), configuration is split across three different Maven projects. The most up-to-date list of available configuration settings is implicitly given by all configuration classes within the `<Name of Dagger Module>Module.java` files in the following source trees.

* [Workflow: RuntimeContext factory](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-runtimecontext) (workflow-runtimecontext)
* [Workflow: Forked Simple-Module Executor](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-forked-executor) (workflow-forked-executor)
* [Workflow: Service](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-service) (workflow-service)

Alternatively, the list of all _default_ settings defined in the [reference.conf](https://github.com/typesafehub/config/blob/master/HOCON.md#conventional-configuration-files-for-jvm-apps) files of the above projects ([workflow-runtimecontext](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-runtimecontext/src/main/resources/reference.conf), [workflow-service](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-service/src/main/resources/reference.conf)) _should_ be as comprehensive. The documentation below goes into additional detail.

## Architecture

This project provides the following architecture on top of core CloudKeeper:

![High-level Workflow Architecture](/assets/all-inclusive-architecture.svg){: .img-responsive}

The CloudKeeper library provides all functionality, whereas the high-level abstraction layer adds common configuration and glue code on top. The following subsections describe the three kinds of modular components that constitute a runtime system.

## Runtime-Context Provider

A _runtime-context provider_ is the CloudKeeper component that _loads_ and _links_ CloudKeeper plug-in declarations so that modules can be interpreted and executed. Corresponding to its name, the runtime-context provider creates a data structure called the _runtime context_: It contains the CloudKeeper _repository_ of linked plug-in declarations as well as a Java class loader that can be used by marshaler plug-ins.

_Loading_ is the process of locating bundles of plug-in declarations and transforming them into an abstract-syntax tree in memory. _Linking_ is the process of taking abstract syntax trees and combining them into the runtime data structures that can be interpreted and executed. In particular, the linking process includes resolving symbolic references (that is, names) into Java object references, verifying integrity, etc. The linking process also associates marshaler declarations with Java classes – which is obviously necessary to invoke them. A _repository_ is a set of plug-in declarations that is transitively closed; that is, all plugin-declarations in a repository only refer to other declarations also contained in the repository.

In the abstraction provided by this project, two kinds of runtime-context providers can be configured (using setting `com.svbio.workflow.loader`):

1. The Maven-based runtime-context provider understands bundle identifiers of form `x-maven:<groupid>:<artifactid>:ckbundle:<version>`. The ckbundle artifact identified by the Maven coordinates must have been built using the CloudKeeper Maven plugin. All bundle artifacts, including their transitive dependencies, are resolved using the [Eclipse Aether](http://www.eclipse.org/aether/) library – if necessary and configured, this may include downloading artifacts from a remote Maven repository.

   Each CloudKeeper bundle artifact is an XML file that contains a serialized [`MutableBundle`](https://github.com/cloudkeeper-project/cloudkeeper/tree/master/cloudkeeper-core/cloudkeeper-model/src/main/java/com/svbio/cloudkeeper/model/beans/element/MutableBundle.java) instance. By default, a Maven-based runtime-context factory also resolves all JAR files and their transitive dependencies. From these JAR files a `URLClassLoader` is created, which is used as the new runtime context's class loader (such as for loading the Java classes corresponding to CloudKeeper plug-in declarations and for user-defined marshalers).

   However, it is not normally necessary to dynamically create a class loader (except for simple-module executors), and it is in fact normally better not to do so. Advantages of using a fixed class loader (typically the system or the current class loader) are:
   
   1. The overhead of creating a new class loader can be avoided.
   2. The inputs and outputs of a workflow execution are expected to be instances of classes already loaded.   

   In order to use a fixed class loader, one has to instantiate the `MavenRuntimeContextFactoryModule` manually with the single-argument constructor. Example:
     
   ```java
   RuntimeContextComponent runtimeContextComponent
       = DaggerRuntimeContextComponent.builder()
     .configModule(new ConfigModule(config))
     .lifecycleManagerModule(new LifecycleManagerModule(lifecycleManager))
     .mavenRuntimeContextFactoryModule(new MavenRuntimeContextFactoryModule(
       ClassLoader.getSystemClassLoader()
     ))
     .build();
   ```

2. The DSL-class-walker runtime context understands bundle identifiers of form `x-cloudkeeper-dsl:<class name>`, where `class name` must be the name of a DSL module class. This runtime-context provider creates a repository consisting of the given module declaration and all its transitive references. A new class loader is never created, and therefore it must be guaranteed that the plug-in declaration Java classes are available on the classpath.

## Simple-Module Executor

A _simple-module executor_ is the CloudKeeper component that evaluates a simple module with given inputs. It is a functional interface that takes a _runtime state_ as input and returns an execution result. The runtime state comprises:

* the _runtime context_ (see above), which itself contains both the repository of CloudKeeper plug-in declarations and a Java class loader,
* the current call stack, and
* the staging area.

That is, the runtime state represents the entire state of a workflow execution. Sending it to a different JVM means that the execution can be seamlessly continued. Note that a runtime state is not usually self-contained and typically does not reside exclusively in main memory. Instead, it typically contains symbolic references; e.g., a path in the file system that contains the staging area.

This project offers two kinds of simple-module executors (using setting `com.svbio.workflow.executor.invocation`):

1. The forking executor (see class [`ForkingExecutor`](https://github.com/cloudkeeper-project/cloudkeeper/tree/master/cloudkeeper-executors/cloudkeeper-forking-executor/src/main/java/com/svbio/cloudkeeper/executors/ForkingExecutor.java)) invokes a command line for each execution of a simple module. The actual command line invoked for a simple module is built from a template. This template is a list of strings, and it is configured with setting `com.svbio.workflow.executor.commandline`. Each string in the list is expected to be a either:

   1. a format string that will be passed to [`String#format(String, Object...)`](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html#format-java.lang.String-java.lang.Object...-) together with two arguments:

      1. the value of [`Requirements#cpuCores()`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/workflow-bundles/workflow-bundle-core/src/main/java/com/svbio/workflow/bundles/core/Requirements.java) and
      2. the product of `Requirements#memoryGB()` and the value of setting `com.svbio.workflow.executor.memscale`.

   2. one of the following placeholders that will be expanded into a (sub-)list:

      1. `<classpath>` will be replaced by a collection of code sources taken from the current class loader and sufficient to run `com.svbio.workflow.executor.CloudKeeperExecutor`
      2. `<props>` will be replaced by configuration-relevant system properties, that is, by a list of entries of form `"-Dkey=value"` where `key` is either "`config.file`" or starts with "`com.svbio.workflow`" and `value` is the value of the respective system property in the current JVM.

2. The DRMAA executor (see class [`DrmaaSimpleModuleExecutor`](https://github.com/cloudkeeper-project/cloudkeeper/tree/master/cloudkeeper-executors/cloudkeeper-drmaa-executor/src/main/java/com/svbio/cloudkeeper/drm/DrmaaSimpleModuleExecutor.java)) likewise creates command lines, but submits them as a new DRMAA job. Similarly to the command line, it also creates DRMAA native arguments from the format string in setting `com.svbio.workflow.drmaa.nativespec`. It will be passed to [`String#format(String, Object...)`](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html#format-java.lang.String-java.lang.Object...-) together with two arguments:

   1. the value of `Requirements#cpuCores()` and
   2. the product of `Requirements#memoryGB()` and the value of setting `com.svbio.workflow.drmaa.memscale`.
   
   Typically, the DRMAA `memscale` factor should be chosen slightly larger than the one used for the JVM argument of form `"-Xmx%2$dm"`. The reason is of course obvious: A JVM may need more memory than just the maximum heap size.

## Staging-Area Provider

A _staging-area provider_ is an interface specific to this project that creates the CloudKeeper staging area used for a single workflow execution. It takes two parameters:

1. an identifier (also referred to as _prefix_) for the new workflow execution and
2. a flag whether intermediate results should be removed.

Two kinds of staging-area providers can be configured (using setting `com.svbio.workflow.staging`):

* A file-based staging-area provider (see class [`FileStagingAreaService`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-service/src/main/java/com/svbio/workflow/service/FileStagingAreaService.java)). This staging-area provider has a base path, and for each workflow execution a new subdirectory will be created, using the execution-specific identifier as name. For instance, if the base path is `/data` and the execution identifier is `52/`, then then staging area for this execution would be rooted at `/data/52`.

    For optimization purposes, setting `com.svbio.workflow.filestaging.hardlink` can be used to define so-called _hard-link-enabled_ paths. It is often useful to make the temporary directories used by the simple-module executor hard-link enabled: that is, adding path `${com.svbio.workflow.executor.invocation}` to the list of hard-link enabled paths. That way, simple modules that need to copy byte sequences into their working directory would simply perform hard links instead.
    
* An S3-based staging-area provider. This provider works very similar to the file-based staging-area. The layout of the S3 objects is equivalent to the layout of files in the file system.

Both file- and S3-based staging areas marshal Java objects into trees of byte sequences. Each value for an in- or out-port will be represented as a file or as a directory. For each port, metadata is stored about the marshaler used to create the representation. Example: Assuming a simple module has an in-port 'number' of type `Integer`, two files would be created:

1. a file with name "number", containing the integer as string, e.g., "24"
2. a file with name "number.meta.xml", containing the chain of marshaler plug-in used to create the representation (note that, by default, the `IntegerMarshaler` simply converts `Integer` objects to `String`, for which then the `StringMarshaler` would be invoked):

   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
   <object-metadata xmlns="http://www.svbio.com/cloudkeeper/staging/2.0.0">
     <marshalers>
       <marshaler>
         <name>cloudkeeper.serialization.IntegerSerialization</name>
         <bundle-identifier>x-ck-system-bundle:2.0.0.0</bundle-identifier>
       </marshaler>
       <marshaler>
         <name>cloudkeeper.serialization.StringSerialization</name>
         <bundle-identifier>x-ck-system-bundle:2.0.0.0</bundle-identifier>
       </marshaler>
     </marshalers>
   </object-metadata>
   ```

## The High-Level Java Interface

When writing ad-hoc tools or scripts to embed CloudKeeper, project [`workflow-api`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-api) provides essentially a single functional interface of relevance: [`WorkflowService`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-api/src/main/java/com/svbio/workflow/api/WorkflowService.java). This interface provides method `create()` for creating a [`CloudKeeperEnvironment`](https://github.com/cloudkeeper-project/cloudkeeper/tree/master/cloudkeeper-core/cloudkeeper-model/src/main/java/com/svbio/cloudkeeper/model/api/CloudKeeperEnvironment.java) instance. It requires the same two arguments as documented for the staging-area provider: an identifier (prefix) and whether cleaning of intermediate results is requested. Note that `CloudKeeperEnvironment` is a core CloudKeeper interface that allows creating workflow executions in the usual way.

Alternatively, `WorkflowService` also provides simple start/stop/status methods. In this case, the assumption is that inputs are already ready to be picked up from the configured staging area.

## RESTful Web Interface

The simple start/stop/status methods mentioned in the previous section can be easily mapped to a RESTful interface. In fact, project [`workflow-servlet`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-servlet) does exactly that. It contains a class [`WorkflowServiceResource`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-servlet/src/main/java/com/svbio/workflow/servlet/WorkflowServiceResource.java), that defines the REST endpoints. The workflow servlet component can be easily embedded into other projects by simply creating an `HttpServlet` instance using [`WorkflowServletComponent`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-servlet/src/main/java/com/svbio/workflow/servlet/WorkflowServletComponent.java). EclipseLink MOXy is used for serialization of HTTP requests and responses, meaning that both XML and JSON are supported.

* `POST /api/executions`
  
  Start a new workflow execution. On successful execution, will respond with a redirect to `/api/executions/{eid}`, where `eid` is the newly assigned CloudKeeper execution ID. Example request:

  ```xml
  <execute-workflow-request xmlns:ck="http://www.svbio.com/cloudkeeper/1.0.0">
    <ck:proxy-module ref="svbio.ckmodules.pipeline.TumorPipelineModule"/>
    <bundle-identifiers>
      <bundle-identifier>
        x-maven:com.svbio.ckmodules:ckmodules-toplevel:ckbundle:1.0.1.12-SNAPSHOT
      </bundle-identifier>
    </bundle-identifiers>
    <prefix>52/</prefix>
    <cleaning-requested>true</cleaning-requested>
  </execute-workflow-request>
  ```

* `DELETE /api/executions/{eid}`

  Stop workflow execution with execution id `eid`.
    
* `GET /api/executions/{eid}`

  Return status of workflow execution with execution id `eid`. Example response:

  ```xml
  <execution-status xmlns:ck="http://www.svbio.com/cloudkeeper/1.0.0">
    <execution-id>1</execution-id>
    <request>
      <ck:proxy-module ref="svbio.ckmodules.pipeline.TumorPipelineModule"/>
      <bundle-identifiers>
        <bundle-identifier>
          x-maven:com.svbio.ckmodules:ckmodules-toplevel:ckbundle:1.0.1.12-SNAPSHOT
        </bundle-identifier>
      </bundle-identifiers>
      <prefix>52/</prefix>
      <cleaning-requested>true</cleaning-requested>
    </request>
    <status>RUNNING</status>
    <failure-description/>
  </execution-status></pre>
  ```

  The execution status is available also for jobs that have already completed.

# Modularity in Practice

The usefulness of a programming language is crucially determined by its support for the entire engineering process, and CloudKeeper makes no exception. Workflows written in CloudKeeper need to be easy to debug as well as reliable to deploy. In fact, CloudKeeper needs to support an additional use case not relevant for many other languages: Given the large size of the inputs that CloudKeeper is frequently used with, tests often need to happen in a distributed fashion and on machines that typically are not development machines – meaning that executable code frequently has to be redeployed to the test machine.

At least the following use cases may be relevant:

|                 | Execution Environment | Source Repository | Artifact Repository |
| --------------- | --------------------- | ----------------- | ------------------- |
| **Development** |
| Debugging       | single JVM on laptop  | not checked in    | not checked in      |
| Smoke Tests     | multiple JVMs on laptop | 〃              | not checked in or snapshot |
| Realistic Tests | cluster               | 〃                | snapshot            |
| **Production**  |
| Real Data       | 〃                    | checked in        | release             |
{: .table}

## Example Projects

The following examples illustrate the use of project `cloudkeeper-all-inclusive`:

1. [CloudKeeper bundle](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-samples/workflow-sample-ckbundle)
2. [Embedded Workflow Service](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-samples/workflow-sample-embedded-ck)

## A Production Runtime System

A CloudKeeper runtime system that may be used as stand-alone server is provided by the simplistic project [`workflow-server`](https://github.com/cloudkeeper-project/cloudkeeper-all-inclusive/tree/master/workflow-server). It simply includes the `workflow-servlet` component described above and starts it in a Jetty server.
