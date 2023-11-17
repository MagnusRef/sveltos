---
title: Notifications - Projectsveltos
description: Sveltos is an application designed to manage hundreds of clusters by providing declarative APIs to deploy Kubernetes add-ons across multiple clusters.
tags:
    - Kubernetes
    - add-ons
    - helm
    - clusterapi
    - multi-tenancy
    - Sveltos
    - Slack
authors:
    - Gianluca Mardente
---

When you create a ClusterProfile instance using Sveltos, it automatically starts watching for clusters that match the ClusterProfile's clusterSelector field. Once a match is found, [Sveltos](https://github.com/projectsveltos) deploys all of the referenced add-ons, such as helm charts or Kubernetes resource YAMLs, in the matching workload cluster.

After all the necessary add-ons are deployed, you might want to perform other operations, such as running a CI/CD pipeline. However, it's important to ensure that the cluster is healthy, i.e, all necessary add-ons are deployed, before proceeding. To help with this, Sveltos can be configured to assess the cluster health and send notifications if there are any changes.

These notifications can be used by other tools to perform additional actions or trigger workflows. With Sveltos, you can easily deploy and manage Kubernetes add-ons, while ensuring the health and stability of your clusters.

## ClusterHealthCheck

[ClusterHealthCheck](https://github.com/projectsveltos/libsveltos/raw/main/api/v1alpha1/clusterhealthcheck_type.go) is the CRD introduce to:

1. define the cluster health checks;
2. instruct Sveltos when and how to send notifications.

### Cluster Selection

The clusterSelector field is a Kubernetes label selector. Sveltos uses it to detect all the clusters to assess health and send notifications about.

### LivenessChecks
The livenessCheck field is a list of __cluster liveness checks__ to evaluate.

Currently supported types are:

1. __Addons__: Addons type instructs Sveltos to evaluate state of add-ond deployment in such a cluster;
2. __HealthCheck__: HealthCheck type allows to define a custom health check for any Kubernetes type.

### Notifications
The notifications fields is a list of all __notifications__ to be sent when liveness check states change.

Currently supported types are:

1. <img src="../assets/kubernetes_logo.png" alt="Kubernetes" width="25" /> Kubernetes events (__reason=ClusterHealthCheck__);
2. <img src="../assets/slack_logo.png" alt="Slack" width="25" />  Slack message; 
3. <img src="../assets/webex_logo.png" alt="Webex" width="25" />  Webex message.

### Detect pods in crashloopbackoff state and send Slack notification

![Send Slack Notification for Pods in Crashloopbackoff state](assets/notification.gif)

Using following HealthCheck and ClusterhealthCheck instances, we are instructing Sveltos to:

1. detect pods in crashloopbackoff state in any cluster with labels __env=fv```
2. send a Slack notification when such an event is detected

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: HealthCheck
metadata:
 name: crashing-pod
spec:
 group: ""
 version: v1
 kind: Pod
 script: |
   function evaluate()
     hs = {}
     hs.status = "Healthy"
     hs.ignore = true
     if obj.status.containerStatuses then
        local containerStatuses = obj.status.containerStatuses
        for _, containerStatus in ipairs(containerStatuses) do
          if containerStatus.state.waiting and containerStatus.state.waiting.reason == "CrashLoopBackOff" then
            hs.status = "Degraded"
            hs.ignore = false
            hs.message = obj.metadata.namespace .. "/" .. obj.metadata.name .. ":" .. containerStatus.state.waiting.message
            if containerStatus.lastState.terminated and containerStatus.lastState.terminated.reason then
              hs.message = hs.message .. "\nreason:" .. containerStatus.lastState.terminated.reason
            end
          end
```

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: ClusterHealthCheck
metadata:
 name: crashing-pod
spec:
 clusterSelector: env=fv
 livenessChecks:
 - name: crashing-pod
   type: HealthCheck
   livenessSourceRef:
     kind: HealthCheck
     apiVersion: lib.projectsveltos.io/v1alpha1
     name: crashing-pod
 notifications:
 - name: slack
   type: Slack
   notificationRef:
     apiVersion: v1
     kind: Secret
     name: slack
     namespace: default
```

All YAMLs can be found [here](https://github.com/projectsveltos/demos/tree/main/observability)

### Example

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: ClusterHealthCheck
metadata:
  name: production
spec:
  clusterSelector: env=fv
  livenessChecks:
  - name: addons
    type: Addons
  notifications:
  - name: event
    type: KubernetesEvent
  - name: slack
    type: Slack
    notificationRef:
      apiVersion: v1
      kind: Secret
      name: slack
      namespace: default
  - name: webex
    type: Webex
    notificationRef:
      apiVersion: v1
      kind: Secret
      name: webex
      namespace: default      
```

where slack secret contains Slack channel and token

```bash
kubectl create secret generic slack --from-literal=SLACK_TOKEN=<your token> --from-literal=SLACK_CHANNEL_ID=<your channel id> --type=addons.projectsveltos.io/cluster-profile 
```

where webex secret contains Webex room and token

```bash
kubectl create secret generic slack --from-literal=WEBEX_TOKEN=<your token> --from-literal=WEBEX_ROOM_ID=<your channel id> --type=addons.projectsveltos.io/cluster-profile 
```

In this example, when add-ons are deployed in any cluster matching the clusterSelector, Sveltos:

1. generates a Kubernetes event;
2. send a slack message;
3. send a webex message.

To list events generated by Sveltos

```bash
clusterprofile.config.projectsveltos.io/kyverno created
➜  sveltos git:(main) ✗ kubectl get events -n default --field-selector reason=ClusterHealthCheck                                                                                                                                            
LAST SEEN   TYPE      REASON               OBJECT                  MESSAGE
31s         Normal    ClusterHealthCheck   clusterhealthcheck/hc   cluster Capi:default/sveltos-management-workload...
16s         Warning   ClusterHealthCheck   clusterhealthcheck/hc   cluster Capi:default/sveltos-management-workload...
```

Event type will be set to: type: Normal when add-ons are deployed.

Event message contains information on cluster:
1. cluster type: Capi or Sveltos
2. cluster namespace
3. cluster name

![Sveltos Slack notification](assets/slack_notitifcation.png) 

Above is an example of Slack notification delivered.

### HealthCheck CRD

Are you tired of manually checking the health of your Kubernetes resources? With Sveltos, you can define custom health checks using [Lua](https://www.lua.org/) scripts! Sveltos watches and evaluates Kubernetes resources, and can send notifications when cluster health changes.

To define a custom health check, simply create a [HealthCheck](https://github.com/projectsveltos/libsveltos/blob/main/api/v1alpha1/healthcheck_type.go) CRD.

Its Spec section contains following fields:

1. ```Spec.Group*/*Spec.Version*/*Spec.Kind`` fields indicating which Kubernetes resources the HealthCheck is for. Sveltos will watch and evaluate those resources anytime a change happens;
2. ```Spec.Namespace``` field can be used to filter resources by namespace;
3. ```Spec.LabelFilters``` field can be used to filter resources by labels;
4. ```Spec.Script``` can contain a [Lua](https://www.lua.org/pil/contents.html) script, which define a custom health check.

 with fields specifying which Kubernetes resources the check is for, and a Lua script defining the check. The Lua script must contain a function `evaluate()` that returns a table with a status field (__Healthy__/__Progressing__/__Degraded__/__Suspended__) and optional message field.

When providing Sveltos with a [Lua script](https://www.lua.org/), Sveltos expects following format:

1. must contain a function ```function evaluate()```. This is the function that is directly invoked and passed a Kubernetes resource (inside the function ```obj``` represents the passed in Kubernetes resource);
2. must return a Lua table with following fields:
   1. status: which can be set to either one of	__Healthy__/__Progressing__/__Degraded__/__Suspended__;
   2. ignore: this is a bool indicating whether Sveltos should ignore this resource. If hs.ignore is set to true, Sveltos will ignore the resource causing that result;
   3. message: this is a string that can be set and Sveltos will print if set.

In the follwoing example[^1], we are creating an HealthCheck that watches for all ConfigMap.
hs is the health status object we will return to Sveltos. It must contain a status attribute which indicates whether the resource is Healthy, Progressing, Degraded or Suspended. By default we set it to Healthy and hs.ignore=true, since we don’t want to mess with the status of other, non-OPA ConfigMaps. Optionally, the health status object may also contain a message.

Identify if the ConfigMap is indeed an OPA policy or another kind of ConfigMap. If it is a OPA policy, retrieve the value of the openpolicyagent.org/policy-status annotation. The annotation is set to {"status":"ok"} if the policy was loaded successfully, if errors occurred during loading (e.g., because the policy contained a syntax error) the cause will be reported in the annotation. Depending on the value of the annotation, we set the status and message attributes appropriately.

At the end, we return the hs object to Sveltos.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: HealthCheck
metadata:
 name: opa-configmaps
spec:
 group: ""
 version: v1
 kind: ConfigMap
 script: |
   function evaluate()
     hs = {}
     hs.status = "Healthy"
     hs.ignore = true
     local opa_annotation = "openpolicyagent.org/policy-status"
     if obj.metadata.annotations ~= nil then
       if obj.metadata.annotations[opa_annotation] ~= nil then
         hs.ignore = false
         if obj.metadata.annotations[opa_annotation] == '{"status":"ok"}' then
           hs.status = "Healthy"
           hs.message = "Policy loaded successfully"
         else
           hs.status = "Degraded"
           hs.message = obj.metadata.annotations[opa_annotation]
         end
       end
     end
     return hs
   end
```

Following ClusterHealthCheck, will sends a Webex message as notification anytime a ConfigMap
with an incorrect OPA policy is found.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: ClusterHealthCheck
metadata:
 name: hc
spec:
 clusterSelector: env=fv
 livenessChecks:
 - name: deployment
   type: HealthCheck
   livenessSourceRef:
     kind: HealthCheck
     apiVersion: lib.projectsveltos.io/v1alpha1
     name: opa-configmaps
 notifications:
 - name: webex
   type: Webex
   notificationRef:
     apiVersion: v1
     kind: Secret
     name: webex
     namespace: default
```

When writing an HealthCheck with Lua, it might be handy to validate it before using it.
In order to do so, clone [sveltos-agent](https://github.com/projectsveltos/sveltos-agent) repo.
Then in *pkg/evaluation/healthchecks* directory, create a directory for your resource if one does not exist already. If a directory already exists, create a subdirectory. Inside it, create:

1. file named *healthcheck.yaml* containing the HealthCheck instance with Lua script;
2. file named *healthy.yaml* containing a Kubernetes resource supposed to be Healthy for the Lua script created in #1 (this is optional);
3. file named *progressing.yaml* containing a Kubernetes resource supposed to be Progressing for the Lua script created in #1 (this is optional);
4. file named *degraded.yaml* containing a Kubernetes resource supposed to be Degraded for the Lua script created in #1 (this is optional);
3. file named *suspended.yaml* containing a Kubernetes resource supposed to be Suspended for the Lua script created in #1 (this is optional);
5. *make test*

That will load the Lua script, pass it the healthy (if available), progressing (if available), degraded (if available), suspended (if available) resources and verify result is the expected one.

### Failed Certificate

Following HealthCheck will detect degraded Certificates.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: HealthCheck
metadata:
 name: failed-cert
spec:
 group: "cert-manager.io"
 version: "v1"
 kind: "Certificate"
 script: |
  function evaluate()
    hs = {}
    hs.ignore = true
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "False" then
            hs.ignore = false
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
        end
      end
    end
    return hs
  end
```

### Another example

Following HealthCheck, considers all Deployments. Any Deployment:

1. with number of available replicas matching number of requested replicas is marked as Healthy;
2. with number of available replicas different than number of requested replicas is marked as Progressing;
3. with number of unavailable replicas set and different than zero, is marked as Degraded.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: HealthCheck
metadata:
 name: deployment-replicas
spec:
 group: "apps"
 version: v1
 kind: Deployment
 script: |
   function evaluate()
     hs = {}
     hs.status = "Progressing"
     hs.message = ""
     if obj.status ~= nil then
       if obj.status.availableReplicas ~= nil then
         if obj.status.availableReplicas == obj.spec.replicas then
           hs.status = "Healthy"
         else
           hs.status = "Progressing"
           hs.message = "expected replicas: " .. obj.spec.replicas .. " available: " .. obj.status.availableReplicas
         end
       end
       if obj.status.unavailableReplicas ~= nil then
          hs.status = "Degraded"
          hs.message = "deployments have unavailable replicas"
       end
     end
     return hs
   end
```

[^1]: Credit for this example to https://blog.cubieserver.de/2022/argocd-health-checks-for-opa-rules/

## Notifications and multi-tenancy

If following label is set on HealthCheck instance created by tenant admin

```
projectsveltos.io/admin-name: <admin>
```

Sveltos will make sure tenant admin can define notifications only looking at resources it has been [authorized to by platform admin](multi-tenancy.md).

Sveltos suggests using following Kyverno ClusterPolicy, which will take care of adding proper label to each HealthCheck at creation time.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
  annotations:
    policies.kyverno.io/title: Add Labels
    policies.kyverno.io/description: >-
      Adds projectsveltos.io/admin-name label on each HealthCheck
      created by tenant admin. It assumes each tenant admin is
      represented in the management cluster by a ServiceAccount.
spec:
  background: false
  rules:
  - exclude:
      any:
      - clusterRoles:
        - cluster-admin
    match:
      all:
      - resources:
          kinds:
          - HealthCheck
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(projectsveltos.io/serviceaccount-name): '{{serviceAccountName}}'
            +(projectsveltos.io/serviceaccount-namespace): '{{serviceAccountNamespace}}'
    name: add-labels
  validationFailureAction: enforce
```