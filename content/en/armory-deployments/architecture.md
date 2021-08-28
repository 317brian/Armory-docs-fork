---
title: "Armory Deployments Architecture"
linkTitle: "Architecture"
exclude_search: true
weight: 10
---

## Key Components

### Armory Deployments Plugin 

The Armory Deployments Plugin for Armory Enterprise (Spinnaker™) provides the Kubernetes Progressive Stage. This plugin connects to Armory Cloud services, which are hosted and managed by Armory, outside of your Armory Enterprise (Spinnaker) installation. These cloud services do the following: 

- Provide information about what Kubernetes Accounts are available as deployment targets
- Execute deployments

The plugin enables you to use a single stage to perform a progressive deployment strategy. This strategy allows you to deploy a new version of your application, and route traffic to the new version incrementally. In between each scaling event, the stage can be configured to wait for an event, such as a manual approval, before continuing.

For information about enabling the stage, see [Get Started With Armory Deployments for Spinnaker]({{< ref "armory-deployments-for-spinnaker" >}}).

### Armory Cloud Agent

The Armory Cloud Agent allows Armory Cloud Services to interact with your Kubernetes clusters and orchestrate deployments without direct network access to your clusters. The Agent gets installed in every deployment target and connects those clusters to the Agent Hub in Armory Cloud. The connections are encrypted long-lived gRPC HTTP2 connections. The connections are used for bidirectional communication between Armory Cloud Services and Armory Cloud Agents. The Agent issues API calls to your Kubernetes Cluster based on requests from Armory Cloud.

### Armory Cloud

Armory Cloud is a collection of cloud-based services that Armory operates. These services are used to provide capabilities beyond those that are native to Spinnaker. These services are provided as a cloud service to minimize operational complexity.

