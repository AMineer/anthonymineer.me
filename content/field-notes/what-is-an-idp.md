---
title: "What Is an Internal Developer Platform?"
date: 2026-09-01
tags: [platform-engineering, idp, developer-experience, devops, gitops, kubernetes, ai]
draft: false
---

## Start here

A few weeks ago I wrote an article about Port.io and some of its use cases. I realized that maybe I should also cover what an Internal Developer Platform or IDP for short is. So to get us kicked off we are going back to basics and exploring IDPS and their various flavors. 

Internal Developer Platform is one of those terms that somehow gets less clear the more people use it.

Depending on who you ask, an IDP is Backstage. Or Port. Or Kubernetes. Or a collection of Terraform modules. Or a developer portal. Or a service catalog. Or the thing your platform team has been building for three years that nobody is quite sure how to explain.

Some of those can be parts of an IDP. None of them, by themselves, are a particularly useful definition of one.

There are plenty of formal definitions. The CNCF describes platforms broadly as collections of capabilities, documentation, and tools that support developing, deploying, operating, and managing software delivery.

For the rest of this article, I am going to use a slightly more practical definition:

**An Internal Developer Platform is the set of capabilities an engineering organization assembles so developers can build, deploy, and operate software through supported paths without needing to understand every implementation detail underneath them.**

The important part is "set of capabilities."

An IDP is not necessarily one product. It is usually a collection of systems connected together behind an opinionated interface.

That interface might be a developer portal. It might be Git. It might be a Kubernetes API. It might even be a set of well-built CI templates.

The implementation matters less than what the platform actually allows developers to do.

That is where I think a lot of the confusion starts.

## The portal is not the platform

Let's clear this one up first.

A developer portal and an Internal Developer Platform are related, but they are not the same thing.

A portal is usually the front door.

It gives developers a place to find things:

- services
- documentation
- ownership
- scorecards
- environments
- dependencies
- deployment information
- self-service actions

An IDP is what actually makes those actions work.

A simple version looks something like this:

```text
Developer
   |
   v
Developer Portal
   |
   +-- Service Catalog
   +-- Documentation
   +-- Scorecards
   +-- Self-Service Actions
   |
   v
Platform Capabilities
   |
   +-- Git
   +-- CI/CD
   +-- Terraform
   +-- Kubernetes
   +-- Cloud APIs
   +-- GitOps
   +-- Secrets
   +-- Observability
   +-- Security Policy
   +-- FinOps
```
If someone clicks "Create GCP Project" in a portal, the portal did not create the project.

Something behind it did.

Maybe it opened a merge request.

Maybe it triggered a GitLab pipeline.

Maybe Terraform ran.

Maybe Crossplane reconciled a resource.

Maybe an API called another API which called another API because apparently we enjoy these things.

The portal is the interface.

The platform is the machinery underneath it.

That distinction matters because it is entirely possible to build an excellent developer portal without actually building much of a platform.

If all the portal does is give developers nicer links to Jira, Jenkins, Terraform Cloud, Grafana, and Confluence, you have improved discoverability.

That is useful.

It is not the same thing as giving developers a self-service engineering platform.

Before you build the catalog, define your nouns

There is a less exciting part of building an IDP that I think gets skipped far too often.

Before deciding which integrations to install, which scorecards to build, or what the developer portal should look like, the organization needs to agree on what the things inside the platform actually mean.

Start with the word service.

Ask ten engineering teams what a service is and you may get ten slightly different answers.

Is a service:

a Git repository?
something independently deployable?
a Kubernetes workload?
an API?
a business capability?
a collection of workloads owned by the same team?
a scheduled job?
a data pipeline?
a frontend?
a third-party SaaS product your team operates?

This sounds semantic until you start building the data model.

Then it becomes architectural.

Imagine a monorepo containing twelve independently deployed applications.

If repository = service, you have one service.

Operationally, you probably have twelve.

Now take one application deployed into development, staging, and production.

If every runtime instance becomes a service, you suddenly have three services when the engineering organization probably thinks it owns one.

