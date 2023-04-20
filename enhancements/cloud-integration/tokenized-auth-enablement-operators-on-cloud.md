---
title: tokenized-auth-enablement-operators-on-cloud
authors:
  - "@bentito"
  - "@jharrington22"
  - "@gallettilance"
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - "@joelanford" # for OLM, OperatorHub changes
  - "@bparees" # for OLM, OperatorHub changes
  - "@csrwng" # for HyperShift changes
  - "@spadgett" # for Console changes
  - "@abutcher" # for CCO changes
approvers:
  - "@sdodson"
  - "@joelanford"
  - "@bparees"
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
  - "@JoelSpeed " # for CredentialsRequest changes
creation-date: 2023-01-21
last-updated: 2023-04-19
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - https://issues.redhat.com/browse/OCPBU-4
  - https://issues.redhat.com/browse/PORTENABLE-471
  - https://issues.redhat.com/browse/CCO-286
see-also:
  - []
replaces:
  - []
superseded-by:
  - []
---

# Tokenized Authentication for Red Hat Operators on Cloud Providers

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Many Cloud Providers offer services that allow authentication via Temporary Access Tokens (TAT).

For example, Amazon Web Services (AWS) Secure Token Service (STS) allows you to request temporary limited-privilege
credentials. A concrete example is write permission on an S3 bucket on AWS. STS is supported on all the major cloud
providers where OpenShift is deployed.

Currently, several Red Hat operators provide detailed instructions for how to use the operator on clusters where CCO is
in Manual Mode. The process is designed to have manual operations performed by the customer that are inconsistent across
layered product operators.

This enhancement seeks to bring a unified set of standards and tooling to the process of enabling an
operator to access cloud provider resources using TAT, and improve the UX for consumers of these operators. 

## Motivation

Several Red Hat operators have normal operating modes where they assure their operands have required cloud 
resources to perform properly. In Mint Mode, CCO will automatically create the IAM roles and credentials required for 
the operator to assume the role and authenticate with the Cloud Provider.

In Manual mode or more generally on clusters that support TAT (CCO not necessarily installed), an admin installing an 
operator is required to:

Extract the CredentialsRequest from the operator's image or codebase in order to know what IAM role is appropriate for 
the operator to assume
xIf using the Cloud Credential Utility, `ccoclt`:
1. Find and download a compatible `ccoclt` binary
2. Accept that `ccoclt` is a binary you downloaded that has god-mode access to your Cloud Provider
3. Use `ccoclt` to create the IAM Role
4. Use `ccoclt` to create the secret with the credentials expected by the operator in mint mode

Here is a diagram for how this works on AWS:

![](STS_today.jpg)

This enhancement seeks to unify this process across operators so users of several of them have the same experience and
similar steps to perform. We aim to automate and reduce those steps as much as possible to make installing operators
(via OLM) simpler on clusters where TAT authentication is supported.

### User Stories

* As a cluster admin, I want to know which OLM Operators are safe to install because they will not be interacting with
  the Cloud Provider on a cluster that only supports TAT authentication with the Cloud Provider
* As a cluster admin, I want to know which OLM Operators support tokenized authentication for my cloud, so that I can
  provide token-based access for cloud resources for them.
* As a cluster admin of a cluster using tokenized cloud auth, I want to know what's required to install and upgrade OLM 
 Operators whenever those operators manage resources that authenticate against my cloud so they can function properly
* As a cluster admin, I want the experience of TAT authentication to be as similar as possible from one Cloud Provider 
  to the other, so that I can minimize Cloud Specific knowledge and focus more on OpenShift.
* As an Operator developer, I want to have a standard framework to define tokenized authentication requirements and
  consume them, per supported cloud, so that my operator will work on token-based authentication clusters.
* As an Operator Hub browser, I want to know which operators support tokenized cloud auth and on which clouds so I can
  see only a filtered list of operators that will work on the given cluster.
* As an Operator Hub browser, I want to be informed / reminded in the UI that the cluster only supports TAT 
  authentication with the Cloud Provider, so that I don't confuse the cluster with one that will try to mint long lived
  credentials.
* As an Operator Hub browser, I want to be able to easily provide what's required to the OLM operators I install through
  the UI.
* As the HyperShift team, where CCO is not installed so the only supported authentication mode is via TAT, I want the
  Red Hat branded operators that must reach the Cloud Provider API, to be enabled to work with TAT credentials in a
  consistent, and automated fashion so that customer can use those operators as easily as possible, driving the use of
  layered products.

### Goals

Allow Day-2 Operators to access cloud provider resources as seamlessly as possible when being installed, used and
updated on STS enabled clusters.

While providing the above goal, allow for multi-tenancy. In this sense multi-tenancy means that an operator may
enable its operands to communicate with the cloud provider instead of communicating with the cloud itself. Operands 
would use the same set of credentials as the operator. The operator will need to maintain its own logic to minimize 
conflicts when sharing credentials with operands.