Several specific services in Armory Cloud are important for understanding how Armory Deployments functions. These services have endpoints with which users and non-cloud services interact. Details of the external URLs for these services are covered in the [Networking](#networking).

#### Agent Hub

Agent Hub routes deployment commands to Armory Cloud Agents and caches data received from them. Agent Hub does not require direct network access to the agents since they connect to Agent Hub through an encrypted long-lived gRPC HTTP2 connection. Agent Hub uses this connection to send deployment commands to the Agent for execution.

#### OIDC auth service

The Open ID Connect (OIDC) service is used to authorize and authenticate machines and users. The Armory Cloud Agent, Armory Enterprise (Spinnaker) plugin, and other services all authenticate against this endpoint. The service provides an identity token that can be passed to the Armory API and Agent Hub.

#### Rest API

This endpoint receives API calls from clients outside of Armory Cloud (such as the Deployments Plugin). Clients connect to these APIs to interact with Armory Deployments.

#### Cloud Console

The Armory Cloud Console provides a UI to configure authentication and authorization and to create tokens.

#### Temporary Deployments API

The current Armory Enterprise (Spinnaker) plugin uses a temporary API to invoke Armory Deployments. These API calls are in the process of being migrated to the REST API to simplify the overall architecture. This temporary API is expected to be removed during 2021.

### Argo Rollouts

Armory Deployments for Kubernetes uses the [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) controller in each target Kubernetes cluster to enable various deployment strategies.

## Architecture

Armory Deployments contains components that you manage and components that Armory manages in the cloud. The components you manage allow Armory’s cloud services to integrate with your existing infrastructure.

{{< figure src="/images/armory-deploy-architecture/armory-deploy-k8s-overview.jpeg" alt="The Armory command line interface and its integrations connect to Armory Cloud. Armory Cloud uses the Agent Hub to connect to your Kubernetes cluster using a gRPC connection established between the Agent Hub and Armory Cloud Agent, which is installed in your cluster." >}}

You connect your Kubernetes clusters to Armory Cloud by installing the Armory Cloud Agent. The Agent version is installed separately from your Spinnaker cluster and does not directly talk to Spinnaker. The Agent establishes a bidirectional link with Armory Hub. Armory Hub uses this link to route communication from services within Armory Cloud to the Agent in your Kubernetes cluster. The Agent enables Armory Cloud to act as a control plane for your infrastructure.

### How it works

Armory Deployments uses [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) to power deployments in Kubernetes clusters. The Argo Rollouts controller is a [Kubernetes controller](https://kubernetes.io/docs/concepts/architecture/controller/) and set of [Custom Resource Definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

{{< figure src="/images/armory-deploy-architecture/armory-deploy-argo-overview.jpeg" alt="In your Kubernetes cluster, the Armory Agent enables communication with Armory Cloud services through the Agent Hub. The Argo Rollout controller performs the deployments in the Kubernetes cluster." >}}

When you start a deployment, Armory Deployments processes your deployment request and generates [Argo Rollout](https://argoproj.github.io/argo-rollouts/) manifest(s) to execute the deployment. Armory Deployments then triggers Kubernetes infrastructure changes using Armory Cloud’s bidirectional link with the Agent.  The Agent creates the generated CRDs in your Kubernetes cluster to trigger actions from Argo. Users do not need to create or manage the Argo Rollout CRDs. Armory Deployments manages these automatically.

You can track the status of a deployment in the Kubernetes Progressive stage for Spinnaker. This stage reaches out to Armory Cloud to determine the current status of the deployment.

## Security

### Armory Cloud Identity and Access Management (IAM) 

Armory Cloud uses OIDC to authenticate both user and machine principals and issue short-lived access tokens, which are signed JSON web tokens (JWTs).

The Armory Cloud API consumes these access tokens in order to validate that a request has authorization for a given tenant’s resources and operations.

Use the the [Armory Cloud Console](https://console.cloud.armory.io/) to manage the following:

- Ceate credentials for machines and scope them for specific permissions and use cases.
- Invite and manage users
- Enable OIDC based external identity providers (IdP), such as Okta, Auth0, or OneLogin.

The following concepts can help you when configuring access in the Cloud Console:

- **Organization**
  
  A company like “Acme” or “Armory.”

- **Environment**

  A collection of accounts and their associated resources that you explicitly define. Environments are useful for separation and isolation, such as when you want to have distinct non-production and production environments. Accounts added to one environment are not accessible by machine credentials scoped to another environment.

- **Tenant**

  The combination of an Organization and Environment.

- **Principals**

  There are two types of principals:  a Machine or a User.

  *Machine principals* are credentials that are created within the Armory Cloud Console with specific scopes. They are used by service accounts, such as for allowing Spinnaker to connect to Armory Cloud.

  *User principals* are users that are invited to an organization, either by being invited or logging in through a configured external (IdP) such as Okta or Auth0.

- **Scopes**

  Scopes are individual permissions that grant access to certain actions. They can be assigned to Machine to Machine credentials. For example, the scope `read:infrastructure:data` allows a machine credential to fetch cached data about infrastructure and list accounts that are registered with the Agent.

- **Groups**

  Groups are attached to user principals and sourced from an external IdP like Okta or LDAP. Note that they are not currently being used by the Armory Cloud API.

### Networking

All network traffic is secured and encrypted with TLS end-to-end.
The following network endpoints are used for communication into Armory Cloud:

| DNS                    | Port | Protocol                                         | Description                                 |
|------------------------|------|------------------------------------------------------|----------------------------------------------------------------------------|
| agents.cloud.armory.io | 443  | TLS enabled gRPC over HTTP/2<br>TLS version 1.2    | Agent Hub connection. gRPC is used to provide bidirectional, on demand communication. |
| api.cloud.armory.io    | 443  | HTTP over TLS (HTTPS)<br>TLS version 1.2 | Armory REST API     |
| auth.cloud.armory.io             | 443  | HTTP over TLS (HTTPS)<br>TLS version 1.2          | OIDC Service
| console.cloud.armory.io          | 443  | HTTP over TLS (HTTPS)<br>TLS version 1.2           | Web UI    |
| grpc.deployments.cloud.armory.io | 443  | TLS enabled gRPC over HTTP/2<br>TLS version 1.2             | Temporary Deployments API. This API will be replaced with new calls in the REST API in the future. |

### Data encryption

#### Encryption in transit

All data is encrypted in transit between Clients and Armory Cloud. 

Encryption in transit is over HTTPS using TLS encryption. When using Armory-provided software for both the client and server, these connections are secured by TLS 1.2. Certain APIs support older TLS versions for clients that do not support 1.2.

#### Encryption at Rest

Encryption at rest uses AES256 encryption.