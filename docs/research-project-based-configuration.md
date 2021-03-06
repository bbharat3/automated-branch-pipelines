# Research on making a project based configuration

There are still some requirements to meet before the POC is ready for use:

* The part of the configuration that belongs to a project should not be coupled to the
  ABP service. Thus, it should live in a configuration file in the project's repository.
* Project configuration is thus delivered to the service and the service then reacts on
  what it receives.

Such a configuration have the following benefits:

* The responsibility of the project configuration of a branch pipeline is on the project members
* Projects will not interfere with each other and the ABP service does not need to be
  updated with new configuration when-ever a project decide on something else
* There will be VCS history of the projects configuration
* The ABP service configuration file does not grow very large and only contain ABP configuration

## Goal for changes

Move the project related configuration from the ABP service to a project specific configuration file.

## Background

Currently the project specific configuration includes:

* CI server URL
* Jenkins seed job name
* Definition of pipelines: Map from branch prefix to a list of job names

_The first POC of ABP is written for Jenkins, Jenkins Job DSL, and a generic URL trigger._

In the write-up about [Analysis of responsibilities and data ownership in ABP](analysis-of-responsibilites-and-data.md) there is a summary of thoughts on what parts of the system naturally own data and events.

### Key challenge

The absolute key challenge in the current design, is that the project configuration file describing the build pipeline is expected to be in a SCM.
That file and the SCM can not be expected to be accessible from the SCM trigger mechanism. From our point of view we would prefer a design, where the trigger mechanism also explains the recipe for creating the new build flow, thus the trigger mechanishm needs to read the project configuration file.

On command line within the project that needs the build pipeline, it would be easy, but from many SCM hook mechanisms there is not file or repository access.

## Questions

These questions heavily influence our decisions:

* How important is the abstraction of CI server / build system? This would be pretty
  easy to solve if we focused on Jenkins only. But if we focused on Jenkins, the ABP
  service becomes obsolete.
* If we want to keep the ABP service and the build system abstraction, is it worth
  considering a Jenkins specific solution using the same library code? It might be an
  easier setup to use for some customers.


## Approach 1: Add configuration to SCM trigger

The SCM (e.g. Bitbucket) is the system sending requests to the ABP service. It currently
passes JSON containing things like repository and branch name. It could be extended to
include the project configuration.

Then the JSON payload would in essence become the project configuration file stored in
SCM.

Pros:
* Easy to adjust the current ABP service to handle this

Cons:
* Harder to setup. Even if the Bitbucket hook can be configured programmatically (via
  Bitbucket's REST API), project's would need a script to send the configuration to
  Bitbucket once for every configuration change. Otherwise, this would have to be done
  manually in the Bitbucket UI.
* It might not be a generic solution either, as SCM hook support are very different and we also want to allow to just use curl or similar to trigger the event manually.


## Approach 2: The ABP service uses SCM to read the configuration

Even though the configuration is project specific, it is interpreted and used by the ABP
service. Thus, the ABP service could be extended to read from a project's SCM repository.

Pros:
* There is no issue with how to pass the information to the ABP service.

Cons:
* Git/SCM client has to be installed on the system running the ABP service, or
  implemented in Java by the ABP service.
* SCM access: The system running the ABP service needs network access to each project's
  repository. This may not be a problem in practice, since the build system also needs
  this access.
* SCM configuration: This will add to the service configuration a list of
  projects that it can handle and the projects repository details. So this means we still have specific configuration in the ABP configuration file, which is what we try to avoid.
  We would also have to
  decide on a naming convention for the project configuration file (like how Jenkins 2.0
  uses the `Jenkinsfile`).


## Approach 3: The configuration is read when running the job

Since the configuration is in SCM, it could just be read when running the build job on
Jenkins, using Job DSL.

Pros:
* Easy to setup using Jenkins. The system architecture becomes simpler because the
  ABP service component is no longer needed.

Cons:
* The ABP service would become obsolete. Its functionality would be implemented in Job
  DSL or a Jenkins plugin instead.
* We mix the logic from the ABP service with the projects build pipeline configurations 
  as we have to be re-implemented the logic from ABP with the Job DSL. Also we need to 
  share and distribute that common ABP logic between several projects this way.
* The solution will only be able to handle Jenkins as build system.


## Approach #4: Jenkins job calling the ABP service

This can be seen as a special case of number 3 above. It would be a quick way to
experiment with a setup where Bitbucket calls Jenkins directly.

Instead of reimplementing the service logic in either Groovy/Job DSL or packaged as a
Jenkins plugin, we can define a Job DSL job invoked by Bitbucket. The job reads the
project configuration from SCM and the ABP service. The service will then, as today, call
other jobs on the same Jenkins or even a different Jenkins instance.

The service can be called in different ways:

* With HTTP like Bitbucket does today. This has the disadvantage that the service needs
  to be started. It also still needs to be modified to receive project configuration in
  requests.
* As a standalone jar. Job DSL can call out to execute a process on the Jenkins master.
  The ABP service needs to be modified in order to do this, so it shuts down when done.


## Approach #5: A stateful service

If we made the ABP service stateful, we could accomplish being able to add and remove
projects dynamically by letting the projects register with themselves with the service.

This would require a significant amount of work to the service, for example
adding new API calls and making a decision about persistence.

We would also need to consider how projects are added - is it done via a script when
starting a new project or is it tied to the SCM?


## Conclusion

If we want to keep the build system abstraction, the choice is either approach number 1,
2, or 5.

The 1st approach requires few implementation changes, but introduces a manual
setup step that has to be carried out whenever a project's pipeline changes.

The 2nd approach still requires a bit of project configuration in the service, namely how
to access the SCM to read the configuration. However, this should be a one-time setup.
Otherwise, it requires little from the projects using the ABP service, just that the
configuration file follows a naming convention. It does require a bit of implementation
changes in the ABP service.

If the top priority is to satisfy a specific set of tools, so that is okay locking in to Jenkins,
then the 3rd approach provides the simplest user experience. It could be combined with
still having the ABP service for other use cases, by sharing some of the Java code in a
library.
