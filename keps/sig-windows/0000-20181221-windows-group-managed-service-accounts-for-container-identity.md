---
kep-number: 0000
title: Windows Group Managed Service Accounts for Container Identity
authors:
  - "@patricklang,@jeremywx,@ddebroy"
owning-sig: "sig-windows"
participating-sigs:
  - "sig-auth"
  - "sig-node"
  - "sig-windows"
  - "sig-docs"
  - "sig-arch"
reviewers:
  - @liggitt
  - @mikedanese
  - @yujuhong
  - @patricklang
approvers:
  - @liggitt
  - @yujuhong
  - @patricklang
editor: TBD
creation-date: 2018-06-20
last-updated: 2019-01-16
status: provisional
---

# Windows Group Managed Service Accounts for Container Identity


## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [User Stories [optional]](#user-stories-optional)
      * [Web Applications with MS SQL Server](#Web-Applications-with-MS-SQL-Server)
    * [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Drawbacks [optional]](#drawbacks-optional)
* [Alternatives [optional]](#alternatives-optional)

## Summary

Active Directory is a service that is built-in and commonly used on Windows Server deployments for user and computer identity. Apps are run using Active Directory identities to enable common user to service, and service to service authentication and authorization. This proposal aims to support a specific type of identity, Group Managed Service Accounts (GMSA), for use with Windows Server containers. This will allow an operator to choose a GMSA at deployment time, and run containers using it to connect to existing applications such as a database or API server without changing how the authentication and authorization are performed.

## Motivation

There has been a lot of interest in supporting GMSA for Windows containers since it's the only way for a Windows application to use an Active Directory identity. This is shown in asks and questions from both public & private conversations:

- https://github.com/kubernetes/kubernetes/issues/51691 "For windows based containers, there is a need to access shared objects using domain user contexts."
- Multiple large customers are asking the Microsoft Windows team to enable this feature through container orchestrators
- Multiple developers have blogged how to use this feature, but all are on standalone machines instead of orchestrated through Kubernetes
  - https://artisticcheese.wordpress.com/2017/09/09/enabling-integrated-windows-authentication-in-windows-docker-container/
  - https://community.cireson.com/discussion/3853/example-cireson-scsm-portal-on-docker-windows-containers
  - https://cloudiqtech.com/windows-2016-docker-containers-using-gmsa-connect-to-sql-server/
  - https://success.docker.com/api/asset/.%2Fmodernizing-traditional-dot-net-applications%2F%23integratedwindowsauthentication 


### Goals

- Windows users can run containers with an existing GMSA identity on Kubernetes
- No extra files or Windows registry keys are needed on each Windows node. All needed data should flow through Kubernetes+Kubelet APIs
- Prevent pods from being inadvertently scheduled with service accounts that do not have access to a GMSA

### Non-Goals

- Running Linux applications using GMSA or a general Kerberos based authentication system
- Replacing any existing Kubernetes authorization or authentication controls. Specifically, a subset of users cannot be restricted from creating pods with Service Accounts authorized to use certain GMSAs within a namespace if the users are already authorized to create pods within that namespace. Namespaces serve as the ACL boundary today in Kubernetes and we do not try to modify or enhance this in the context of GMSAs to prevent escalation of privilege through a service account authorized to use certain GMSAs.
- Providing unique container identities. By design, Windows GMSA are used where multiple nodes are running apps as the same Active Directory principal.
- Isolation between container users and processes running as the GMSA. Windows already allows users and system services with sufficient privilege to create processes using a GMSA.
- Preventing the node from having access to the GMSA. Since the node already has authorization to access the GMSA, it can start processes or services using as the GMSA. Containers do not change this behavior.
- Restricting specification of GMSA credspecs in pods or containers if RBAC is not enabled or the admission webhook described below is not installed/enabled.


## Proposal

### Background

#### What is Active Directory?
Windows applications and services typically use Active Directory identities to facilitate authentication and authorization between resources. In a traditional virtual machine scenario, the computer is joined to an Active Directory domain which enables it to use Kerberos, NTLM, and LDAP to identify and query for information about other resources in the domain. When a computer is joined to Active Directory, it is given an unique identity represented as a computer object in LDAP.

#### What is a Windows service account?
A Group Managed Service Account (GMSA) is a shared Active Directory identity that enables common scenarios such as authenticating and authorizing incoming requests and accessing downstream resources such as a database server, file share, or other workload. It can be used by multiple authorized users or computers at the same time.

#### How is it applied to containers?
To achieve the scale and speed required for containers, Windows uses a group managed service account in lieu of individual computer accounts to enable Windows containers to communicate with Active Directory. As of right now, the Host Computer Service (which exposes the interface to manage containers) in Windows cannot use individual Active Directory computer & user accounts - it only supports GMSA.

Different containers on the same machine can use different GMSAs to represent the specific workload they are hosting, allowing operators to granularly control which resources a container has access to. However, to run a container with a GMSA identity, an additional parameter must be supplied to the Windows Host Compute Service to indicate the intended identity. This proposal seeks to add support in Kubernetes for this parameter to enable Windows containers to communicate with other enterprise resources.

It's also worth noting that Docker implements this in a different way that's not managed centrally. It requires dropping a file on the node and referencing it by name, eg: docker run --credential-spec='file://gmsa-cred-spec1.json' . For more details see the Microsoft doc.


### User Stories


#### Web Applications with MS SQL Server
A developer has a Windows web app that they would like to deploy in a container with Kubernetes. For example, it may have a web tier that they want to scale out running ASP.Net hosted in the IIS web server. Backend data is stored in a Microsoft SQL database, which all of the web servers need to access behind a load balancer. An Active Directory Group Managed Service Account is used to avoid hardcoded passwords, and the web servers run with that credential today. The SQL Database has a user with read/write access to that GMSA so the web servers can access it. When they move the web tier into a container, they want to preserve the existing authentication and authorization models.

When this app moves to production on containers, the team will need to coordinate to make sure the right GMSA is carried over to the container deployment.

1. An Active Directory domain administrator will:

  - Join all Windows Kubernetes nodes to the Active Directory domain.
  - Provision the GMSA and gives details to Kubernetes cluster admin.
  - Assign access to a AD security group to control what machines can use this GMSA. The AD security group should include all authorized Kubernetes nodes.

2. A Kubernetes cluster admin (or cluster setup tools or an operator) will install a cluster scoped CRD (of kind GMSACredSpec) whose instances will be used to store GMSA credential spec configuration:

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: gmsacredspecs.auth.k8s.io
spec:
  group: auth.k8s.io
  version: v1alpha1
  names:
    kind: GMSACredSpec
    plural: gmsacredspecs
  scope: Cluster
```

A GMSACredSpec may be used by sidecar containers across different namespaces. Therefore the CRD needs to be cluster scoped.

3. A Kubernetes cluster admin will create a GMSACredSpec object populated with the credential spec associated with a desired GMSA:

  - The cluster admin may run a utility Windows PowerShell script to generate the YAML definition of a GMSACredSpec object populated with the GMSA credential spec details. Note that the credential spec YAML follows the structure of the credential spec (in JSON) as referred to in the [OCI spec](https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec). The utility Powershell script for generating the YAML will be largely identical to the already published [Powershell script](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/windows-server-container-tools/ServiceAccounts/CredentialSpec.psm1) with the following differences: [a] it will output the credential spec in YAML format and [b] it will encapsulate the credential spec data within a Kubernetes object YAML (of kind GMSACredSpec). The GMSACredSpec YAML will not contain any passwords or crypto secrets. Example credspec YAML for a GMSA webapplication1:

```
apiVersion: auth.k8s.io/v1alpha1
kind: GMSACredSpec
metadata:
  name: "webapp1-credspec"
credspec:
  ActiveDirectoryConfig:
    GroupManagedServiceAccounts:
    - Name: WebApplication1
      Scope: DEMO
    - Name: WebApplication1
      Scope: contoso.com
  CmsPlugins:
  - ActiveDirectory
  DomainJoinConfig:
    DnsName: contoso.com
    DnsTreeName: contoso.com
    Guid: 244818ae-87ca-4fcd-92ec-e79e5252348a
    MachineAccountName: WebApplication1
    NetBiosName: DEMO
    Sid: S-1-5-21-2126729477-2524075714-3094792973
```
  
  - With the YAML from above (or generated manually), the cluster admin will create a GMSACredSpec object in the cluster.

4. A Kubernetes cluster admin will configure RBAC for the GMSACredSpec so that only desired service accounts can use the GMSACredSpec:

  - Create a cluster wide role to get and "use" the GMSACredSpec:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webapp1-gmsa-user
rules:
- apiGroups: ["auth.k8s.io"]
  resources: ["gmsacredspecs"]
  resourceNames: ["webapp1-credspec"]
  verbs: ["get", "use"]
```

  - Create a rolebinding and assign desired service accounts to the above role:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: use-webapp1-gmsa
  namespace: default
subjects:
- kind: ServiceAccount
  name: webapp-sa
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: webapp1-gmsa-user
  apiGroup: rbac.authorization.k8s.io
```
  
5. Application admins will deploy app pods that require a GMSA identity along with a Service Account authorized to use the GMSAs. There will be two ways to specify the GMSA credspec details for pods and containers. It is expected that users will typically use the first option as it is more user friendly. The second option is available mainly due to an artifact of the design choices made and described here for completeness.

  - Specify the name of the desired GMSACredSpec object (e.g. `webapp1-credspec`): If an application administrator wants containers to be initialized with a GMSA identity, specifying the names of the desired GMSACredSpec objects is mandatory. In the alpha stage of this feature, the name of the desired GMSACredSpec can be set through an annotation on the pod (applicable to all containers): `pod.alpha.kubernetes.io/windows-gmsa-credspec-name` as well as for each container through annotations of the form: `container.windows-gmsa-credspec-name.alpha.kubernetes.io/container-name`. In the beta stage, the annotations will be promoted to a field in the securityContext of the pod: `podspec.securityContext.windows.gmsaCredSpecName` and in the securityContext of each container:  `podspec.container[i].securityContext.windows.gmsaCredSpecName`. The GMSACredSpec name for a container will override the GMSACredSpec name specified for the whole pod. Sample pod spec showing specification of GMSACredSpec at the pod level and overriding it for one of the containers:

```
apiVersion: v1
kind: Pod
metadata:
  name: iis
  labels:
    name: iis
  annotations: {
    pod.alpha.kubernetes.io/windows-gmsa-credspec-name : webapp1-credspec
    container.windows-gmsa-credspec-name.alpha.kubernetes.io/iis : webapp2-credspec
  }
spec:
  containers:
    - name: iis
      image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
      ports:
        - containerPort: 80
    - name: logger
      image: eventlogger:2019
      ports:
        - containerPort: 80
  nodeSelector:
    beta.kubernetes.io/os : windows
```

  - Specify the contents of the `credpec` field of GMSACredSpec that gets passed down to the runtime: Specifying the credpec contents in JSON form is optional and unnecessary. It will get automatically populated by GMSAAuthorizer as described in the next section based on the name of the GMSACredSpec object. In the alpha stage of this feature, a JSON representation of the contents of the desired GMSACredSpec may be set through an annotation on the pod (applicable to all containers): `pod.alpha.kubernetes.io/windows-gmsa-credspec` as well as for each container through annotations of the form: `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name`. In the beta stage, the annotations will be promoted to a field the securityContext of the pod `podspec.securityContext.windows.gmsaCredSpec` and in the securityContext of each  container: `podspec.container[i].securityContext.windows.gmsaCredSpec`. The credpec JSON for a container will override the credpec JSON specified for the whole pod.

  The ability to specify credspecs for each container within a pod aligns with how security attributes like `runAsGroup`, `runAsUser`, etc. can be specified at the pod level and overridden at the container level if desired.

6. A mutating webhook admission controller, GMSAAuthorizer, will act on pod creations and execute the following checks and actions:

  - Validate GMSA credspec name references: Check the pod spec for references to names of GMSACredSpec objects in the pod and per-container annotations [`pod.alpha.kubernetes.io/windows-gmsa-credspec-name` and `container.windows-gmsa-credspec-name.alpha.kubernetes.io/container-name` for Alpha] or securityContext fields [`podspec.securityContext.windows.gmsaCredSpecName` and `podspec.container[i].securityContext.windows.gmsaCredSpecName` Beta onwards]. Next, make sure the GMSACredSpec objects (referred to by name) do exist and prepare a JSON representation of the contents of the `credspec` field of the GMSACredSpec. If the specified names of GMSACredSpec cannot be resolved, pod creation will be failed by GMSAAuthorizer with 404: NotFound.

  - Validate or populate GMSA credspec JSON: Check the pod spec for inlined credspec JSON in the pod and per-container annotations [`pod.alpha.kubernetes.io/windows-gmsa-credspec` and `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name` for Alpha] or securityContext fields [`podspec.securityContext.windows.gmsaCredSpec` and `podspec.container[i].securityContext.windows.gmsaCredSpec` Beta onwards]. These may exist as a result of the app admin optionally specifying the credspec JSON (besides the name).
    - If a credspec JSON is not specified already, GMSAAuthorizer will look up the corresponding GMSACredSpec name and use the credspec JSON prepared from the previous step to populate the annotations or securityContext fields for credspec JSON.
    - If a credspec JSON is specified, look up the corresponding GMSACredSpec name and perform a deep equal check of the members of the specified credspec and prepared credspec from previous step. If the deep equal check fails, GMSAAuthorizer will be fail pod creation with 400: BadRequest.

  - Authorize Kubernetes Service Account to use GMSA credspec: Check the specified ServiceAccount is authorized for the `use` verb on all specified GMSACredSpecs at the pod level and for each container. If the authorization check fails due to absence of requisite RBAC roles, the pod creation will be failed with a 403: Forbidden. 
  
  Note that on a Kubernetes cluster with Windows nodes configured for GMSA, it is expected that the kubernetes administrator has configured and enabled the GMSAAuthorizer webhook as well as enabled RBAC authorization mode on the cluster. This is typically taken care of by a kubernetes distribution setup mechanism or kubernetes cluster setup scripts. Please see Alternatives section for other options to handle absence of RBAC authorization mode as well as absence of GMSAAuthorizer webhook.

7. Kubelet.exe in Windows nodes will examine the credspec related annotations [`pod.alpha.kubernetes.io/windows-gmsa-credspec` and `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name` for Alpha] or securityContext fields [`podspec.securityContext.windows.gmsaCredSpec` and `podspec.container[i].securityContext.windows.gmsaCredSpec` Beta onwards] for a given pod as well as for each container in the pod. For each container, Kubelet will compute an effective credspec - either the credspec specified for the whole pod or a credspec specified specifically for the container. During Alpha, Kubelet will set the effective credspec for each container as annotation: `container.alpha.kubernetes.io/windows-gmsa-credspec`. Beta onwards, Kubelet will set the effective credspec for each container in a new security context field `WindowsContainerSecurityContext.CredentialSpec` which will require an update to the CRI API. Please see the Implementation section below for details on the enhancements necessary in Kubelet and CRI.

8. The Windows CRI implementation will access the credspec JSON through annotations [`container.alpha.kubernetes.io/windows-gmsa-credspec` for Alpha] or securityContext field [`WindowsContainerSecurityContext.CredentialSpec` Beta onwards] in `CreateContainerRequest` for each container in a pod. The CRI implementation will transmit the credspec JSON through a runtime implementation dependent mechanism to a specific container runtime. For example:
 - Docker (to be supported in Alpha): will receive the path to a file created on the node's file system under `C:\ProgramData\docker\CredentialSpecs\` and populated by dockershim with the credspec JSON. Docker will read the contents of the credspec file and pass it to Windows Host Compute Service (HCS) when creating and starting a container in Windows.
 - ContainerD (to be supported in Beta): will receive a OCI Spec with [windows.CredentialSpec]( https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec) populated by CRIContainerD with the credspec JSON. The OCI spec will be passed to a OCI runtime like RunHCS.exe when starting a container.
Please see the Implementation section below for details on the enhancements necessary for select CRI implementations and their corresponding runtimes.

9. The Windows container runtime (Docker or ContainerD + RunHCS) will depend on Windows Host Compute Service (HCS) APIs/vmcompute.exe to start the containers and assign them user identities  corresponding to the GMSA configured for each container. Using the GMSA identity, the processes within the container can authenticate to a service protected by GMSA like database. The containers that fail to start due to invalid GMSA configuration (as determined by HCS or container runtime like Docker) will end up in a failed state with the following when described:
```
State:              Waiting
      Reason:           CrashLoopBackOff
Last State:         Terminated
      Reason:           ContainerCannotRun
      Message:          [runtime specific error e.g. "encountered an error during CreateContainer:..."]
```

There are a couple of failure scenarios to consider in the context of incorrect GMSA configurations for containers and hosts where containers do get started in spite of incorrect GMSA configuration:

 - On a Windows node not connected to AD, a GMSA credspec JSON is passed that is structurally valid JSON: Windows HCS APIs are not able to detect the fact that the Windows node is not connected to a domain and starts the container normally.
 - On a Windows node connected to AD, a GMSA credspec JSON is passed that the node is not configured to use: Windows HCS APIs are not able to detect the fact that the Windows node is not authorized to use the GMSA and starts the container.

In the above scenarios, after a container has started with the GMSA credspec JSON configured, within the container, validation commands like `whoami /upn` fails with `Unable to get User Principal Name (UPN) as the current logged-on user is not a domain user` and `nltest /query` fails with `ERROR_NO_LOGON_SERVERS`/`ERROR_NO_TRUST_LSA_SECRET`. An init container is recommended to be added to pod specs (where GMSA configurations are desired) to validate that the desired domain can be reached (through `nltest /query`) as well as the desired identity can be obtained (through `whoami /upn`) with the configured GMSA credspecs. If the validation commands fail, the init container will exit with failures and thus prevent pod bringup. The failure can be discovered by describing the pod and looking for errors along the lines of the following for the GMSA validating init container:
```
State:           Waiting
      Reason:        CrashLoopBackOff
Last State:      Terminated
      Reason:        Error
```

10. During any pod update, any changes to a pod's `securityContext` will be blocked (as is the case today) by `ValidatePodUpdate` Beta onwards. Updates of the annotations for GMSACredSpec will be rejected by GMSAAuthorizer in Alpha stage.


### Implementation Details/Notes/Constraints [optional]

#### GMSA specification for pods and containers
In the Alpha phase of this feature we will use the following annotations on a pod for GMSA credspec:
  - References to names of GMSACredSpec objects will be specified through the following annotations:
    - At the pod level: `pod.alpha.kubernetes.io/windows-gmsa-credspec-name`
    - At the container level: `container.windows-gmsa-credspec-name.alpha.kubernetes.io/container-name`
  - The contents of the credspec will be specified through or populated in the following annotations:
    - At the pod level: `pod.alpha.kubernetes.io/windows-gmsa-credspec`
    - At the container level: `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name`

In the Beta phase of this feature, the annotations will graduate to fields in the pod spec:
  - References to names of GMSACredSpec objects will be specified through the following fields:
    - At the pod level `podspec.securityContext.windows.gmsaCredSpecName`
    - At the container level: `podspec.container[i].securityContext.windows.gmsaCredSpecName`
  - The contents of the credspec will be specified through or populated in the following fields:
    - At the pod level: `podspec.securityContext.windows.gmsaCredSpec`
    - At the container level: `podspec.container[i].securityContext.windows.gmsaCredSpec`

#### GMSAAuthorizer Admission Controller Webhook
A new admission controller webhook, GMSAAuthorizer, will be implemented to act on pod creation and updates.

During pod creation, GMSAAuthorizer will first check for references to names of GMSACredSpec objects either through annotations or securityContext fields described above. If the GMSACredSpec object is not found, pod creation is failed with a 404 NotFound error. If found, the contents of the `credspec` field of the GMSACredSpec object will be used to:
  - Populate the contents of the credspec JSON annotations or securityContext fields if not already specified.
  - Check in a deep-equal fashion, the contents of the credspec JSON annotations or securityContext fields if already specified. If the check fails, pod creation will be failed with 400: BadRequest

Next, GMSAAuthorizer will ensure the service account specified for the pod is authorized for a special `use` verb on the GMSACredSpec objects whose name is specified through the annotations or securityContext fields. GMSAAuthorizer will generate custom `AttributesRecord`s with `verb` set to `use`, `name` set to the GMSACredSpec object and `user` set to the service account of the pod. Finally, the `AttributesRecord`s will be passed to authorizers to check against RBAC configurations. A failure from the authorizes results in a 403: Forbidden.

During pod updates, changes to the credspec annotations will be blocked by GMSAAuthorizer and failed with 400: BadRequest.

If the GMSAAuthorizer webhook is not installed and configured, no authorization checks will be performed on the contents of the credspec JSON. This will allow arbitrary credspec JSON to be specified for pods/containers and sent down to the container runtime. Therefore when configuring Windows worker nodes for GMSA support, in the Alpha stage, Kubernetes cluster administrators need to ensure that the GMSAAuthorizer webhook is installed and configured. 

In the Beta stage of this feature, we may consider adding a special check for the presence of GMSAAuthorizer webhook in the kubelet and fail pod creations with `gmsaCredSpec` or `gmsaCredSpecName` fields populated in podspecs.

#### Changes in CRI API:

In the Alpha phase, no changes will be required in the CRI API. Annotation `container.alpha.kubernetes.io/windows-gmsa-credspec` in `ContainerConfig` will contain the credspec JSON associated with each container.

In the Beta phase, a new field `CredentialSpec String` will be added to `WindowsContainerSecurityContext` in `ContainerConfig`. This field will be populated with the credspec JSON of a Windows container by Kubelet.

#### Changes in Kubelet/kuberuntime for Windows: 

In the Alpha phase, `applyPlatformSpecificContainerConfig` will be enhanced (under a feature flag: WindowsGMSA) to analyze the credspec related annotations on the pod [`pod.alpha.kubernetes.io/windows-gmsa-credspec` and `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name`] and determine an effective credential spec for each container:
 - If `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name` is populated, effective credential spec of the container is set to that value.
 - If `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name` is absent but `pod.alpha.kubernetes.io/windows-gmsa-credspec` is populated, effective credential spec of the container is set to the contents of `pod.alpha.kubernetes.io/windows-gmsa-credspec`.
 - If `container.windows-gmsa-credspec.alpha.kubernetes.io/container-name` is absent and `pod.alpha.kubernetes.io/windows-gmsa-credspec` is absent effective credential spec is nil.
Next, `container.alpha.kubernetes.io/windows-gmsa-credspec` annotation in `ContainerConfig` will be populated with the effective credential spec of the container.

In the Beta phase, `DetermineEffectiveSecurityContext` will be enhanced (also under a feature flag: WindowsGMSA) to analyze the `securityContext.windows.gmsaCredSpec` fields for the pod overall and each container in the podspec and determine an effective credential spec for each container in the same fashion described above (and as it does today for several fields like RunAsUser, etc). Next, `ContainerConfig.WindowsContainerSecurityContext.CredentialSpec` will be populated with the effective credential spec for the container.

#### Changes in Dockershim

During Alpha, `dockerService.CreateContainer` function will be enhanced (under a feature flag: WindowsGMSA) to create a temporary file with a unique name on the host file system under path `C:\ProgramData\docker\CredentialSpecs\`. This file will be populated with the contents of `container.alpha.kubernetes.io/windows-gmsa-credspec` annotation in `CreateContainerRequest.ContainerConfig`. Beta onwards, the `dockerService.CreateContainer` (under a feature flag: WindowsGMSA) will use the contents of `WindowsContainerSecurityContext.CredentialSpec` to populate the file.

The temporary credspec file's path will be used to populate `HostConfig.SecurityOpt` with a credspec file specification. The credspec file will be deleted as soon as `CreateContainer` has been invoked on the Docker client.

#### Changes in CRIContainerD

During Alpha, updating the CRI API and thus enabling interactions with ContainerD as a runtime is not planned. Once the CRI API has been updated to pass the `WindowsContainerSecurityContext.CredentialSpec` during Beta, CRIContainerD should be able to access the credspec JSON. At that point, CRIContainerD will need to be enhanced to populate the [windows.CredentialSpec]( https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec) field of the OCI runtime spec for Windows containers with the credspec JSON passed through CRI.

#### Changes in Windows OCI runtime
The Windows OCI runtime already has support for `windows.CredentialSpec` and is implemented in Moby/Docker as well hcsshim/runhcs.

### Risks and Mitigations

#### Threat vectors and countermeasures
1. Prevent an unauthorized user from referring to an existing GMSA configmap in the pod spec: The GMSAAuthorizer Admission Controller along with RBAC policies with the `use` verb on a GMSA configmap ensures only users allowed by the kubernetes admin can refer to the GMSA configmap in the pod spec.
2. Prevent an unauthorized user from using an existing Service Account that is authorized to use an existing GMSA configmap: The GMSAAuthorizer Admission Controller checks the `user` as well as the service account associated with the pod have `use` rights on the GMSA configmap.
3. Prevent an unauthorized user from reading the GMSA credspec and using it directly through docker on Windows hosts connected to AD that user has access to: RBAC policy on the GMSA configmaps should only allow `get` verb for authorized users.

## Graduation Criteria

- alpha - Initial implementation with webhook and annotations on pods with no API changes in PodSpec or CRI. Kubelet and Dockershim enhancements will be guarded by a feature flag and disabled by default. Manual e2e tests with domain joined Window nodes with Docker as the container runtime in a cluster needs to pass.
- beta - Annotations will be promoted to fields in PodSpec and CRI API. Feature flag will be enabled by default. Basic e2e test infrastructure in place in Azure leveraging the test suites for Windows e2e along with dedicated DC host VMs. Automated testing will target Docker container run time but some manual testing of ContainerD integration also needs to succeed.
- ga - e2e tests passing consistently and tests targeting ContainerD/RunHCS  passing as well assuming ContainerD/RunHCS for Windows is stable.


## Implementation History

Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.
Major milestones might include

- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded

## Drawbacks [optional]

Why should this KEP _not_ be implemented.

## Alternatives 

### Other authentication methods

There are other ways to handle user-service and service-service authentication, but they generally require code changes. This proposal is focused on enabling customers to use existing on-premises Active Directory identity in containers.

For cloud-native applications, there are other alternatives:

- Kubernetes secrets - if both services are run in Kubernetes, this can be used for username/password or preshared secrets available to each app
- PKI - If you have a PKI infrastructure, you could choose to deploy application-specific certificates and change applications to trust specific public keys or intermediate certificates
- Cloud-provider service accounts - there may be other token-based providers available in your cloud. Apps can be modified to use these tokens and APIs for authentication and authorization requests.

### Injecting credentials from a volume

For certain authentication use cases, a preferred approach may be to surface a volume to the pod with the necessary data that a pod needs to assume an identity injected in the volume. In these cases, the container needs to implement logic to consume and act on the injected data.

In case of GMSA support, nothing inside the containers of a pod perform any special steps around assuming an identity as that is taken care of by the container runtime at container startup. A container runtime driven solution like GMSA however does require CRI enhancements as mentioned earlier.

### Specifying only the name of GMSACredSpec objects in pod spec fields/annotations

To keep the pod spec changes minimal, we considered having a single field/annotation that specifies the name of the GMSACredSpec object (rather than an additional field that is populated with the contents of the credspec). This approach had the following drawbacks compared to retrieving and storing the credspec data inside annotations/fields:

- Complicates the Windows CRI code with logic to look up GMSACredSpec objects which may be failure prone.
- Requires the kubelet to be able to access GMSACredSpec objects which may require extra RBAC configuration in a locked down environment.
- Contents of `credspec` in a GMSACredSpec object being referred to may change after pod creation. This leads to confusing behavior.

### Enforce presence of GMSAAuthorizer and RBAC mode to enable GMSA functionality in Kubelet

In order to enforce authorization of service accounts in a cluster before they can be used in conjunction with an approved GMSA, we considered adding checks in the Kubelet layer processing GMSA credspecs. Such enforcement however does not align well with parallel mechanisms and also leads to core Kubernetes code being opinionated about something that may not be necessary.

Today, if PSP or RBAC mode is not configured in a cluster, nothing stops pods with special capabilities from being scheduled. To align with this, we should allow GMSA configurations on pods to be enabled without requiring GMSAAuthorizer to be running and RBAC mode to be enabled.

Further, decoupling basic GMSA functionality in the Kubelet and CRI layers from authorization keeps the core Kuberenetes code non-opinionated around enforcement of authorization of service accounts for GMSA usage. Kubernetes cluster setup tools as well as Kubernetes distribution vendors can ensure that RBAC mode is enabled and GMSAAuthorizer is configured and installed when Windows nodes joined to a domain are deployed in a cluster.


<!-- end matter --> 
<!-- references -->
[oci-runtime](https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#credential-spec)
[manage-serviceaccounts](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/manage-serviceaccounts)