Neither model is inherently wrong.

The problem is not making the decision intentionally.

A useful starting definition might be:

A service is a logical software capability with defined ownership and an independent operational lifecycle, which may be implemented by one or more repositories, workloads, APIs, and infrastructure resources.

Your organization may choose something different.

The exact sentence matters less than agreeing on it.

Once that definition exists, the rest of the model starts getting much cleaner:
```
Team
  |
  v
Service
  |
  +-- Repository
  |
  +-- API
  |
  +-- Pipeline
  |
  +-- Deployment
  |
  +-- Workload
  |
  +-- Infrastructure
  |
  +-- Incident
  |
  +-- Jira Work
  |
  +-- Documentation
```
Now a repository is not a service.

A Kubernetes deployment is not a service.

A cloud project is not a service.

They are things related to a service.

That distinction becomes extremely important once you start building scorecards and automation.

If the data model is wrong, the metrics built on top of it will also be wrong.

Who owns this service?

How many production services do we have?

Which services are deployed through Argo?

What percentage meet our security standard?

Which services have open incidents?

Which teams have configuration drift?

Those questions are only easy to answer if everyone agrees on what the word service means in the first place.

This is why I would spend more time on the data model than most teams initially think is necessary.

The catalog is not just a collection of records.

It is your organization's model of how software actually fits together.

So what actually makes something an IDP?

There is no official checklist, but useful IDPs tend to provide some combination of the same capabilities.

Discovery

Developers can answer questions like:

What services exist?
Who owns this application?
What repository is it in?
Where is it deployed?
What depends on it?
Where are the dashboards?
Is it meeting our engineering standards?

This is where service catalogs are valuable.

Self-service

Developers can request common capabilities without opening an infrastructure ticket.

For example:

Create cloud project
Create storage bucket
Create database
Create repository
Create Kubernetes namespace
Deploy new service
Request certificate
Create DNS record
Provision secrets

The key is that the platform does not just expose raw infrastructure.

It provides an opinionated way to consume it.

Standardization

The platform encodes organizational standards into the path developers use.

A database request might automatically include:

encryption
backup policy
tagging
monitoring
networking
IAM
cost ownership

The developer requests a database.

The platform handles what a database means inside your company.

Delivery

A mature platform usually extends beyond infrastructure provisioning into how applications actually reach production.

That means things like:
```
Source
  |
  v
Build
  |
  v
Test
  |
  v
Artifact
  |
  v
Deploy
  |
  v
Observe
```
The more of that path the platform can make predictable, the more useful it becomes.

There is more than one flavor of IDP

This is where the conversation usually gets messy.

Two companies can both say they have an Internal Developer Platform and have built completely different things.

Neither one is necessarily wrong.

IDPs tend to develop around whatever problem the organization was trying to solve first.

Flavor 1: The developer portal

This is probably the easiest IDP to recognize.

The organization starts with a portal or catalog and builds outward.

Think:
```
Portal
  |
  +-- Services
  +-- Teams
  +-- Documentation
  +-- Dependencies
  +-- Scorecards
  +-- Self-Service Actions
```
Products like Port, Backstage, Cortex, OpsLevel, and others live in this space.

The first win is usually visibility.

Before the portal:

Who owns payment-service?

"I think Sarah's team."

Where is the dashboard?

"Check the README."

Which cluster is it in?

"Prod, probably."

After the portal, that information becomes structured and searchable.

That alone can be worth doing.

But the portal becomes much more interesting when it stops being passive.

Once developers can take actions from it, the portal starts becoming an interface into the platform.

Now "Create Environment" can kick off automation.

"Deploy Service" can trigger a workflow.

"Fix Scorecard Violation" can open a remediation path.

This is where the portal stops being a directory and starts becoming operational.

The failure mode is when the organization stops at the catalog.

A beautifully organized list of services is useful, but nobody has ever shortened deployment lead time by making the links to Grafana easier to find.

Flavor 2: The self-service infrastructure platform

This one usually grows out of Cloud Engineering or Infrastructure Engineering.

