# Shuffle Architecture
Documentation to understand the Shuffle architecture and thoughts behind our choices. Important to understand if you want to contribute or decide whether it works for your organization.

**PS: This document is aimed at on-premises installations, but has some relevant information about cloud too**

## Table of contents
* [Introduction](#introduction)
* [Overview](#architecture_overview)
* [Frameworks](#frameworks)
* [Automation Engine](#automation_engine)
* [Technologies](#technologies)
* [How it works](#how_it_works)
* [Authentication](#authentication)
* [Data access mode](#data_access_model)
* [Encryption and Hashing](#encryption_and_hashing)
* [Backend API access](#backend_api_access)
* [Workflow execution model](#workflow_execution_model)
* [Docker container control](#docker_container_control)
* [Learn more](#learn_more)

## Introduction
With a long-term vision of having an Open(API) ecosystem with a hybrid model between cloud and on-prem, this document will be a guide to understand some underlying aspects of Shuffle and how things fit together. Shuffle does **NOT** require internet to work.

Shuffle Installation models: ![models](https://user-images.githubusercontent.com/5719530/167960847-53e81815-a240-4a26-abb9-96b9244be901.jpeg)

## Architecture overview
The platform is split into two main parts: Server and Workers. The server acts as the host of everything from API activity to Workflow validation, while the Workers are another standalone unit, working in a microservice-esque way. The top and bottom part can be installed on different hosts and be clustered.

![Architecture](https://github.com/frikky/shuffle-docs/blob/master/assets/shuffle_architecture.png?raw=true)

## Frameworks
Shuffle uses and is built upon existing, well established frameworks to help the Security community move forward, rather than just increase complexity. 

* [OpenAPI](https://swagger.io/specification/) - Used to load and generate API specifications that's applicable for other platforms that just Shuffle. This is a standard that's widely used by almost every enterprise, but the security industry are lacking behind.
* [Mitre Att&ck](https://attack.mitre.org/) - Used to create general purpose workflows to make onboarding and usage as smooth as possible. Mitre Att&ck detections are directly proportional to risk, and can be used for KPI's to sell reasons for investment in the platform to leadership etc.
* [CACAO](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=cacao#tc-tools) - A standardized way of handling collaborative automation. Shuffle is built for the masses, and we intend to be as close to the CACAO specification as possible.
* [Cytoscape](https://js.cytoscape.org/) - A light-weight and scalable design system, used for node relationships in our Workflow view. Cytoscape provides us all the necessary parts to create a fully functional workflows for any use. 

## Automation Engine
The Shuffle automation engine is entirely built from scratch, relying heavily on Docker and Micro-service, real-time executions. The code for decision making of next nodes can be found [here](https://github.com/frikky/Shuffle/blob/d77ae8260fd32691d8942dece1957915ba1ff3d5/backend/go-app/walkoff.go#L1251), while the SDK for how Apps perform can be found [here](https://github.com/frikky/Shuffle/blob/master/backend/app_sdk/app_base.py). [Read more here](#workflow_execution_model)

## Technologies
The fundamental building blocks of Shuffle are all designed to be modular Docker images, meaning they can run separately in different environments. The list below contains all the necessary parts to execute a workflow. In case you want to contribute, we've added the programming languages as well.

The reason behind the usage of Golang is simple: Stability. Scripting languages like Python are prone to crashing, while Golang is fun, stable and easy to understand.

| Type | Technology | Note |
| ---- | ---------- | ---- |
| Frontend | ReactJS | Cytoscape graphs & Material design |
| Backend  | Golang | Rest API that connects all the different parts |
| Database | Opensearch | A scalable, NoSQL database used as document store of everything. |
| Orborus  | Golang | Runs workers in a specific environment to connect locations |
| Worker   | Golang | Deploys Apps to run Actions defined in a workflow |
| app sdk  | Python | Used by Apps to talk to the backend |

## Versioning
Shuffle uses [semantic versioning](https://semver.org/). All Docker images can be found on [Github](https://github.com/frikky?tab=packages) or on [Dockerhub](https://hub.docker.com/r/frikky/shuffle/tags).

## How it works
Shuffle is a quite complex platform, with lots of different [features](/docs/features) to handle anything necessary for automation. We are an API-first platform, meaning we always build the API for the feature, before developing the frontend. This allows us to quickly prototype and release new features, without necessarily breaking it. Most of our [API endpoints](/docs/API) use the following model to authentication and authorize whether the user has access:

1. User clicks a button in the frontend
2. Backend receives a HTTP request with a Session Token
3. The backend validates it the request in this order:
	1. CORS
	2. [Authenticate](#authentication): Are they logged in?
	3. Authorize: Do they have access to the resource? 
4. If valid, the backend processes the request by accessing info in this order:
	1. Is there cache for the resource? 
	2. Does the resource exist in the database? Request is authenticated in the database, before a return. 
5. The data gathered is processed, before Shuffle returns either of these codes (with certain exceptions):
	- 200: Everything worked!
	- 301: Redirect to a different page to fix the issue.
	- 401: Authentication and Authorization issues.
	- 500: Internal error. Please [contact us](https://shuffler.io/contact) if this occurs.

	If there's a failure, Shuffle should ALWAYS return with data in this format:
	```
	{"success": False, "reason": "Here's the reason it didn't work"}
	```
6. The frontend takes the data and shows it in the UI.

## Authentication
There are two types of authentication tokens in Shuffle: API access and App authentication. 

### Data access model
Accessing data is done PER ORGANIZATION and per user. This means that you need to be a part of the organization you're querying data from. Pre RBAC: If you are an admin, you have access to all information within an Organization, while users only have access to their own resources, and the apps that are enabled.

### Encryption and hashing
Hashed (bcrypt):
- User passwords.

Encrypted (AES-256):
- [App authentication](#app_authentication) and [Files](#files) are being encrypted. The seed used for hashing is random for each organization, and can be set with the environment variable SHUFFLE_ENCRYPTION_MODIFIER in the local version of Shuffle. This is automatically handled in our SaaS offering. How it works: 

	1. Create md5 hash from Org ID + Workflow_id + Auth timestamp + SHUFFLE_ENCRYPTION_MODIFIER
	2. Encrypt the authentication value with [aes.NewCipher](https://cs.opensource.google/go/go/+/go1.17.1:src/crypto/aes/cipher.go;l=32)
	3. Base64 encode the encrypted value (because bytes and strings aren't friends)
	4. Store the value in the database.

```
	Run in reverse to decrypt and retrieve the values.
```

**Encryption Code reference** 

- [Encrypt](https://github.com/Shuffle/shuffle-shared/blob/d5e67ed2cefb5e94f3c516bdd3030384f6241754/shared.go#L11196)
- [Decrypt](https://github.com/Shuffle/shuffle-shared/blob/d5e67ed2cefb5e94f3c516bdd3030384f6241754/shared.go#L11228)


### Backend API access
There are multiple ways to access the API. The first is through the UI and a logged in user. The second is through the API directly with a Bearer token. The third is from a workflow execution.  

- Session Token: Defined in a users' browser as user logs in. 
- Bearer Auth: This is a token provided to each user to be used with the [API](/docs/API)
- Execution token: The execution token is a unique token (UUID) provided to each [workflow execution](#workflow_execution_model). As soon as an execution is triggered, it gives a temporary token which is valid as long as the workflow is running, but which is no longer valid after the workflow finishes. This has access to certain API's used by Apps themselves (files, cache, setting/changing the execution), with most other API's being off limit. 

### App authentication
App authentication is how we authenticate and store an apps' configuration. If an app requires authentication, and the user adds them, these will be encrypted and stored in the database, along with being cached in their encrypted form (AES-256). These values can and should NEVER be decrypted to be seen by a process or human other than during a workflow execution.

How are these values being used then? If they're encrypted, how does the app get access to them? Here's how:

1. The [workflow starts running](#workflow_execution_model). This sets up a lot of different values necessary for Shuffle to find the start node, next nodes, parsing data, fixing conditions etc.
2. The backend looks at ALL nodes in the workflow, checking if any of their parameters have the field "configuration" set to true. This indicates it's a field to be replaced.
3. It finds the appropriate apps' chosen authentication by ID (authentication_id)
4. Shuffle runs [decryption on all](#encryption_and_hashing) the appropriate fields, then replaces the value JUST for this workflow execution.
5. The workflow starts with the right values. These values are and should NOT be available to a user reading the workflow at any point. 
6. When the app and workflow is finished - all these values are cleaned up and removed from the execution to ensure they're not stored.

## Workflow execution model
The execution model of Shuffle can be defined as such:

1. A Workflow is triggered. This can be by a trigger, or a manual execution, making a request towards /api/v1/workflows/<workflow_id>/execute
2. The backend fills in the appropriate gaps (e.g. startnode, source trigger or [app authentication](#app_authentication)), before adding an execution to be retrieved by Orborus at a later stage, with the appropriate priority (0-10, 10 being highest).
3. Orborus runs as an agent, constantly polling for jobs from Shuffle. By default, it retrieves 10 jobs at a time, before checking whether any can be started. This is done based on how many Workers are running in Docker at this time, compared to the environment variable SHUFFLE_ORBORUS_EXECUTION_CONCURRENCY.
	- Orborus also cleans up Docker containers on an interval according to the environment variable SHUFFLE_ORBORUS_EXECUTION_TIMEOUT
4. For each of the new jobs, Orborus creates a container for a Worker with the following information:	An execution ID and authorization, and whether to run in an optimized way.
5. A worker is started FOR EACH EXECUTION, which is in control of the entire duration of an execution. It has two modes:

  1. Unoptimized: If a workflow is started manually, the Worker will periodically poll the backend for updates, pointing all apps to send direct information to the backend. This makes it possible for the user to see updates in real-time for debug purposes.
  2. Optimized: If a Workflow is started from a trigger (not manually), the Worker starts an HTTP server, and acts as a temporary backend for this specific execution. This makes it possible to communicate and deploy Apps faster, without straining the backend. When the workflow is finished, it will send the full execution to the backend. This means the frontend MAY not have the full picture until after the workflow execution finishes.

6. The Worker attempts starting each App's Docker container, starting with the startnode. As it finds that a node has finished, it will check it's status, before starting the following nodes upon success. If the App's Docker image doesn't exist, it will attempt to download in the order of: Backend, Dockerhub. If it doesn't exist, abort the workflow.
7. The apps that were started by a Worker retrieves the full execution in order to be able to [identify variables](/docs/workflows#workflow_variables), [check conditions](/docs/workflows#conditions), authorization, [download files](/docs/features#file_storage) or from the cache etc. Order of operations in an App's Docker container:

	1. Download the workflow execution and current action from the backend.
	2. Check if ALL [Conditions](/docs/workflows#conditions) are valid or not. 
	3. Check each parameter, and whether they contain data that needs replacing ($nodename.json.is.#0.parsed.like_this). 
	4. Check Liquid formatting, and replace values with it.
	5. Validate based on the parameters, if it needs to run ONE TIME or MULTIPLE TIMES (for loop). 
	6. Run the App! Now that we've parsed all the data, we can finally run the actual app image.
	7. If error from the app, return to the Backend with status "FAILURE" and details on the issue. If success, return status "SUCCESS" and the value of the execution. Logs and timestamps are also added for this. 
	8. Exit the Docker container, and self-delete information IF running an optimized Worker.

8. When the Execution in the Worker is in either the FINISHED or ABORTED state, the Worker sends information back to the original Backend about the status of the execution.

To learn about the code behind the execution, check [here](https://github.com/frikky/Shuffle/blob/d77ae8260fd32691d8942dece1957915ba1ff3d5/backend/go-app/walkoff.go#L1251)

## Docker container control
As we've hinted at more than a few times, everything in Shuffle is built around Docker containers. This keeps the environment safe even in cases of compromise, and is necessary to allow a user to build an app for pretty much anything. This allows us to sandbox and control how actions are performed, as it's all built on top of our App SDK. On top of all this? It makes sharing and deploying easier between Shuffle users, which is paramount to a scalable, global system.

The most important parts about containerization in Shuffle is this: 
- Our App SDK is a Docker container
- If an app is made in Shuffle, either from OpenAPI or from scratch, it's built on top of the App SDK. 
- All apps made by and for Shuffle will be released, unless explicitly decided not to.
- [All apps](https://github.com/frikky/shuffle-apps) are built around the App SDK
- If an app doesn't exist on the Worker's server during an execution, the Worker in question will try to download it from local, then remote registries.

## Learn more
[Contact us](https://shuffler.io/contact) or mail us at [help@shuffler.io](mailto:help@shuffler.io), and we'll provide you with the necessary information.