Operator authors have a way to notify, guide, and assist Day-2 Operator admins in providing the required cloud provider
credentials matched to their permission needs for install and update.

Ideally, a solution here will work in both HyperShift (TAT always) and non-HyperShift clusters.

### Non-Goals

Day 1 operators are not included in this proposal, they are pre-provisioned for TAT enabled clusters (at least 
for HyperShift)

Bring Your Own Credentials (BYOC) where an operator can manage and distribute credentials to its operands are out of
scope.


## Proposal

**Cloud Credential Operator (CCO) changes**: Adds a token-aware mode, while nominally set in "Manual" mode, CCO will
look for and process CredentialRequests referenced in Operator CRs when:

```bash
# infrastructure platform is AWS
$ oc get infrastructure cluster -o jsonpath --template '{ .status.platform }'
AWS

# credentialsMode is "Manual"
$ oc get cloudcredential cluster -o jsonpath --template '{ .spec.credentialsMode }'
Manual

# serviceAccountIssuer is non empty
$ oc get authentication cluster -o jsonpath --template='{ .spec.serviceAccountIssuer }'
abutcher-oidc.s3.us-east-1.amazonaws.com
```

This will operate in one of two ways depending on which other parts of this EP for other components are adopted: 
specifically HyperShift/Rosa adding CCO with these token-aware changes or continuing with the current support for the 
pod identity webhook.

CCO's token-aware mode will work by adding, for example for STS, a "role-arn"-type field on operator added 
CredentialsRequest objects. This is a new API field on the CredentialsRequest spec. When CCO acquires a 
CredentialsRequest it will create a Secret (as today mostly) with new functionality: Adding to the Secret a path to the
projected ServiceAccount token (static for OCP) and a roleARN (from the new field on the CredentialsRequest).

Validation of CredentialsRequest by this new token-aware CCO? Maybe. Currently, CCO gets some permissions under ROSA
(unused), could expand these permissions to include tools like AWS's Policy Simulator such that it could validate a role
has permissions, and with this CCO could be the alerting mechanism for a changed CredentialsRequest without sufficient 
permissions.

**HyperShift changes**: Include Cloud Credential Operator with token-aware mode (see above). Allows for processing of 
CredentialsRequest objects added by operators.

**OperatorHub and Console changes**: Allow for input from user of additional fields during install depending on the 
Cloud Provider which will result in ENV variables on the Subscription object that are required in the 
CredentialsRequests created by the Operator. Setting the Subscription config.ENV will allow UX for the information 
needed by CCO or webhook, for input of the cloud credentials while not having to change the Subscription API.

Show in OperatorHub that the cluster is in a mode that supports token-based authentication by reading the 
`.spec.serviceAccountIssuer` from the Authentication CR, `.spec.platformSpec.type` from the Infrastructure CR,
`.spec.credentialsMode` from the CloudCredentials CR.

Show that the operator is enabled for Token-based use by reading the CSV annotation provided by the operator author.

Generate a manual ack for operator upgrade when cloud resources are involved. This can be determined by parsing the 
Subscription objects. Ensures admin signs off that cloud resources are provisioned and working as intended before an 
operator upgrade. This could be by setting the default subscription mode to Manual in the UI for operators that support 
TAT.

**Operator team / Operator SDK changes**: Follow new guidelines for allowing for the operator to work on token auth 
enabled cluster. New guidelines would include the following to use CCO Token mode:

- changing CRD to add CredentialRequest references and putting those referenced
- CredentialRequests into a defined directory in the bundle
- Add a bundle annotation claiming token-based authentication support
- Add a bundle annotation indicating which permissions are required
- add the projected ServiceAccount volume to the Deployment embedded in the CSV;
- handle the Secret themselves (like today’s CCO Mint mode, read and use the credentials in the Secret whose name they 
  know from the CredentialsRequest and is in the same Namespace)
- Guidance to operator authors to structure controller code to alert on missing cloud credentials, not crash.

SDK to support this new template. SDK to validate and in particular: alert on any permission changes between operator
versions.

### Workflow Description

This diagram shows the steps a user will need to take in the proposed flow (specific to STS):

![](STS_tomorrow.jpg)

Making a layered operator ready to work on an STS enabled cluster will involve the following steps:

Operator installation under the proposed system:
1. user goes to console and starts operator install
2. console detects token-based (STS) cluster, and token-based-supporting operator
3. console prompts user to create roles+supply ARN-type string
4. console creates subscription with ARN-type string embedded as a spec.config.env
5. operator deployment is created with ARN-type string in env
6. operator creates `CredentialsRequest` including the ARN-type string
7. Cloud Credential Operator populates Secret based on `CredentialsRequest`
8. operator loads Secret, makes cloud service requests