The original problem sounds something like:

Developers keep opening tickets for infrastructure and the cloud team is becoming the world's most expensive request queue.

So the platform team starts automating common requests.
```
Developer
    |
    v
"Create Project"
    |
    v
Platform Workflow
    |
    v
Terraform
    |
    v
Cloud
```
The first capabilities are usually predictable:

subscriptions, accounts, and projects
storage
databases
network connectivity
certificates
IAM
DNS
Kubernetes namespaces

This type of platform can provide enormous value very quickly.

If creating a cloud project normally takes three days and the platform reduces it to ten minutes, nobody needs a developer experience survey to understand whether that helped.

The danger is stopping there.

Infrastructure provisioning is only one part of shipping software.

A developer can have a perfectly provisioned cloud project and still spend the next three days figuring out how to build, deploy, secure, and monitor the application that goes into it.

That leads to the next flavor.

Flavor 3: The application delivery platform

This is where the platform starts owning more of the path between source code and production.

Instead of asking only:

How do we let developers provision infrastructure?

the question becomes:

How do we give developers a supported way to ship an application?

Now the IDP starts connecting systems like:
```
Git
 |
 v
CI
 |
 v
Artifact Registry
 |
 v
GitOps
 |
 v
Runtime
 |
 v
Observability
```
The platform may provide:

repository templates
standardized CI pipelines
build containers
artifact repositories
Helm charts
Argo CD applications
deployment strategies
secrets integration
logging
metrics
tracing
health checks

This is also where golden paths become important.

Instead of giving developers twelve tools and documentation explaining how to connect them, the platform gives them a supported path through the tools.

For example:
```
Create Production API
        |
        +-- Repository
        +-- CI Template
        +-- Dockerfile
        +-- Infrastructure
        +-- Helm Chart
        +-- Argo Application
        +-- Logging
        +-- Monitoring
        +-- Ownership Metadata
```
The developer asks for an application.

The platform turns that request into the organization's known-good implementation of one.

That is a much more interesting abstraction than "click here to create a namespace."

Flavor 4: The Kubernetes platform

Some organizations use Kubernetes itself as the primary platform abstraction.

Instead of exposing cloud provider APIs directly, they expose higher-level APIs through Kubernetes.

That might look like:
```
Developer
   |
   v
Platform API
   |
   +-- Kubernetes Resources
   +-- Crossplane
   +-- Operators
   +-- Custom Resources
   |
   v
Cloud + Runtime
```
Now a developer may request infrastructure by creating a Kubernetes object.

Something like:
```yaml
apiVersion: platform.company.io/v1
kind: PostgresDatabase
metadata:
  name: orders-db
spec:
  size: medium
  environment: production
```
What actually happens underneath that object could be complicated.

It might create:

a managed database
network rules
DNS
secrets
monitoring
backup configuration
IAM

The developer does not need to know.

That is the point.

The strength of this approach is that you get a real API and continuous reconciliation.

The danger is that platform teams love abstractions slightly more than developers love consuming them.

You can absolutely build an internal API so clever that developers now have to learn your platform abstraction and the cloud service underneath it.

That is not simplification.

That is two APIs.

A good platform hides complexity the developer does not need.

A bad platform moves complexity somewhere less documented.

Flavor 5: The platform-as-a-product IDP

This one is less about technology and more about how the organization thinks.

A platform team can run Port, Backstage, Kubernetes, Terraform, Argo, Crossplane, and every other fashionable tool in the CNCF landscape and still not operate a platform as a product.

The difference starts with the question being asked.

An infrastructure team asks:

What should we automate next?

A platform product team asks:

What is making it difficult for developers to deliver software?

Those questions can lead to very different backlogs.

Maybe developers do not need another self-service form.

Maybe the biggest problem is that CI takes twenty-eight minutes.

Maybe creating a production service requires seven repositories and three approvals.

Maybe nobody understands how applications are supposed to get into Argo.

Maybe fifty percent of platform requests still need manual intervention after the self-service workflow starts.

