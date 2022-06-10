- Start Date: 08.06.1989
- RFC PR: https://github.com/suse-skyscraper/rfc/pull/2

# Skyscraper Initial Design

## Summary

We would like to create a single interface for teams to manage our cloud governance. Currently, even though we manage everything by code, the complexity of multiple cloud providers adds unnecessary work to our team. We hope to make it possible for everyone, including non-technical employees, to be able to own parts of the process, so we can work together to build a better business in the cloud.

## Motivation

The main motivation behind this project is to simplify the process of cloud enablement and maintenance. Even though we work with the different cloud providers, we want to ensure consistent standards, such as tags. These standards help others know information such as who owns a workload, who pays for it, and who to contact when something goes wrong.

Currently, the burden of those standards lies on one team, but the team aren't subject-matter experts for some processes and are unnecessarily in the loop. Instead of creating more work for the team, we can empower the other teams to own the processes that matter to them.

This project will be gradual and initially specific in scope, but we hope to build upon it over time to meet our future business needs.

## Design Detail

### Technical Goals

* The project should be designed as a Kubernetes workload.
* When possible, build upon open-source projects in foundations such as the [Cloud Native Computing Foundation](https://www.cncf.io/).
* We should design with enterprise authentication needs in mind:
  * Support login with Okta.
  * Support SCIM with Okta to push users/groups.
  * IAM based on groups.
* The user interface should offer a best-in-class experience for our users.
  * A PWA with asynchronous communication with our backend.
* Audit all change requests.
* Repository pipelines and dependency monitory:
  * [GitHub Actions](https://github.com/features/actions) for CI and publishing artifacts to [GitHub Packages](https://github.com/features/packages).
  * [Dependabot](https://github.com/dependabot) to manage all dependencies.
  * [Helm Chart Releaser](https://github.com/marketplace/actions/helm-chart-releaser) for publishing helm charts.
* Easy development experience:
  * Use [Skaffold](https://skaffold.dev/) to deploy development artifacts to a local Kubernetes cluster.

### Starting Processes

**Tag Management:**

* Allow users to change tags in the interface, which then would trigger a background job to change the tags in the cloud:
  * AWS account level tags
  * Azure subscription tags
  * GCP project tags
* Have workloads to continuously validate the tags, sending alerts with validation issues.
* Display cloud tags in the UI.
* Export cloud tags to CSV and Excel.

**Cloud Accounts Management:**

* Allow users to request new cloud accounts in the interface.
* Validate user input to ensure that the request data is correct.
* Display all cloud accounts in the UI.
* Export all cloud accounts to CSV and Excel.

**Finance Features:**

* Control Purchase Orders (POs) and Cost Centers (CCs) for our cloud environments.
* Control the life-cycle of POs and send alerts when one is about to expire.
* Display POs and CCs for our finance teams.

### The Parts (Overview)

* The web frontend: a "Progressive Web Application" (PWA) written in [Angular](https://angular.io/)
* The web backend: a server written in [Go](https://go.dev/)
* The backend jobs: [NATS](https://nats.io/)
* Database: [PostgreSQL](https://www.postgresql.org/)
* Deployment: [Helm Charts](https://helm.sh/)

### The Web Frontend

The web frontend is the main interaction point for our users. It should be secure and easy-to-use, hiding the complex business logic away in the web backend.

**Authentication:**

The user cannot do anything of value on the website until it authenticates with Okta. To do this, we use Okta's official [Okta Angular SDK](https://github.com/okta/okta-angular) to establish a secure session.

Once logged in, we can get basic data from Okta, such as a user's groups, name, email address, and "JSON Web Tokens" (JWT). We can use that JWT, and send it to our backend as an identifier of the user. This allows us to essentially log into our web frontend and backend at the same time.

**IAM:**

The web frontend doesn't verify any IAM permissions, that is the responsibility of the web backend. This is because the only place we can trust isn't manipulated by the user is our server under our control.

It can, however, display IAM information like users, permissions, and groups. It is a matter of discussion whether configuring IAM should be done through the frontend or as a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) in Kubernetes.

**Data Entry:**

The web frontend allows data entry to fulfill process. The job of the web frontend is to present this data-entry process in an easy-to-use way. When a user finishes a process, the frontend then should send the data to the backend.

As these forms are asynchronous, we don't need to wait for a workload to finish before we send the next page to the user. Instead, we should provide visual feedback to the state of the task, such as "PENDING", "IN_PROGRESS", "FAILED", and "COMPLETE". We'll need to look into how we accomplish this feature at a later point.

### The Web Backend

**Authentication:**

As the user indirectly uses this backend, we need to rely on the JWT coming from the frontend. We can verify our user by using the project [Okta JWT Verifier Golang](https://github.com/okta/okta-jwt-verifier-golang) by Okta.

Even though we get access to our user and its groups, we want to use SCIM to manage our groups. This allows Okta to push groups and their user memberships to our application. We can save those memberships, use them in our IAM processes, and always have an up-to-date picture of group memberships in our application.

**IAM:**

We'll need to create the ability to assign permissions to groups. These permissions shouldn't just be per-process, instead, we should focus on per-resource. This is because in a company, membership is complex, and one department might not want another to edit its data.

**Data Entry:**

The web frontend communicates with the backend through an API to accept a change request from a user. As we don't want to block a user's browser while waiting for completion, we need to push the job to a backend service.

The web backend can work on those background tasks in a separate process, and dynamically let a user know of its completion.

### Background Jobs

Handling backend jobs can be a complicated prospect. We have many options, and they all have their strengths and weaknesses.

**Limitations:**

* We don't want to build a background job system from scratch.
* We don't want to lock ourselves to a specific cloud provider's implementation.
* It should be easy to use and set up.
* It should be a well-supported tool.
* It should be open source.

When looking at those limitations, we saw the potential to try [NATS](https://nats.io/), an incubating project in the CNCF.

NATS has many features, but we're interested in its messaging services with [JetStream](https://docs.nats.io/nats-concepts/jetstream). With JetStream, we can ensure that our jobs are received and tasks complete.

The workflow would be the following:

* The frontend send a job to the backend.
* The backend validates the job and sends it to NATS.
* NATS sends the job to a backend worker.
  * If there are no backend workers, wait for one to come up.
  * If the task fails, send it again to another worker.
* The backend worker fulfils the job and tells NATS that the job is complete.

### Database

Data will be an essential part of our application and should be considered a first class citizen. Because of this, we need to have a resilient database with low maintenance effort.

**Limitations:**

* We don't want to build a database from scratch.
* We don't want to lock ourselves to a specific cloud provider's implementation.
* It should be easy to use and set up.
* It should be a well-supported tool.
* It should be open source.
* (Optional) Because we're targeting a cloud environment like AWS, use either MySQL or PostgreSQL.

**Wishes:**

* Native JSON data types
* Relational Database (our data will be relational)

Either MySQL or PostgreSQL would be sufficient and the differences can be abstracted away with ORM. We'll start with PostgreSQL, and if we later want, add MySQL support as well.

### Helm Charts

Helm charts are a standardized way to release Kubernetes deployment artifacts, so we should support them from the start.

Helm's [Helm Chart Releaser](https://github.com/marketplace/actions/helm-chart-releaser) can help us deploy and publish our charts.

### Starting Database Tables

**users:**

* id: int
* email: string
* name: string
* PRIMARY_KEY(id)

**groups:**

* id: int
* name: string
* PRIMARY_KEY(id)

**user_group_membership:**

* id: int
* user_id: string
* group_id: string
* PRIMARY_KEY(id)
* FOREIGN_KEY(user_id) -> user
* FOREIGN_KEY(group_id) -> group

**cloud_tenants:**

* cloud_provider: string
* tenant_id: string
* name: string
* active: boolean
* created_at: timestamp
* last_updated_at: timestamp
* PRIMARY_KEY(cloud_provider, tenant_id)

**cloud_account_metadata:**

* cloud_provider: string
* tenant_id: string
* account_id: string
* name: string
* tags_current: json
* tags_desired: json
* tags_drift_detected: true
* active: boolean
* created_at: timestamp
* last_updated_at: timestamp
* PRIMARY_KEY(cloud_provider, tenant_id, account_id)
* FOREIGN_KEY(cloud_provider, tenant_id) -> cloud_tenant

**cloud_account_metadata_jobs:**

* cloud_provider: string
* tenant_id: string
* account_id: string
* tags_current: json
* tags_desired: json
* status: string
* initiator_user_id: int
* PRIMARY_KEY(cloud_provider, tenant_id, account_id) -> cloud_account_metadata
* FOREIGN_KEY(cloud_provider, tenant_id) -> cloud_tenant
* FOREIGN_KEY(initiator_user_id) -> user

**cloud_account_groups:**

* id: int
* name: string
* created_at: timestamp
* last_updated_at: timestamp

**cloud_account_groups_ownership:**

* cloud_provider: string
* tenant_id: string
* account_id: string
* cloud_account_group_id: string
* created_at: timestamp
* last_updated_at: timestamp
* PRIMARY_KEY(cloud_provider, tenant_id, account_id)
* FOREIGN_KEY(cloud_provider, tenant_id) -> cloud_tenant
* FOREIGN_KEY(cloud_account_group_id) -> cloud_account_group

### API

GET /api/v1/profile

* id: int
* email: string
* name: string

GET /api/v1/users

* list
  * id: int
  * email: string
  * name: string

GET /api/v1/user/{id}

* id: int
* email: string
* name: string

GET /api/v1/user/{id}/groups

* list
  * id: int
  * name: string

GET /api/v1/groups

* list
  * id: int
  * name: string

GET /api/v1/groups/{id}

* id: int
* name: string

GET /api/v1/groups/{id}/users

* list
  * id: int
  * name: string
  * email: string

GET /api/v1/cloud_tenants

* list
  * cloud_provider: string
  * tenant_id: string
  * name: string
  * active: boolean
  * created_at: timestamp
  * last_updated_at: timestamp

GET /api/v1/cloud_tenants/{id}

* cloud_provider: string
* tenant_id: string
* name: string
* active: boolean
* created_at: timestamp
* last_updated_at: timestamp

GET /api/v1/cloud_tenants/{cloud_tenant_id}/accounts

* list
  * cloud_provider: string
  * tenant_id: string
  * account_id: string
  * name: string
  * tags_current: json
  * tags_desired: json
  * tags_drift_detected: true
  * active: boolean
  * created_at: timestamp
  * last_updated_at: timestamp

GET /api/v1/cloud_tenants/{cloud_tenant_id}/accounts/{id}

* cloud_provider: string
* tenant_id: string
* account_id: string
* name: string
* tags_current: json
* tags_desired: json
* tags_drift_detected: true
* active: boolean
* created_at: timestamp
* last_updated_at: timestamp

PUT /api/v1/cloud_tenants/{cloud_tenant_id}/accounts/{id}

* cloud_provider: string
* tenant_id: string
* account_id: string
* tags_desired: json
* active: boolean

GET /api/v1/cloud_tenants/{cloud_tenant_id}/accounts/{id}/jobs

* cloud_provider: string
* tenant_id: string
* account_id: string
* tags_current: json
* tags_desired: json
* status: string
* initiator_user_id: int

### Process 1: Sync Tags

1. List all tenants
   1. Get the cloud credential from the application configuration.
   2. With the cloud credential, iterate through the cloud and gather metadata on all accounts, projects, or subscriptions.
   3. Find the cloud account in our database, if it cannot be found, create a new object with empty metadata ("{}" for `tags_desired`).
   4. Save the metadata in our database to the table `cloud_account_metadata`.
      1. If our `tags_current` don't match `tags_desired`, set `tags_drift_detected` to true.

### Process 2: Set Tags

1. User updates tags in the frontend ui.
2. The frontend sends an api request to the backend api to update `cloud_account_metadata`.
3. The backend accepts the job and sends it to a worker.
   1. Set the field `tags_desired` of `cloud_account_metadata` to the user's input.
   2. Create a `cloud_account_metadata_jobs` object.
      1. Set `status` to `PENDING`.
      2. Set `initiator_user_id` to the user.
      3. Set `tags_desired` and `tags_desired` to keep track of changes.
4. A worker picks up the task and changes the tags.
   1. Set the `status` field of `cloud_account_metadata_jobs` to `COMPLETE`.
   2. Set the field `tags_current` of `cloud_account_metadata` to changed tags.
   3. Set the field `tags_drift_detected` of `cloud_account_metadata` to false.

## Drawbacks

* We'll need to spend engineering time to build and maintain the tool.
* We'll need to spend engineering time to build and maintain the infrastructure around the tool.

## Rationale and Alternatives

While other tools might offer tools tag management, we're looking to build tools around our business processes.

Our data and processes are important to us as a company, so we need a tool which, empowers our users to make changes, but also protects and audits these actions.

If we don't take up this task, the CCoE team will be the central point of failure in cloud enablement and management, while also wasting critical engineering time on data entry.

## Prior Art

The CCoE team previously created the project [CCoE Account Metadata](https://github.com/SUSE/ccoe-account-metadata), our first attempt to centralize our data.

In that project, we pushed our cloud metadata to AWS DynamoDB via terraform. While it allowed us run workloads based on our metadata, its reliance on terraform manage data wouldn't scale to other teams well.

As our cloud usage accelerated, so did our workload, just to manage our metadata.

**Feature Set:**

* Collected metadata for AWS, Azure, and GCP.
* Exported our metadata to SharePoint as CSV and Excel files to allow other teams to see our data.
* We started crawling our AWS and Azure accounts, so we could detect drift.

## Unresolved questions

* How do we allow others to add custom functionality around Skyscraper?
* How do we manage IAM?
* How do we sign our artifacts for supply chain security?
