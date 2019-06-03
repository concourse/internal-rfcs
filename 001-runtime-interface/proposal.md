# Runtime/Storage Interface

# Summary

Currently, build execution in Concourse is very tightly coupled to Garden and Baggageclaim. For
each step, we directly create a Garden container and any associated Baggageclaim volumes and use
those as necessary to run each step of the build and store any inputs/outputs.

This RFC is to change this flow to use a "runtime interface" for these aspects of build execution.
This interface can then be implemented by multiple backends, e.g. Garden, Docker, Kubernetes etc.

# Motivation

There is no good reason for Concourse Core concepts, such as `get`, `put`, `task` and `check` to be
tied directly to containers and volumes as units of execution and storage.

The current architecture forces the "bleeding" of separate concerns into different packages.
As an example, when executing a step, the Concourse Core needs to create a container. But because
containers live in Garden on a worker, Core first needs to select a worker to run the container. To
select a worker, Core needs to be aware of the existence of a "pool" of workers, and a "strategy"
to use to select the best worker. So for each step, Core first chooses a worker, then creates the
step container on that worker. But in an ideal world, Core wouldn't need to care which worker a
container runs on; its only concern is that the step is executed successfully.

Moreover, the concept of a "pool" of worker VMs is directly tied to the Garden runtime, and may not
need to exist in other runtimes.

# Proposal

The proposal is to separate out Core and Runtime concerns and provide an API to communicate across
the boundary.

Here are some examples of generic runtime concerns (i.e. platform specific implementation
details) that Core shouldn't need to worry about:
* Implementing units of execution - currently Garden containers
* Implementing units of storage - currently Baggageclaim volumes

Here are some other ancillary concerns that exist in our current worker-based runtime that Core
should not need to know about:
* Garbage collecting unused containers/volumes
* Worker selection - Container placement strategies
* Worker lifecycle - registration, heartbeating, resource usage
* Base resource types - potentially differing versions of base resources on each worker

The idea is to create abstractions that are implemented by these concepts, then design an API which
uses these abstractions to simplify Core concepts. Here are the abstractions we have in mind:
* `Runnable` - unit of execution for a step
* `Artifact` - unit of storage for inputs/outputs/images/caches

There are two interfaces that emerge from these abstractions -

1) Runtime:
```
type Runtime interface {
  Run(spec RunnableSpec) Result
}
```

2) Storage:
```
type Storage interface {
  CreateArtifact() Artifact
  DestroyArtifact(id)
}
```

where the Artifact in question looks like this:
```
type Artifact interface {
  Store()
  Retrieve()
}
```

Each interface is described in more detail below:

## Runtime Interface
This interface represents the ability to run a particular unit of work. The `RunnableSpec` should
contain information about what to run and how to run it. `Run` should return the `Result`, which is
the outcome of running the `Runnable`, and any error that may have been thrown.

Each Concourse step can be run using a `Runnable`. In the current runtime, the `Run()` method can
abstract away worker selection, container placement strategies, build scheduling and container \
creation from the `exec` package.

The Runtime interface allows us to easily create wrapper implementations for other runtimes, e.g.
Docker, without needing to modify core concepts such as `get`, `put`, `task` and 
`check`.

## Storage Interface
Because Concourse allows for outputs from previous steps to be used as inputs to future steps, we
need the `Storage` abstraction. This should be used to model concepts like step inputs, outputs and
caches.

Depending on the kind of step (or check) being run, Core can create the required input, output and
cache artifacts using `CreateArtifact`. The `Artifact`s created can then be included as part of the
`RunnableSpec` for the next `Runnable`. Consider the following pipeline snippet as an example:

pipeline.yml:
```
jobs:
- name: unit
  plan:
  - get: booklit
    trigger: true
  - task: test
    file: booklit/ci/test.yml
```

test.yml:
```
---
platform: linux

image_resource:
  type: registry-image
  source: {repository: golang}

inputs:
- name: booklit

run:
  path: booklit/ci/test
```

When setting up the build plan, the Concourse Core knows that the output of the `get` step is an 
input to the `task` step. Step execution would be as follows:

1) `storage.CreateArtifact()` to represent "booklit"
2) `runtime.Run()` the `get` step with the "booklit" `Artifact` declared as an output in the
   `RunnableSpec`
3) `runtime.Run()` the `task` step with the "booklit" `Artifact` as an input in the `RunnableSpec`
4) `storage.DestroyArtifact()` "booklit" when no longer required

Note that in this case, Core does not care about the actual contents of the `Artifact`; it assumes
that if the `get` was successful, the `Artifact` is correctly populated.

In the current runtime, `Storage` would be modeled as a distributed "blobstore" on different
workers (using Baggageclaim). `CreateArtifact` could simply create a database artifact object
that isn't backed by any actual storage until one of two things occur:
1) `artifact.Store()` is called
2) the `Artifact` is passed in as an input/output for a `Runnable`

One implication of this interface is that Core would need to manage the lifecycle of an `Artifact`.
Note that this does not mean that Core would need to manage the volume lifecycle; instead, the
volume lifecycle would depend on the artifact so that once the `Artifact` is deleted, any associated
volumes would also get deleted.

## Artifact Interface

In most cases, the Concourse Core does not need to know the actual contents of an `Artifact`; the
contents are only really used when the artifact is attached to a `Runnable`. However, there are a
few cases where code that lies above the `Run/Store` abstraction needs to read the contents of an
artifact. They are:

1) *Running `fly execute`:* Users may upload or download user artifacts from their local machine.
Since these `Artifact`s need to be streamed to/from `fly`, their contents need to be exposed to
the API.
2) *Fetching images:* Because Concourse allows the user to specify an `image_resource` can be
specified in the task definition as opposed to the pipeline, there exists a use case where a user
may first do a `get` and then run a `task` that was defined in the output `Artifact` of the `get`.

To satisfy these use cases, we need to `Store()` and `Retrieve()` information from within an
`Artifact`.

# Open Questions

## Kubernetes Runtime
If we were to model the Kubernetes runtime using these interfaces, one logical option would be to
have our `Runnable`s be Kubernetes pods and our `Artifact`s be objects in a blobstore. However,
concepts such as resource caches, task caches and inputs/outputs might be difficult to model under
this scenario. It may be worth investigating other solutions for a Kubernetes runtime.

# Answered Questions

> If there were any major concerns that have already (or eventually, through
> the RFC process) reached consensus, it can still help to include them along
> with their resolution, if it's otherwise unclear.
>
> This can be especially useful for RFCs that have taken a long time and there
> were some subtle yet important details to get right.
>
> This may very well be empty if the proposal is simple enough.


# New Implications

These changes should *not* affect users, they serve only to simplify the code base and make future
runtime/storage implementations easier. For example, if we decide to swap out Baggageclaim with a
centralized blobstore, the only code that would need to change is the implementation of the
`Storage` interface.