That is why platform teams eventually need product metrics.

Things like:

Time to provision

Time to first deployment

Golden path adoption

Self-service completion rate

Manual intervention rate

Deployment lead time

Platform availability

Developer satisfaction

Configuration compliance

Manual tickets avoided

The point is not measuring everything because dashboards are fun.

The point is knowing whether the platform is actually making engineering easier.

Most real platforms are hybrids

This is probably the most important thing to understand.

Very few mature IDPs fit neatly into one category.

A real enterprise implementation might look like this:
```
                    Developer
                        |
                        v
                     Portal
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
     Catalog        Scorecards     Self-Service
                                        |
                                        v
                                      Git
                                        |
                                        v
                                    GitLab CI
                                        |
                           +------------+------------+
                           |                         |
                           v                         v
                       Terraform                  Argo CD
                           |                         |
                    +------+------+                  v
                    |      |      |              Kubernetes
                   GCP   Azure   AWS
```
That is simultaneously:

a developer portal
a cloud provisioning platform
an application delivery platform
a GitOps platform
a governance platform

Trying to decide which one it "really" is misses the point.

The platform exists to provide capabilities.

The architecture should follow the capabilities the organization needs.

The data model may become the most valuable part

There is another reason I think getting the data model right matters more than it first appears.

The catalog you are building is not just a catalog.

Implemented properly, it starts becoming a context engine for your entire software organization.

Think about what eventually gets connected to a service:
```
                         Team
                           |
                           v
Repository ------------> Service <------------- Documentation
   |                       |                         |
   v                       |                         |
Pull Requests              +----> Dependencies      |
   |                       |                         |
   v                       +----> APIs               |
Pipeline                   |                         |
   |                       +----> Incidents <--------+
   v                       |
Deployment ----------------+
   |                       |
   v                       +----> Jira Work
Workload                   |
   |                       +----> Scorecards
   v                       |
Infrastructure ------------+
   |
   +-- Cost
   +-- Security
   +-- Configuration
   +-- Observability
```
At first that graph helps humans.

A developer can open a service and understand who owns it, where the code lives, where it is running, whether it meets engineering standards, what infrastructure supports it, and whether it currently has an operational problem.

That alone is useful.

But the same context becomes much more interesting when you put an AI tool or agent in front of it.

An LLM by itself knows essentially nothing about how your organization operates.

It does not know that payments-api belongs to the Payments team.

It does not know that the repository builds through a particular GitLab pipeline.

It does not know that Argo deployed version 1.8.4 twenty minutes before a production incident started.

It does not know which GCP project hosts the workload, which Terraform workspace manages its infrastructure, which dashboards belong to it, or whether the application currently passes your production-readiness scorecard.

Your IDP can know all of those things.

That changes the types of questions an AI tool can realistically answer.

Instead of:

Explain how Kubernetes readiness probes work.

you can start asking:

Why is payments-api unhealthy?

Or:

What changed immediately before this incident?

Or:

Which production services are currently out of sync in Argo?

Or:

Show me services with Terraform drift that also have an open production incident.

Or:

Which applications are failing our production-readiness standard, and what are they missing?

And eventually:

Create remediation work for every service that is missing the required deployment configuration.

That is a very different use of a developer platform.

The IDP starts as a way for humans to find and consume engineering capabilities.

Over time, its data model can become the semantic layer that lets agents understand those same capabilities.

The relationships are the important part.
```
Service
  |
  +-- owned by --> Team
  |
  +-- built from --> Repository
  |
  +-- deployed by --> Argo Application
  |
  +-- runs as --> Workload
  |
  +-- depends on --> Database
  |
  +-- monitored by --> Dashboard
  |
  +-- affected by --> Incident
  |
  +-- governed by --> Scorecard
```
That is no longer just inventory.

It is a map of how your engineering organization works.

And the quality of that context comes directly back to the decisions we made earlier.

If service means repository in one part of the catalog, Kubernetes deployment in another, and business application somewhere else, the AI does not magically fix the ambiguity.

It consumes it.