For the operator author team:
- Add CredentialRequests to the bundle, known location;
- Add "role-arn"-type fields to the CredentialRequests
- Add references to the CredentialRequests in the CRD spec;
- Use SDK to validate bundle to catch permission changes or mis-configuration
- Add eventing to report status on a CR to indicate lacking STS credentials for fully operational deploy or update.

For the Cloud Credential Operator:
- Add a Token mode that will watch for CredentialRequests (as per mint mode) and then resolve time-based tokens
  per cloud platform. 
- Make Secret resource available for the operator via pod identity webhook. 
- Pod identity webhook logic to account for per-operand tenancy permissions.

For Operator Administrator:
- Supply cloud credentials for STS (ARN ID, etc) in the known location and change CR to reflect this.

For OperatorHub, Console:
- Prompt for missing credentials that prevent fully operational install on STS cluster. Same for upgrades.
- Badge indicating STS enabled operators (including operators that don't need cloud credentials)

This graph shows the system as it needs to work (AWS only), not the proposed solution workflow:
```mermaid
graph  LR
  sa([ServiceAccount Signing Keys]) -- Public <br> Key -->s3[S3 Bucket w/ OIDC Config];
  sa([ServiceAccount Signing Keys]) -- Private <br> Key -->token_signing;
  service<-.->config
  service<-.->Pod
  Pod-->role
  subgraph cluster [AWS]
  s3;
  iam_id[IAM Identity Provider] -- Trust Tokens<br>Signed -->s3;
  role[IAM Role For Pod]<-->service[AWS Services];
  end
  subgraph cluster2 [OpenShift]
  token_signing[Token Signing]-->projected[Projected<br>ServiceAccount<br>Token];
  subgraph Pod
  config[AWS Config File]-->projected
  end
  end
  classDef plain fill:#ddd,stroke:#fff,stroke-width:4px,color:#000;
  classDef k8s fill:#326ce5,stroke:#fff,stroke-width:4px,color:#fff;
  classDef cluster fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;
  classDef cluster2 fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;
  class config,role,service,s3,projected,token_signing,iam_id,pod1,pod2 k8s;
  class sa plain;
  class cluster cluster;
  class cluster2 cluster1;
```

### API Extensions

Added to the CredentialsRequest API 

Adding to the `spec` field, the following subfields:

`cloudTokenString`;<br>
`cloudTokenPath`

On AWS STS, the `cloudTokenString` would be something like:
`arn:aws:iam::269733383066:role/newstscluster-openshift-logging-role-name`<br>
This provides the role information which linked by policy to access various cloud resources.

`cloudTokenPath` provides the path to a JWT web token needed to access the resources.

### Risks and Mitigations

### Drawbacks

## Design Details

We have two PoC items:<br>
How CCO changes might be implemented: https://github.com/openshift/cloud-credential-operator/pull/525

How Cluster Logging Operator can be changed to work with this modified CCO: 
https://github.com/gallettilance/cluster-logging-operator/tree/sts

### Test Plan

Consider the following in developing a test plan for this enhancement:
- CCO should add unit tests for the new manner of working when in Manual mode but the 3-way test indicates running on
  a TAT enabled cluster.
- CCO should add e2e tests for the new manner of working when in Manual mode but the 3-way test indicates running on
  a TAT enabled cluster.
- CI will need to add a TAT mode for various cloud providers as a base for the e2e tests.
- A "canary" operator should be fielded in CI with e2e tests such that we have signal on breaking changes in TAT enabled
  clusters.

### Graduation Criteria

### Upgrade / Downgrade Strategy

### Version Skew Strategy

### Operational Aspects of API Extensions

Should be none. If the new fields on CredentialsRequest are missing, cloud resources will be unavailable for an 
operator, but there should be no other impacts.

#### Failure Modes

If the added CredentialsRequest fields contemplated in this EP are missing, operators should wait them to appear. If
after a timeout period, the fields are not present, the operator should write events about the lack of access to cloud 
resources to the status of an appropriate resource to let admins know there are missing credentials.

## Implementation History

Not implemented yet.

## Alternatives

This could be implemented by with chanes to Pod Identity Webhook (Deployed on classic ROSA, ROSA on HyperShift, 
standard OCP and Hypershift)

Will work by annotating the ServiceAccount triggering the projection of the service account tokens into pods created for
the operator (same method and resultant actions as: https://github.com/openshift/hypershift/pull/1351) the README here
gives an AWS specific example: https://github.com/openshift/aws-pod-identity-webhook)

Possibly some changes to logic for Pod Identity Webhooks as needed for fine-grained credential
management to allow for multi-tenancy (see definition earlier). Also, need to add pod identity webhooks targeting other
cloud providers and change the annotations to more generic naming.

## Infrastructure Needed

TAT (STS on AWS for instance) base for CI e2e tests.