A bad data model with an AI interface is still a bad data model.

It just answers confidently and much faster.

This is why concepts like ownership, relationships, naming, and ontology suddenly matter even more in an AI-enabled engineering organization.

The platform team is no longer only designing an interface for developers.

It is defining the context through which machines may eventually understand the engineering estate.

I think this is going to become one of the more important, and probably less obvious, reasons to invest in an IDP.

The portal may be what everyone sees.

The context graph behind it may ultimately be the more valuable asset.

Where golden paths fit

Golden paths deserve their own explanation because they are one of those ideas everyone agrees with right up until you ask what one actually is.

A golden path is the platform team's recommended implementation of a common engineering task.

The important word is recommended.

A golden path should encode the organization's preferred way to solve a problem while removing unnecessary decisions from the developer.

For example, the golden path for a production API might automatically include:

Repository
CI pipeline
Container build
Artifact registry
Infrastructure
Secrets
Argo CD
Observability
Security scanning
Ownership
Cost metadata

The developer still makes the decisions that matter to the application.

The platform makes the decisions that should already be standardized.

That is a healthy abstraction boundary.

A golden path becomes much less useful when it turns into:

Here are forty-seven mandatory fields required to use our simplified platform.

At that point the paved road has speed bumps every six feet.

The best golden paths are usually easier than going around them.

That is what drives adoption.

IDP does not mean no abstraction leaks

One of the more dangerous promises around platform engineering is that developers will somehow stop needing to understand infrastructure.

They will not.

The goal is not to make infrastructure disappear.

The goal is to reduce the amount of infrastructure knowledge required for routine work.

A developer probably should understand:

CPU and memory
scaling
networking basics
secrets
application health
deployment behavior

They probably should not need to understand:

which Terraform backend your organization uses
how your cloud landing zone is assembled
what labels Finance requires
which IAM role the pipeline assumes
how your centralized DNS workflow operates
which logging workspace their application belongs in

Those are platform concerns.

A good IDP creates an intentional boundary between the two.

What an IDP is not

This is probably the easiest way to wrap the definition up.

An Internal Developer Platform is not automatically:

A developer portal.

A portal may be how developers interact with it.

Kubernetes.

Kubernetes may be one of its runtime or API layers.

Terraform.

Terraform may provision infrastructure behind it.

A service catalog.

A catalog tells you what exists. A platform helps you do something with it.

CI/CD templates.

They may be an important golden path, but delivery is only one capability.

A collection of automation scripts.

Automation becomes a platform when it is offered as a consistent, supported capability instead of a folder full of shell scripts everyone is afraid to modify.

And, probably most importantly:

An IDP is not something you buy.

You can buy a lot of the components.

You can buy the portal.

You can buy the catalog.

You can buy workflow engines, CI systems, observability products, security tooling, and cloud services.

But the platform is how those things are assembled around the way your engineering organization actually works.

That part is yours.

So what is an IDP, really?

The easiest way I have found to think about it is this:

The developer portal is the front door. Golden paths are the roads. Automation is the machinery underneath them. The Internal Developer Platform is the system that makes all of it work together.

The technology is important.

But the real objective is reducing the amount of organizational and technical knowledge required to safely ship software.

If a developer can go from:

I need a service

to:

My service is running in production, follows our standards, is observable, has an owner, and can be operated by the team
without opening six tickets and learning the internal structure of the cloud platform, the IDP is doing its job.

And if the same platform can answer:

What is this service, who owns it, how is it deployed, what infrastructure supports it, is it healthy, and does it meet our standards?

then you have built something more valuable than a portal.

You have started building a context layer for the engineering organization itself.

The harder question comes next.

Because building the platform does not mean anyone will use it.

And that is where a lot of IDP programs start to get interesting.

Next in this thread: why Internal Developer Platforms fail, and why the portal is usually not the reason.

References
CNCF TAG App Delivery, Platforms Working Group, "Glossary"
PlatformEngineering.org, platform engineering and Internal Developer Platform guidance
Port documentation, data modeling, relationships, and software catalog concepts