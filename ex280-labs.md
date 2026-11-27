## EX280 Practice Labs  OpenShift Administrator

## Table of Contents

- [Lab 1  HTPasswd Identity Provider](#lab-1-htpasswd-identity-provider)
- [Lab 2  Managing Users](#lab-2-managing-users)
- [Lab 3  Managing Groups](#lab-3-managing-groups)
- [Lab 4  RBAC and Permissions](#lab-4-rbac-and-permissions)
- [Lab 5  Custom Roles](#lab-5-custom-roles)
- [Lab 6  Resource Quota](#lab-6-resource-quota)
- [Lab 7  Limit Range](#lab-7-limit-range)
- [Lab 8  Deploy from Resource Manifests](#lab-8-deploy-from-resource-manifests)
- [Lab 9  Deploy from Images, Templates and Helm Charts](#lab-9-deploy-from-images-templates-and-helm-charts)
- [Lab 10  Deploy Jobs for One-time Tasks](#lab-10-deploy-jobs-for-one-time-tasks)
- [Lab 11  Manage Application Deployments](#lab-11-manage-application-deployments)
- [Lab 12  ReplicaSets](#lab-12-replicasets)
- [Lab 13  Labels and Selectors](#lab-13-labels-and-selectors)
- [Lab 14  Configure Services](#lab-14-configure-services)
- [Lab 15  Manage Secrets](#lab-15-manage-secrets)
- [Lab 16  Manage Configuration Maps](#lab-16-manage-configuration-maps)
- [Lab 17  Mount Storage (PV, PVC, Deployment & Route](#lab-17-mount-storage-pv-pvc-deployment-route)
- [Lab 18  Storage Classes](#lab-18-storage-classes)
- [Lab 19  StatefulSets](#lab-19-statefulsets)
- [Lab 20  Mount Storage  PV and PVC](#lab-20-mount-storage-pv-and-pvc)
- [Lab 21  ServiceAccounts and SCC](#lab-21-serviceaccounts-and-scc)
- [Lab 22  Install and Configure an Operator (File Integrity Operator)](#lab-22-install-and-configure-an-operator-file-integrity-operator)
- [Lab 23  Networking: Expose Applications through a Secure Route](#lab-23-networking-expose-applications-through-a-secure-route)
- [Lab 24  Automatically Scale an Application (Horizontal Pod Autoscaler)](#lab-24-automatically-scale-an-application-horizontal-pod-autoscaler)
- [Lab 25  Configure a Project Template with a LimitRange](#lab-25-configure-a-project-template-with-a-limitrange)
- [Lab 26  Configure Network Policies](#lab-26-configure-network-policies)
- [Lab 27  Configure Liveness Probes](#lab-27-configure-liveness-probes)
- [Lab 28  Kustomize: Base Resources](#lab-28-kustomize-base-resources)
- [Lab 29  Kustomize: Overlays and Patches](#lab-29-kustomize-overlays-and-patches)
- [Lab 30  Import, Export and Configure Kubernetes Resources](#lab-30-import-export-and-configure-kubernetes-resources)
- [Lab 31  Locate and Examine Container Images](#lab-31-locate-and-examine-container-images)
- [Lab 32  Query, Format and Filter Kubernetes Resources](#lab-32-query-format-and-filter-kubernetes-resources)
- [Lab 33  Cluster Health, Status, Events and Alerts](#lab-33-cluster-health-status-events-and-alerts)
- [Lab 34  Logs and Container/Pod Troubleshooting](#lab-34-logs-and-containerpod-troubleshooting)
- [Lab 35  Use OpenShift Product Documentation](#lab-35-use-openshift-product-documentation)
- [Lab 36  Ingress Controller and Cluster Network Ingress](#lab-36-ingress-controller-and-cluster-network-ingress)
- [Lab 37  Software Defined Networking and Network Troubleshooting](#lab-37-software-defined-networking-and-network-troubleshooting)
- [Lab 38  TLS for External and Internal Application Traffic](#lab-38-tls-for-external-and-internal-application-traffic)
- [Lab 39  Configure a LoadBalancer Service](#lab-39-configure-a-loadbalancer-service)
- [Lab 40  Expose Non-HTTP Applications with SNI](#lab-40-expose-non-http-applications-with-sni)
- [Lab 41  Operator Lifecycle: Uninstall and Delete](#lab-41-operator-lifecycle-uninstall-and-delete)
- [Lab 42  Run Privileged Applications and Manage SCC](#lab-42-run-privileged-applications-and-manage-scc)
- [Lab 43  Configure Application Access to Kubernetes APIs](#lab-43-configure-application-access-to-kubernetes-apis)
- [Lab 44  Create and Manage Kubernetes CronJobs](#lab-44-create-and-manage-kubernetes-cronjobs)

  

## Remaining Labs to Add

The following Labs 28–44 are now written as complete practice exercises, including tasks, command-by-command solutions, verification and cleanup where applicable. They are additional practice material; check the current Red Hat EX280 objectives for the exact exam scope and OpenShift version.

### Declarative Resource Management

<a id="lab-28-kustomize-base-resources"></a>
## Lab 28  Kustomize: Base Resources

**Objective:** Build and deploy a reusable set of Kubernetes resources with Kustomize.

**Task details:** Create a base containing a Deployment and ClusterIP Service for an nginx application. Use two replicas, label the Pods `app=web`, and expose container port 8080 (nginx listens on port 80, so use port 80 for this working example).

**Solution**

**Step 1  Create the project and directory**
```bash
oc new-project ex280-kustomize
mkdir -p ~/ex280-kustomize/base
cd ~/ex280-kustomize/base
```

**Step 2  Create the Deployment**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - name: http
          containerPort: 80
```

**Step 3  Create the Service and Kustomization file**
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 8080
    targetPort: 80
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

**Step 4  Preview and apply**
```bash
oc kustomize .
oc apply -k .
```

**Step 5  Verify**
```bash
oc get deployments,pods,services
oc get pods -l app=web
oc get endpoints web
```

The Deployment should become available with two replicas and the Service should have endpoints. `oc apply -k` reads the `kustomization.yaml` file and applies the resources it lists.

**Cleanup**
```bash
oc delete project ex280-kustomize
```

<a id="lab-29-kustomize-overlays-and-patches"></a>
## Lab 29  Kustomize: Overlays and Patches

**Objective:** Use Kustomize overlays to maintain environment-specific configuration without duplicating base manifests.

**Task details:** Create a base Deployment and Service, then create `dev` and `prod` overlays. Development should run one replica; production should run three. Give each overlay its own namespace and environment label.

**Solution**

**Step 1  Create the directory structure**
```bash
mkdir -p ~/ex280-overlays/base ~/ex280-overlays/overlays/{dev,prod}
cd ~/ex280-overlays
```

**Step 2  Create base resources**
```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 8080
    targetPort: 80
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

**Step 3  Create the development overlay**
```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ex280-dev
resources:
- ../../base
commonLabels:
  environment: dev
replicas:
- name: web
  count: 1
```

**Step 4  Create the production overlay**
```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ex280-prod
resources:
- ../../base
commonLabels:
  environment: prod
replicas:
- name: web
  count: 3
```

**Step 5  Build and inspect both variants**
```bash
oc kustomize overlays/dev
oc kustomize overlays/prod
```

**Step 6  Create namespaces and deploy**
```bash
oc create namespace ex280-dev
oc create namespace ex280-prod
oc apply -k overlays/dev
oc apply -k overlays/prod
```

**Step 7  Verify**
```bash
oc get deployment,service -n ex280-dev
oc get deployment,service -n ex280-prod
oc get deployment web -n ex280-dev -o jsonpath='{.spec.replicas}{"\n"}'
oc get deployment web -n ex280-prod -o jsonpath='{.spec.replicas}{"\n"}'
```

Expected replica counts are one in `ex280-dev` and three in `ex280-prod`. If your Kustomize version warns about deprecated `commonLabels`, use `labels` with a `pairs` entry instead.

**Cleanup**
```bash
oc delete namespace ex280-dev ex280-prod
```

<a id="lab-30-import-export-and-configure-kubernetes-resources"></a>
## Lab 30  Import, Export and Configure Kubernetes Resources

**Objective:** Export live resources, edit their YAML representation and apply the updated manifest declaratively.

**Task details:** Deploy nginx, export its Deployment, change the replica count to two, save the manifest and apply it. Compare the live object with the local manifest.

**Solution**

**Step 1  Create a project and deploy an application**
```bash
oc new-project ex280-export
oc create deployment web --image=nginx:1.25
oc expose deployment web --port=80
oc rollout status deployment/web
```

**Step 2  Export the Deployment**
```bash
oc get deployment web -o yaml > web-deployment.yaml
```

Open `web-deployment.yaml` and remove server-generated fields before reusing it. At minimum remove `status`, `metadata.uid`, `metadata.resourceVersion`, `metadata.creationTimestamp`, `metadata.generation`, `metadata.managedFields` and any `metadata.annotations` that are generated by the cluster. Keep `apiVersion`, `kind`, `metadata.name`, labels, selectors and the Pod template.

**Step 3  Edit the desired state**
Change `spec.replicas` to `2`. For a minimal clean manifest, use:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: ex280-export
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Step 4  Apply the manifest**
```bash
oc apply -f web-deployment.yaml
oc rollout status deployment/web
```

**Step 5  Compare local and live configuration**
```bash
oc get deployment web
oc get deployment web -o yaml
oc diff -f web-deployment.yaml
```

`oc diff` displays differences between the local configuration and the live object; no output means there are no differences detected for the submitted configuration. `oc apply` is the declarative workflow. `oc create -f` is for creating a resource that does not already exist.

**Cleanup**
```bash
oc delete project ex280-export
```

### Images and Resource Queries

<a id="lab-31-locate-and-examine-container-images"></a>
## Lab 31  Locate and Examine Container Images

**Objective:** Find container images, inspect image metadata and distinguish tags from digests.

**Task details:** Inspect an image available from a public registry, identify its repository, tag and digest, and inspect an image used by a running Pod.

**Solution**

**Step 1  Search the cluster's image streams**
```bash
oc get imagestreams -A
oc get imagestream -n openshift
```

ImageStreams are OpenShift resources and are not a complete catalogue of every image available in external registries.

**Step 2  Inspect an image reference**
```bash
oc describe imagestream -n openshift
oc get imagestream <IMAGESTREAM_NAME> -n openshift -o yaml
```

For a particular tag:
```bash
oc get imagestream <IMAGESTREAM_NAME> -n openshift \
  -o jsonpath='{range .status.tags[*]}{.tag}{"  "}{.items[0].dockerImageReference}{"\n"}{end}'
```

**Step 3  Inspect the image used by a Pod**
```bash
oc get pods -n openshift -o wide
oc get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{.spec.containers[*].image}{"\n"}'
oc get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{.status.containerStatuses[*].imageID}{"\n"}'
```

The Pod specification shows the requested image reference. The container status `imageID` normally shows the resolved image digest.

**Step 4  Compare tag and digest references**
```bash
# Tag reference (can point to a different image after the tag is moved)
docker.io/library/nginx:1.25

# Digest reference (content-addressed reference)
docker.io/library/nginx@sha256:<DIGEST>
```

Use a real digest returned by the registry or runtime; do not type the placeholder literally. A tag is a human-readable pointer and may be mutable. A digest identifies image content. Registry inspection tools such as `skopeo inspect docker://docker.io/library/nginx:1.25` can show remote image metadata when `skopeo` is installed and the registry is reachable.

**Step 5  Inspect an image with Skopeo (optional)**
```bash
skopeo inspect docker://docker.io/library/nginx:1.25
skopeo inspect --raw docker://docker.io/library/nginx:1.25 | head
```

**Cleanup**
No cluster resources are created by this lab. Remove any temporary Pod you created while testing:
```bash
oc delete pod <POD_NAME> -n <NAMESPACE> --ignore-not-found
```

<a id="lab-32-query-format-and-filter-kubernetes-resources"></a>
## Lab 32  Query, Format and Filter Kubernetes Resources

**Objective:** Query resources using label selectors, output formats, JSONPath and cross-namespace commands.

**Task details:** Create two labelled Deployments, list only resources with a chosen label, display selected columns and extract values with JSONPath.

**Solution**

**Step 1  Create sample resources**
```bash
oc new-project ex280-query
oc create deployment frontend --image=nginx:1.25
oc create deployment backend --image=nginx:1.25
oc label deployment frontend tier=frontend
oc label deployment backend tier=backend
```

**Step 2  Filter by label**
```bash
oc get deployments --show-labels
oc get deployments -l tier=frontend
oc get deployments -l 'tier in (frontend,backend)'
```

**Step 3  Use common output formats**
```bash
oc get deployments -o wide
oc get deployment frontend -o yaml
oc get deployment frontend -o json
```

**Step 4  Select custom columns**
```bash
oc get deployments \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,REPLICAS:.spec.replicas
```

**Step 5  Extract values with JSONPath**
```bash
oc get deployment frontend -o jsonpath='{.spec.replicas}{"\n"}'
oc get pods -l tier=frontend \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

**Step 6  Query across all namespaces**
```bash
oc get pods -A
oc get deployments -A
oc get events -A --sort-by=.lastTimestamp
```

Use `-n <namespace>` for one project and `-A` or `--all-namespaces` for cluster-wide listing. Field selectors are supported only for fields the API resource allows, for example:
```bash
oc get pods --field-selector=status.phase=Running
```

**Cleanup**
```bash
oc delete project ex280-query
```

### Cluster Administration and Troubleshooting

<a id="lab-33-cluster-health-status-events-and-alerts"></a>
## Lab 33  Cluster Health, Status, Events and Alerts

**Objective:** Assess cluster health using nodes, ClusterOperators, cluster version, events and monitoring alerts.

**Task details:** Collect a basic health snapshot and identify any nodes or operators that are not in a healthy state.

**Solution**

**Step 1  Check cluster version and status**
```bash
oc get clusterversion
oc describe clusterversion version
oc get clusteroperators
```

ClusterOperators normally report `AVAILABLE=True`, `PROGRESSING=False` and `DEGRADED=False`. A cluster may be updating, so interpret the conditions in context.

**Step 2  Check nodes**
```bash
oc get nodes
oc describe nodes
oc get nodes -o wide
```

Investigate nodes that are `NotReady`, have pressure conditions, or show unexpected taints.

**Step 3  Review events**
```bash
oc get events -A --sort-by=.lastTimestamp
oc get events -n openshift-ingress --sort-by=.lastTimestamp
```

Look for repeated `Warning` events such as `FailedScheduling`, `FailedMount`, `BackOff` and image pull failures. Events are time-limited and may disappear, so also inspect the affected resource.

**Step 4  Inspect monitoring alerts in the console**
In the OpenShift web console, open **Observe → Alerting → Alerts**. Review firing alerts, their severity, description and affected object. The exact menu labels can vary by OpenShift version and user permissions.

If permitted, inspect monitoring resources from the CLI:
```bash
oc get pods -n openshift-monitoring
oc get prometheusrules -A
```

**Step 5  Build a concise health snapshot**
```bash
oc get clusterversion
oc get clusteroperators
oc get nodes
oc get co | grep -v 'True.*False.*False'
```

The final grep is only a quick screen, not a definitive health test: column order and conditions can vary. Always inspect suspicious operators with `oc describe clusteroperator <NAME>`.

**Cleanup**
Read-only lab; no cleanup required.

<a id="lab-34-logs-and-containerpod-troubleshooting"></a>
## Lab 34  Logs and Container/Pod Troubleshooting

**Objective:** Diagnose a failing Pod using status, events, descriptions and current or previous container logs.

**Task details:** Create a Deployment with an invalid image, identify the problem, correct it and verify that the application becomes available.

**Solution**

**Step 1  Create a failing Deployment**
```bash
oc new-project ex280-troubleshoot
oc create deployment broken --image=nginx:does-not-exist
```

**Step 2  Inspect its status**
```bash
oc get pods -l app=broken -w
oc get deployment broken
oc describe deployment broken
oc get events --sort-by=.lastTimestamp
```

The Pod will commonly show `ErrImagePull` or `ImagePullBackOff`. The `describe pod` output and events usually contain the registry error.

**Step 3  Inspect logs**
```bash
oc get pods -l app=broken
oc logs <POD_NAME>
```

An image-pull failure happens before the container starts, so there may be no application logs. For a container that has restarted, use:
```bash
oc logs <POD_NAME> --previous
```

For a multi-container Pod, specify the container:
```bash
oc logs <POD_NAME> -c <CONTAINER_NAME>
```

**Step 4  Correct the image**
```bash
oc set image deployment/broken broken=nginx:1.25
oc rollout status deployment/broken
```

**Step 5  Verify the recovery**
```bash
oc get deployment broken
oc get pods -l app=broken
oc describe pod <NEW_POD_NAME>
oc logs deployment/broken
```

**Troubleshooting checklist**
- `Pending`: inspect scheduling events, node capacity, PVCs and taints.
- `ImagePullBackOff`: inspect image name, tag, registry access and pull secret.
- `CrashLoopBackOff`: inspect current and previous logs, command, environment and probes.
- `CreateContainerConfigError`: inspect referenced ConfigMaps, Secrets and volume settings.
- `Running` but not reachable: inspect Service selectors, endpoints, targetPort, NetworkPolicy and Route.

**Cleanup**
```bash
oc delete project ex280-troubleshoot
```

<a id="lab-35-use-openshift-product-documentation"></a>
## Lab 35  Use OpenShift Product Documentation

**Objective:** Use the documentation that matches the installed OpenShift version to find resource syntax, command options and supported configuration.

**Task details:** Find the documentation for Deployments, Routes and NetworkPolicy. Confirm the cluster version, locate an example manifest and validate a local manifest before applying it.

**Solution**

**Step 1  Identify the cluster version**
```bash
oc get clusterversion version
oc version
```

Record the OpenShift version before using documentation. Documentation for a different major or minor release may describe fields or behavior that differ from your cluster.

**Step 2  Open the matching documentation**
Use the official OpenShift documentation: https://docs.openshift.com/ . Select the documentation version corresponding to the cluster, then use the search function for:
- Deployments and workload configuration
- Routes and TLS termination
- Network policies
- `oc` command-line reference

**Step 3  Inspect API resources from the cluster**
```bash
oc explain deployment
oc explain deployment.spec
oc explain route.spec --api-version=route.openshift.io/v1
oc explain networkpolicy.spec --api-version=networking.k8s.io/v1
```

`oc explain` reads the API schema served by the cluster and is useful for checking available fields.

**Step 4  Validate a manifest without applying it**
```bash
oc apply --dry-run=server -f resource.yaml
```

Server-side dry run asks the API server to validate the object without persisting it. If server-side dry run is unavailable for a particular resource or cluster configuration, use the validation options supported by that `oc` version.

**Step 5  Check command syntax**
```bash
oc help
oc explain route
oc set probe --help
oc create route --help
```

**Key exam habit:** Verify syntax and resource fields rather than guessing. Prefer the documentation version matching the cluster and use `oc explain` for the live API schema.

**Cleanup**
Read-only lab; no cleanup required. Do not apply a manifest unless you intend to create or modify that resource.

### Networking

<a id="lab-36-ingress-controller-and-cluster-network-ingress"></a>
## Lab 36  Ingress Controller and Cluster Network Ingress

**Objective:** Inspect OpenShift IngressControllers and understand how Routes are handled by the cluster ingress layer.

**Task details:** Identify the default IngressController, its domain, replicas and endpoint publishing strategy. Inspect a Route and determine which router admitted it.

**Solution**

**Step 1  Inspect ingress configuration**
```bash
oc get ingresscontroller -n openshift-ingress-operator
oc describe ingresscontroller default -n openshift-ingress-operator
oc get ingress.config.openshift.io cluster -o yaml
```

**Step 2  Inspect router Pods and Services**
```bash
oc get pods -n openshift-ingress -o wide
oc get services -n openshift-ingress
oc get deployments -n openshift-ingress
```

**Step 3  Create a test application and Route**
```bash
oc new-project ex280-ingress
oc create deployment web --image=nginx:1.25
oc expose deployment web --port=80
oc expose service web
oc get route web
```

**Step 4  Inspect Route admission and host**
```bash
oc describe route web
oc get route web -o yaml
```

Look under `status.ingress` for the router name and admission condition. The Route host is normally based on the cluster ingress domain.

**Step 5  Test the route**
```bash
HOST=$(oc get route web -o jsonpath='{.spec.host}')
curl -I "http://${HOST}"
```

If the client cannot resolve the host or reach the router, verify DNS and the cluster's external ingress networking. Do not modify the cluster-wide IngressController for this basic exercise.

**Cleanup**
```bash
oc delete project ex280-ingress
```

<a id="lab-37-software-defined-networking-and-network-troubleshooting"></a>
## Lab 37  Software Defined Networking and Network Troubleshooting

**Objective:** Troubleshoot Pod-to-Pod and Pod-to-Service connectivity using selectors, endpoints, DNS and NetworkPolicy.

**Task details:** Deploy a frontend and backend. Confirm connectivity, then apply a NetworkPolicy that blocks ingress to the backend and verify the effect.

**Solution**

**Step 1  Create the project and workloads**
```bash
oc new-project ex280-network
oc create deployment frontend --image=registry.access.redhat.com/ubi9/ubi -- sleep 36000
oc create deployment backend --image=registry.access.redhat.com/ubi9/ubi -- sleep 36000
oc label deployment frontend role=frontend
oc label deployment backend role=backend
oc expose deployment backend --port=8080 --target-port=8080
```

This simple pair of containers is for network-path practice; it does not run an HTTP server. For an HTTP test, use an application image that listens on the target port.

**Step 2  Inspect Service selection and endpoints**
```bash
oc get svc backend -o yaml
oc get endpoints backend
oc get pods --show-labels -o wide
```

A Service with no endpoints usually has a selector that does not match ready Pod labels, or the Pods are not ready.

**Step 3  Check DNS from a temporary Pod**
```bash
oc run dns-test --image=registry.access.redhat.com/ubi9/ubi \
  --restart=Never --command -- sleep 36000
oc wait --for=condition=Ready pod/dns-test --timeout=90s
oc exec dns-test -- getent hosts backend
```

If `getent` is not present in the selected image, use a diagnostic image that contains DNS utilities. The fully qualified service name is `backend.ex280-network.svc.cluster.local`.

**Step 4  Apply a policy that permits only frontend Pods**
```yaml
# backend-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-only
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

```bash
oc apply -f backend-policy.yaml
oc get networkpolicy
oc describe networkpolicy allow-frontend-only
```

**Step 5  Understand and test the result**
The policy selects backend Pods and allows ingress only from Pods labelled `role=frontend` in the same namespace. Other sources are denied for ingress to selected backend Pods. NetworkPolicy enforcement depends on the cluster network plugin.

For a meaningful connection test, deploy a server listening on port 8080 in the backend and use a client Pod to connect before and after applying the policy. Check Pod labels, Service endpoints, DNS and policy selectors if the result differs from expectations.

**Cleanup**
```bash
oc delete project ex280-network
```

<a id="lab-38-tls-for-external-and-internal-application-traffic"></a>
## Lab 38  TLS for External and Internal Application Traffic

**Objective:** Configure Edge, Re-encrypt and Passthrough Routes and understand where TLS terminates.

**Task details:** Deploy one HTTP backend and one TLS backend. Create an Edge Route to the HTTP backend, and Re-encrypt and Passthrough Routes to the TLS backend.

**Solution**

**Step 1  Create the project and HTTP application**
```bash
oc new-project ex280-tls
oc create deployment web-http --image=nginx:1.25
oc expose deployment web-http --port=80
oc rollout status deployment/web-http
```

**Step 2  Generate a test certificate**
```bash
openssl req -x509 -nodes -days 30 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=web.example.test"
oc create secret tls web-backend-tls --cert=tls.crt --key=tls.key
```

Use test certificates only in this lab. Production endpoints should use certificates issued and managed according to the organization's PKI requirements.

**Step 3  Deploy a TLS backend**
Create `web-tls.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-tls
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-tls
  template:
    metadata:
      labels:
        app: web-tls
    spec:
      containers:
      - name: tls-server
        image: registry.access.redhat.com/ubi9/ubi
        command: ["/bin/sh", "-c"]
        args:
        - exec openssl s_server -accept 8443 -cert /certs/tls.crt -key /certs/tls.key -www
        ports:
        - name: tls
          containerPort: 8443
        volumeMounts:
        - name: certificate
          mountPath: /certs
          readOnly: true
      volumes:
      - name: certificate
        secret:
          secretName: web-backend-tls
---
apiVersion: v1
kind: Service
metadata:
  name: web-tls
spec:
  selector:
    app: web-tls
  ports:
  - name: tls
    port: 8443
    targetPort: tls
```

```bash
oc apply -f web-tls.yaml
oc rollout status deployment/web-tls
oc get service web-tls
```

The `openssl s_server -www` process provides a simple HTTPS test endpoint. If the image used in your environment does not contain OpenSSL, use an approved image that provides `openssl s_server` or substitute a TLS-enabled test application.

**Step 4  Create an Edge Route**
```bash
oc create route edge web-edge --service=web-http
oc get route web-edge
```

With Edge termination, TLS ends at the OpenShift router. Router-to-Pod traffic is HTTP.

**Step 5  Create a Re-encrypt Route**
```bash
oc create route reencrypt web-reencrypt --service=web-tls \
  --cert=tls.crt --key=tls.key --dest-ca-cert=tls.crt
oc get route web-reencrypt
```

The router terminates the client TLS connection and establishes a second TLS connection to the backend. `--dest-ca-cert` supplies the CA certificate used to validate the backend certificate. This self-signed certificate is for isolated practice only.

**Step 6  Create a Passthrough Route**
```bash
oc create route passthrough web-passthrough --service=web-tls
oc get route web-passthrough
```

Passthrough does not terminate TLS at the router. The backend application presents its own certificate to the client.

**Step 7  Verify all three Routes**
```bash
oc get routes
EDGE_HOST=$(oc get route web-edge -o jsonpath='{.spec.host}')
REENC_HOST=$(oc get route web-reencrypt -o jsonpath='{.spec.host}')
PASS_HOST=$(oc get route web-passthrough -o jsonpath='{.spec.host}')
curl -kI "https://${EDGE_HOST}"
curl -k "https://${REENC_HOST}" | head
curl -k "https://${PASS_HOST}" | head
```

The Edge Route reaches the nginx HTTP service. The Re-encrypt and Passthrough Routes reach the TLS test service. The `-k` option is used only because this lab uses a self-signed certificate.

| Termination | TLS ends at | Router-to-backend traffic |
|---|---|---|
| Edge | Router | HTTP by default |
| Re-encrypt | Router, then a new TLS connection to backend | HTTPS |
| Passthrough | Backend application | TLS remains encrypted through router |

**Cleanup**
```bash
oc delete project ex280-tls
rm -f tls.key tls.crt web-tls.yaml
```

<a id="lab-39-configure-a-loadbalancer-service"></a>
## Lab 39  Configure a LoadBalancer Service

**Objective:** Create and inspect a Service of type `LoadBalancer` and compare it with ClusterIP and NodePort.

**Task details:** Deploy nginx, create a LoadBalancer Service and inspect whether the cluster platform assigns an external address.

**Solution**

**Step 1  Create the application**
```bash
oc new-project ex280-lb
oc create deployment web --image=nginx:1.25
```

**Step 2  Create the LoadBalancer Service**
```yaml
# web-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 80
```

```bash
oc apply -f web-lb.yaml
oc get service web-lb -w
```

**Step 3  Inspect the assigned address and endpoints**
```bash
oc describe service web-lb
oc get service web-lb -o wide
oc get endpoints web-lb
```

On platforms with a load-balancer integration, an external IP or hostname may be assigned. On bare-metal or unsupported environments, `EXTERNAL-IP` may remain `<pending>` until a load-balancer implementation is configured. This is not fixed by changing the Service YAML alone.

**Step 4  Compare service types**
```bash
oc expose deployment web --name=web-clusterip --port=8080 --target-port=80
oc expose deployment web --name=web-nodeport --type=NodePort --port=8081 --target-port=80
oc get services
```

- `ClusterIP`: reachable inside the cluster.
- `NodePort`: exposes a port on each node, subject to network access and firewall rules.
- `LoadBalancer`: requests an external load balancer from the platform integration.

**Cleanup**
```bash
oc delete project ex280-lb
```

<a id="lab-40-expose-non-http-applications-with-sni"></a>
## Lab 40  Expose Non-HTTP Applications with SNI

**Objective:** Understand the limits of OpenShift Routes for non-HTTP TCP protocols and the role of TLS SNI.

**Task details:** Inspect the router's default certificate and learn how a TLS passthrough Route uses the TLS Server Name Indication value to select a backend. Do not expose a production service for this exercise.

**Solution**

**Step 1  Understand the requirement**
An ordinary OpenShift Route is primarily designed for HTTP/HTTPS traffic. TLS passthrough Routes can forward encrypted TLS traffic to a backend without decrypting it at the router. The router can use the SNI hostname from the TLS ClientHello to select the Route. A client must send the expected hostname using SNI.

**Step 2  Inspect ingress and the default certificate**
```bash
oc get ingresscontroller default -n openshift-ingress-operator -o yaml
oc get secret -n openshift-ingress
```

Do not print or copy private keys from cluster Secrets. The IngressController's default certificate is distinct from application certificates used behind passthrough Routes.

**Step 3  Review passthrough Route configuration**
A passthrough Route has a TLS section similar to:
```yaml
spec:
  host: secure.example.test
  to:
    kind: Service
    name: tls-backend
  port:
    targetPort: tls
  tls:
    termination: passthrough
```

The Service must expose a named TLS target port and the backend must speak TLS. The application presents the certificate to the client.

**Step 4  Inspect a Route in the cluster**
```bash
oc get routes -A
oc get route <ROUTE_NAME> -n <NAMESPACE> -o yaml
oc describe route <ROUTE_NAME> -n <NAMESPACE>
```

**Step 5  Test SNI when a passthrough service is available**
```bash
openssl s_client -connect <ROUTER_HOST>:443 -servername <ROUTE_HOST>
```

Replace both placeholders with the actual router address and Route host. The `-servername` option supplies SNI. Without the expected SNI hostname, the router may not select the intended passthrough Route.

**Important distinction:** SNI is a TLS hostname-selection mechanism, not a general-purpose TCP routing feature for arbitrary unencrypted protocols. For non-HTTP protocols that do not use TLS/SNI, use a supported Service exposure pattern such as LoadBalancer or NodePort, subject to platform and security requirements.

**Cleanup**
Read-only lab; remove any test application or Route created separately.

### Operators and Application Security

<a id="lab-41-operator-lifecycle-uninstall-and-delete"></a>
## Lab 41  Operator Lifecycle: Uninstall and Delete

**Objective:** Identify the resources involved in an Operator installation and remove an Operator safely.

**Task details:** Inspect an installed Operator in a test namespace, identify its Subscription, ClusterServiceVersion (CSV) and custom resources, then remove the installation. Do not remove a cluster-wide Operator used by other teams.

**Solution**

**Step 1  Find installed Operators**
```bash
oc get subscriptions -A
oc get csv -A
oc get installplans -A
```

**Step 2  Inspect the target Subscription and CSV**
```bash
oc get subscription <SUBSCRIPTION_NAME> -n <OPERATOR_NAMESPACE> -o yaml
oc get csv -n <OPERATOR_NAMESPACE>
oc describe csv <CSV_NAME> -n <OPERATOR_NAMESPACE>
```

Record the Operator namespace, package, channel, source and installed CSV. Identify custom resources created by the Operator before removing it.

**Step 3  Inspect custom resources**
```bash
oc api-resources | grep -i <OPERATOR_KEYWORD>
oc get <CUSTOM_RESOURCE_KIND> -A
```

Export any data that must be retained. Deleting an Operator may stop reconciliation; deleting its custom resources may also delete application-specific resources depending on the Operator.

**Step 4  Remove the Subscription**
```bash
oc delete subscription <SUBSCRIPTION_NAME> -n <OPERATOR_NAMESPACE>
```

Removing the Subscription stops future updates through that Subscription, but does not necessarily remove every Operator resource or CSV in every installation mode.

**Step 5  Remove remaining Operator resources only when appropriate**
```bash
oc get csv -n <OPERATOR_NAMESPACE>
oc get installplan -n <OPERATOR_NAMESPACE>
```

If the CSV remains and this is a dedicated lab installation, remove it:
```bash
oc delete csv <CSV_NAME> -n <OPERATOR_NAMESPACE>
```

Delete the OperatorGroup and namespace only if they were created exclusively for this lab and no other Operator or workload uses them:
```bash
oc get operatorgroup -n <OPERATOR_NAMESPACE>
# Only for a dedicated lab namespace:
oc delete namespace <OPERATOR_NAMESPACE>
```

**Step 6  Verify**
```bash
oc get subscriptions,csv,installplans -A
```

**Key concept:** Removing a Subscription, CSV, OperatorGroup, custom resources and namespace are distinct actions. Check the Operator's documentation and installation scope before deleting cluster-scoped resources.

<a id="lab-42-run-privileged-applications-and-manage-scc"></a>
## Lab 42  Run Privileged Applications and Manage SCC

**Objective:** Inspect Security Context Constraints and grant the minimum SCC required to a dedicated ServiceAccount.

**Task details:** Create a ServiceAccount, inspect the `anyuid` and `privileged` SCCs, grant `anyuid` to the ServiceAccount and verify the assignment. Do not grant the privileged SCC unless a task explicitly requires it.

**Solution**

**Step 1  Create a project and ServiceAccount**
```bash
oc new-project ex280-scc
oc create serviceaccount app-sa
```

**Step 2  Inspect SCCs**
```bash
oc get scc
oc describe scc restricted
oc describe scc anyuid
oc describe scc privileged
```

`restricted` is the normal least-privilege baseline in many OpenShift configurations. `anyuid` permits running with arbitrary UIDs; `privileged` grants much broader host-level capabilities and should be avoided unless essential.

**Step 3  Grant `anyuid` to the ServiceAccount**
```bash
oc adm policy add-scc-to-user anyuid -z app-sa -n ex280-scc
```

**Step 4  Verify the SCC user assignment**
```bash
oc describe scc anyuid
oc auth can-i use scc/anyuid --as=system:serviceaccount:ex280-scc:app-sa
```

**Step 5  Run a test Pod with the ServiceAccount**
```bash
oc run uid-test --image=registry.access.redhat.com/ubi9/ubi \
  --serviceaccount=app-sa --restart=Never --command -- sleep 36000
oc get pod uid-test
oc describe pod uid-test
```

The SCC selected for a Pod can be seen in its annotations on supported OpenShift versions:
```bash
oc get pod uid-test -o jsonpath='{.metadata.annotations.openshift\.io/scc}{"\n"}'
```

**Step 6  Remove the grant and test resources**
```bash
oc adm policy remove-scc-from-user anyuid -z app-sa -n ex280-scc
oc delete project ex280-scc
```

**Exam tip:** Bind an SCC to the specific ServiceAccount, not to all authenticated users or a broad group. Grant only the least privilege required.

<a id="lab-43-configure-application-access-to-kubernetes-apis"></a>
## Lab 43  Configure Application Access to Kubernetes APIs

**Objective:** Give an application a ServiceAccount and narrowly scoped RBAC permissions to read selected API resources.

**Task details:** Create a ServiceAccount named `app-reader`, grant it permission to get and list Pods only in the current namespace, and verify allowed and denied actions.

**Solution**

**Step 1  Create a project and ServiceAccount**
```bash
oc new-project ex280-api
oc create serviceaccount app-reader
```

**Step 2  Create a namespace-scoped Role**
```yaml
# pod-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```bash
oc apply -f pod-reader-role.yaml
```

**Step 3  Bind the Role to the ServiceAccount**
```bash
oc create rolebinding app-reader-binding \
  --role=pod-reader \
  --serviceaccount=ex280-api:app-reader
```

**Step 4  Test authorization**
```bash
oc auth can-i list pods --as=system:serviceaccount:ex280-api:app-reader -n ex280-api
oc auth can-i get pods --as=system:serviceaccount:ex280-api:app-reader -n ex280-api
oc auth can-i create deployments.apps --as=system:serviceaccount:ex280-api:app-reader -n ex280-api
oc auth can-i list pods --as=system:serviceaccount:ex280-api:app-reader -n default
```

Expected: list/get Pods in `ex280-api` are allowed; creating Deployments and listing Pods in another namespace are denied. The administrator running `oc auth can-i --as` must be permitted to impersonate that ServiceAccount.

**Step 5  Inspect the binding**
```bash
oc get role pod-reader -o yaml
oc get rolebinding app-reader-binding -o yaml
oc describe rolebinding app-reader-binding
```

**Step 6  Understand in-Pod API access**
A Pod using this ServiceAccount receives a projected ServiceAccount token and cluster CA information through its mounted token volume, subject to the cluster's token configuration. Applications should use the in-cluster API endpoint and HTTPS, read the token from the mounted file, and avoid copying long-lived credentials into images or source control.

**Cleanup**
```bash
oc delete project ex280-api
```

### Scheduled Workloads

<a id="lab-44-create-and-manage-kubernetes-cronjobs"></a>
## Lab 44  Create and Manage Kubernetes CronJobs

**Objective:** Create and manage a CronJob, configure its schedule and concurrency policy, and inspect the Jobs and Pods it creates.

**Task details:** Create a CronJob that runs every minute, uses the `Forbid` concurrency policy, retains a small number of completed Jobs and prints the current time. Suspend and resume it, then inspect its execution.

**Solution**

**Step 1  Create a project**
```bash
oc new-project ex280-cron
```

**Step 2  Create the CronJob manifest**
```yaml
# report-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: report
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report
            image: registry.access.redhat.com/ubi9/ubi
            command: ["/bin/sh", "-c", "date; echo EX280 scheduled task"]
```

**Step 3  Apply and inspect**
```bash
oc apply -f report-cronjob.yaml
oc get cronjob report
oc describe cronjob report
```

**Step 4  Wait for a scheduled Job**
```bash
oc get jobs -w
```

In another terminal, inspect the Job and Pod:
```bash
oc get jobs
oc get pods
oc logs job/<JOB_NAME>
```

Replace `<JOB_NAME>` with the actual Job name shown by `oc get jobs`.

**Step 5  Suspend and resume the schedule**
```bash
oc patch cronjob report --type=merge -p '{"spec":{"suspend":true}}'
oc get cronjob report
oc patch cronjob report --type=merge -p '{"spec":{"suspend":false}}'
```

Suspending a CronJob prevents future scheduled Jobs; it does not automatically stop Jobs that are already running.

**Step 6  Trigger a one-off Job from the CronJob (optional)**
```bash
oc create job report-manual --from=cronjob/report
oc get jobs
oc logs job/report-manual
```

**Step 7  Clean up**
```bash
oc delete project ex280-cron
```

**Key concepts**
- `schedule` uses standard five-field cron syntax.
- `Forbid` skips a scheduled run if the previous Job is still running.
- `Allow` permits concurrent Jobs; `Replace` replaces an active Job with a new one.
- History limits control how many finished Job objects are retained.
- A CronJob creates Jobs; each Job creates one or more Pods.

---

### Coverage notes

The following objectives are already represented in the current README and do not need duplicate labs:

- HTPasswd, users, passwords, groups, RBAC and custom roles: Labs 1–5.
- ResourceQuota, LimitRange and project resource settings: Labs 6–7 and 24–25.
- YAML manifests, application deployment, templates and Helm: Labs 8–11.
- ReplicaSets, labels, selectors and Services: Labs 12–14.
- Secrets and ConfigMaps: Labs 15–16.
- ServiceAccounts and SCC basics: Lab 21.
- Operator installation: Lab 22.
- Secure Edge Routes and TLS: Lab 23.
- Project templates and project self-service permissions: Labs 4 and 25.
- NetworkPolicies: Lab 26.
- One-time Jobs: Lab 10.
- Liveness probes: Lab 27.
- Project creation is exercised throughout the existing labs; project deletion is included in the cleanup exercise.

The web console is already used in Labs 22, 23, 25 and 26. The additional console practice can be incorporated into Labs 33 and 36 rather than creating a duplicate standalone lab.

---

<a id="lab-1-htpasswd-identity-provider"></a>
## Lab 1  HTPasswd Identity Provider

**Objective:** Configure HTPasswd as an identity provider and manage users

**Task details:**
1. Create an htpasswd file with users alice, bob, charlie and ted
2. Create a secret from the htpasswd file in the openshift-config namespace
3. Configure OAuth to use the htpasswd secret
4. Wait for the OAuth pods to restart
5. Verify the users and identities

**Solution**

**Step 1  Create the htpasswd file**
```bash
htpasswd -c -B -b htpasswd alice redhat
htpasswd -B -b htpasswd ted redhat
htpasswd -B -b htpasswd bob redhat
htpasswd -B -b htpasswd charlie redhat

# Verify
cat htpasswd
```

**Step 2  Create the secret in openshift-config**
```bash
oc create secret generic my-htpass-secret \
  --from-file=htpasswd=htpasswd \
  -n openshift-config

# Verify
oc get secret my-htpass-secret -n openshift-config
```

**Step 3  Configure OAuth**
```bash
oc get oauth cluster -o yaml > oauth.yaml
vi oauth.yaml
```

Ensure the spec section contains:
```yaml
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: my-htpass-secret
    mappingMethod: claim
    name: developer
    type: HTPasswd
```

```bash
oc replace -f oauth.yaml
```

**Step 4  Wait for OAuth to restart**
```bash
oc get pods -n openshift-authentication -w
# Wait until: oauth-openshift-xxxx   1/1   Running
```

**Step 5  Verify users and identities**
```bash
oc get users
oc get identities
```

---

<a id="lab-2-managing-users"></a>
## Lab 2  Managing Users

**Objective:** Remove a user and change a user password

**Task details:**
1. Remove user ted from the htpasswd file and update the secret
2. Clean up the OpenShift user and identity objects for ted
3. Verify ted can no longer log in
4. Change the password of alice to 'password'
5. Verify alice can log in with the new password

**Solution**

**Step 1  Remove ted from htpasswd and update the secret**
```bash
htpasswd -D htpasswd ted

# Verify
cat htpasswd

# Update the secret
oc create secret generic my-htpass-secret \
  --from-file=htpasswd=htpasswd \
  --dry-run=client -o yaml \
  -n openshift-config | oc replace -f -
```

**Step 2  Clean up OpenShift objects**
```bash
oc delete user ted
oc delete identity developer:ted
```

**Step 3  Verify**
```bash
oc login -u ted -p redhat
# Should fail 

oc login -u alice -p redhat
# Should succeed 
```

**Step 4  Change alice's password**
```bash
htpasswd -B -b htpasswd alice password

# Update the secret
oc create secret generic my-htpass-secret \
  --from-file=htpasswd=htpasswd \
  --dry-run=client -o yaml \
  -n openshift-config | oc replace -f -

# Wait for OAuth to reload
oc get pods -n openshift-authentication -w
```

**Step 5  Verify new password**
```bash
oc login -u alice -p password
# Should succeed 
```

---

<a id="lab-3-managing-groups"></a>
## Lab 3  Managing Groups

**Objective:** Create groups, assign users and manage group permissions

**Task details:**
1. Create a new project called manage-groups
2. Create a group called testers with users alice, bob and charlie
3. Assign the view role to the testers group in the manage-groups project
4. Remove alice from the testers group
5. Verify only bob and charlie have access to the project

**Solution**

**Step 1  Create the project**
```bash
oc new-project manage-groups
```

**Step 2  Create the group and add users**
```bash
oc adm groups new testers alice bob charlie
```

**Step 3  Assign the view role to the group**
```bash
oc adm policy add-role-to-group view testers -n manage-groups
```

**Step 4  Remove alice from the group**
```bash
oc adm groups remove-users testers alice
```

**Step 5  Verify**
```bash
oc describe group testers
oc get rolebindings -n manage-groups
```

---

<a id="lab-4-rbac-and-permissions"></a>
## Lab 4  RBAC and Permissions

**Objective:** Manage cluster roles, project roles and self-provisioner permissions

**Task details:**
1. View existing cluster role bindings
2. Test whether a regular user can create projects by default
3. Remove the default self-provisioner rights from all OAuth users
4. Grant user alice explicit permission to create projects
5. Create a new project called permissions
6. Grant user bob admin rights in the permissions project
7. Create a group called developers and add user bob

**Solution**

**Step 1  View cluster role bindings**
```bash
oc get clusterrolebindings -o wide | grep self
oc describe clusterrolebinding.rbac self-provisioners
```

**Step 2  Test as alice**
```bash
oc login -u alice -p password
oc new-project alice10
# Succeeds by default 
```

**Step 3  Remove self-provisioner from all OAuth users**
```bash
oc login -u kubeadmin
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```

**Step 4  Verify alice can no longer create projects**
```bash
oc login -u alice -p password
oc new-project alice10
# Should fail 
```

**Step 5  Check who can create projects**
```bash
oc adm policy who-can create project
```

**Step 6  Grant alice self-provisioner explicitly**
```bash
oc adm policy add-cluster-role-to-user self-provisioner alice

# Verify
oc login -u alice -p password
oc new-project alice22
# Should succeed 
```

**Step 7  Create project and grant bob admin rights**
```bash
oc login -u kubeadmin
oc new-project permissions
oc policy add-role-to-user admin bob -n permissions

# Verify
oc get rolebinding -o wide -n permissions
```

**Step 8  Create developers group and add bob**
```bash
oc adm groups new developers
oc adm groups add-users developers bob
```

---

## Lab 4b  User Roles and Permissions

**Objective:** Manage user roles and permissions in OpenShift

**Task details:**
- mariam: full cluster administration rights
- jane: able to create new projects
- john: not able to create new projects
- ian: view access to all projects in the cluster
- alice: not able to modify the cluster
- Ensure mariam is the only user with permission to modify the cluster

**Solution**

```bash
# mariam: full cluster admin
oc adm policy add-cluster-role-to-user cluster-admin mariam

# jane: can create new projects
oc adm policy add-cluster-role-to-user self-provisioner jane

# john: cannot create new projects
oc adm policy remove-cluster-role-from-user self-provisioner john

# ian: view access to all projects
oc adm policy add-cluster-role-to-user view ian

# alice: cannot modify the cluster
oc adm policy remove-cluster-role-from-user self-provisioner alice
```

---

## Lab 4c  Project Permissions

**Objective:** Set up project permissions for users and groups

**Task details:**
- Create projects: avalon, atlantis, neptune, siren, triton, garuda, nagas
- jane: admin for avalon, atlantis and garuda
- john: view access for avalon and siren
- alice: edit access in avalon (excluding roles and rolebindings)
- Create group heroes (member: alice) with admin rights for neptune, siren, triton, nagas
- Create group wizards (member: ian) with view access for atlantis and edit access for siren

**Solution**

**Create projects**
```bash
oc new-project avalon
oc new-project atlantis
oc new-project neptune
oc new-project siren
oc new-project triton
oc new-project garuda
oc new-project nagas
```

**Assign user permissions**
```bash
# jane as admin
oc adm policy add-role-to-user admin jane -n avalon
oc adm policy add-role-to-user admin jane -n atlantis
oc adm policy add-role-to-user admin jane -n garuda

# john view access
oc adm policy add-role-to-user view john -n avalon
oc adm policy add-role-to-user view john -n siren

# alice edit access
oc adm policy add-role-to-user edit alice -n avalon
```

**Create groups and assign permissions**
```bash
oc adm groups new heroes
oc adm groups new wizards

oc adm groups add-users heroes alice
oc adm groups add-users wizards ian

oc adm policy add-role-to-group admin heroes -n neptune
oc adm policy add-role-to-group admin heroes -n siren
oc adm policy add-role-to-group admin heroes -n triton
oc adm policy add-role-to-group admin heroes -n nagas

oc adm policy add-role-to-group view wizards -n atlantis
oc adm policy add-role-to-group edit wizards -n siren
```

**Verify**
```bash
oc get groups
oc get rolebindings -n avalon
oc get rolebindings -n siren
oc adm policy who-can get pods -n avalon | grep john
oc adm policy who-can create deployments -n avalon | grep alice
```

---

<a id="lab-5-custom-roles"></a>
## Lab 5  Custom Roles

**Objective:** Create a custom role and assign it to a user

**Task details:**
- Create a custom role viewnagas in the nagas project
- The role should provide read-only access to pods, deployments and imagestreams
- The role should permit get, list and watch
- Assign the viewnagas role to john

**Solution**

```bash
# Create the custom role
oc create role viewnagas \
  --verb=get,list,watch \
  --resource=pods,deployments,imagestreams \
  -n nagas

# Assign the role to john
oc adm policy add-role-to-user viewnagas john -n nagas

# Verify
oc describe role viewnagas -n nagas
oc get rolebindings -n nagas
```

---

<a id="lab-6-resource-quota"></a>
## Lab 6  Resource Quota

**Objective:** Create a ResourceQuota for the neptune project

**Task details:**
- Create a ResourceQuota named neptune-quota in the neptune project
- Total memory consumption across all containers: 3Gi maximum
- Total CPU usage across all containers: 2 cores maximum
- Total memory requests across all containers: 1Gi maximum
- Maximum number of replication controllers: 3
- Maximum number of pods: 5
- Maximum number of services: 3

**Solution**

```bash
# Create the ResourceQuota
oc create quota neptune-quota \
  --hard=limits.memory=3Gi,limits.cpu=2,requests.memory=1Gi,replicationcontrollers=3,pods=5,services=3 \
  -n neptune

# Verify
oc describe quota neptune-quota -n neptune
```

---

<a id="lab-7-limit-range"></a>
## Lab 7  LimitRange

**Task details:**

Create a LimitRange named `avalon-limits` in the `avalon` namespace.

- Memory (per container):
  - minimum request: `50Mi`
  - maximum limit: `250Mi`
  - default request: `100Mi`
- CPU (per pod):
  - minimum request: `50m`
  - maximum limit: `250m`
  - default request: `100m`

**Solution**

```bash
# Step 1  Switch to the avalon project
oc project avalon

# Step 2  Create the LimitRange manifest
vi limitrange.yaml

# Step 3  Apply the LimitRange
oc apply -f limitrange.yaml

# Step 4  Verify the LimitRange
oc describe limitrange avalon-limits
```

---

<a id="lab-8-deploy-from-resource-manifests"></a>
## Lab 8  Deploy from Resource Manifests

**Objective:** Deploy and manage an application using YAML manifests

**Task details:**
1. Generate a YAML manifest for a deployment named webserver using image bitnami/nginx in project manifests-demo
2. Edit the manifest to set replicas to 3
3. Apply the manifest to the cluster
4. Verify the pods are running
5. Update the manifest to change replicas to 5
6. Use dry-run to validate the change before applying
7. Check the diff between your local manifest and the cluster
8. Apply the update
9. Restart the deployment
10. Delete all resources using the manifest file

**Solution**

```bash
# Step 1  Create the project and generate the manifest
oc new-project manifests-demo

oc create deployment webserver --image=bitnami/nginx \
  -o yaml --dry-run=client --save-config > webserver.yaml

# Step 2  Edit the manifest
vi webserver.yaml
# Change: replicas: 1 $)B�� replicas: 3

# Step 3  Apply the manifest
oc apply -f webserver.yaml

# Step 4  Verify
oc get pods -w

# Step 5  Update replicas to 5
vi webserver.yaml
# Change: replicas: 3 �� replicas: 5

# Step 6  Validate with dry-run
oc apply -f webserver.yaml --dry-run=server --validate=true

# Step 7  Check the diff
oc diff -f webserver.yaml

# Step 8  Apply the update
oc apply -f webserver.yaml

# Step 9  Restart the deployment
oc rollout restart deployment webserver

# Step 10  Delete all resources
oc delete -f webserver.yaml
```

---

<a id="lab-9-deploy-from-images-templates-and-helm-charts"></a>
## Lab 9  Deploy from Images, Templates and Helm Charts

**Objective**

Deploy applications using three different methods:
- Container images
- OpenShift templates
- Helm charts

**Task details:**

**Part 1  Deploy from image**

1. Create a new project called `deploy-demo`
2. Deploy an application named `myapp` using the image `docker.io/bitnami/nginx`
3. Verify the Pods are running

**Part 2  Deploy from OpenShift template**

4. List the available templates in the `openshift` namespace
5. Examine the `httpd-example` template to see its parameters
6. Save the template to a local file
7. Process the template with parameter `NAME=frontend` and label `app=web`, then apply it to the cluster
8. Verify the created resources

**Part 3  Deploy from Helm chart**

9. Add the Helm repository with the name `nginx-repo` using `https://kubernetesway.github.io/my-helm-repo`
10. Update the Helm repository index
11. Find the chart name and available versions
12. Install the chart using release name `example-app` and chart version `0.1.0`
13. Verify the Helm release
14. Upgrade the `example-app` release to chart version `0.2.0`
15. Verify the upgrade

**Solution**

```bash
# Part 1  Deploy from image

oc new-project deploy-demo
oc new-app --image=docker.io/bitnami/nginx --name=myapp --allow-missing-images
oc get pods -w

# Part 2  Deploy from OpenShift template

# List available templates
oc get templates -n openshift

# Examine the template
oc describe template httpd-example -n openshift

# List template parameters
oc process --parameters httpd-example -n openshift

# Save the template to a local file
oc get template httpd-example -n openshift -o yaml > httpd-example-template.yaml

# Process and apply the template
oc process httpd-example -n openshift \
  -p NAME=frontend \
  -l app=web \
  -o yaml | oc apply -f -

# Verify the created resources
oc get all -l app=web

# Part 3  Deploy from Helm chart

# Add the Helm repository
helm repo add nginx-repo https://kubernetesway.github.io/my-helm-repo

# Update the repository index
helm repo update

# Find the chart name and available versions
helm search repo nginx-repo --versions

# Install chart version 0.1.0
helm install example-app nginx-repo/<chart-name> --version 0.1.0

# Verify the release
helm list
helm status example-app

# Upgrade to chart version 0.2.0
helm upgrade example-app nginx-repo/<chart-name> --version 0.2.0

# Verify the upgrade
helm history example-app
helm status example-app

# Uninstall the Helm release when finished
helm uninstall example-app
```


```

---

<a id="lab-10-deploy-jobs-for-one-time-tasks"></a>
## Lab 10  Deploy Jobs for One-time Tasks

**Objective:** Create and manage a Job in OpenShift for a one-time task.

**Task details:**

1. Generate a Job manifest named `one-time-task` using image `bitnami/nginx` that runs the command `ls`.
2. Save the manifest to a file named `myjob.yaml`.
3. Edit the manifest to add `backoffLimit: 2` and `ttlSecondsAfterFinished: 60`.
4. Deploy the Job.
5. Monitor the Job status.
6. View the logs of the Job Pod.
7. Delete the Job manually.

**Solution**

**Step 1  Generate and save the manifest**

```bash
oc create job one-time-task \
  --image=bitnami/nginx \
  --dry-run=client -o yaml \
  -- ls > myjob.yaml
```

**Step 2  Edit the manifest**

```bash
vi myjob.yaml
```

Ensure the `spec` section looks like this:

```yaml
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 60
  template:
    spec:
      containers:
      - name: one-time-task
        image: bitnami/nginx
        command: ["ls"]
      restartPolicy: Never
```

**Step 3  Deploy the Job**

```bash
oc create -f myjob.yaml
```

**Step 4  Monitor the Job status**

```bash
oc get jobs
oc get pods
```

**Step 5  View the Job Pod logs**

```bash
oc logs <pod-name>
```

**Step 6  Delete the Job manually**

```bash
oc delete job one-time-task
```

---
<a id="lab-11-manage-application-deployments"></a>
## Lab 11  Manage Application Deployments

**Objective:** Deploy, update, scale, rollback and monitor an application deployment

**Task details:**
1. Create a new project called deployment-demo
2. Create a deployment manifest named httpd-deployment.yaml using image nginxinc/nginx-unprivileged:stable with 2 replicas and rolling update strategy
3. Apply the manifest and verify the pods are running
4. Monitor the rollout status
5. Update the nginx-unprivileged container image to nginxinc/nginx-unprivileged:alpine
6. Monitor the rollout and verify the update
7. View the rollout history
8. Rollback to the previous version
9. Verify the rollback was successful
10. Scale the deployment to 5 replicas
11. Pause the deployment
12. Resume the deployment
13. Change the deployment strategy to Recreate and apply

**Solution**

**Step 1  Create the project**
```bash
oc new-project deployment-demo
```

**Step 2  Create the manifest**
```bash
vi httpd-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: bitnamix/nginx-unprivileged:stable
```

```bash
# Step 3  Apply and verify
oc apply -f httpd-deployment.yaml
oc get pods -w

# Step 4  Monitor rollout
oc rollout status deployment httpd-deployment

# Step 5  Update the image
oc set image deployment/httpd-deployment httpd=nginxinc/nginx-unprivileged:alpine

# Step 6  Monitor and verify update
oc rollout status deployment httpd-deployment
oc get all

# Step 7  View rollout history
oc rollout history deployment httpd-deployment
oc rollout history deployment httpd-deployment --revision=1
oc rollout history deployment httpd-deployment --revision=2

# Step 8  Rollback to previous version
oc rollout undo deployment httpd-deployment

# Step 9  Verify rollback
oc describe pod <pod-name> | grep Image

# Step 10  Scale to 5 replicas
oc scale deployment httpd-deployment --replicas=5
oc get pods

# Step 11  Pause the deployment
oc rollout pause deployment httpd-deployment

# Step 12  Resume the deployment
oc rollout resume deployment httpd-deployment

# Step 13  Change strategy to Recreate
vi httpd-deployment.yaml
# Change: type: RollingUpdate �� type: Recreate
# Remove the rollingUpdate section entirely
oc apply -f httpd-deployment.yaml
```

---

<a id="lab-12-replicasets"></a>
## Lab 12  ReplicaSets

**Objective:** Understand and manage ReplicaSets in OpenShift

**Task details:**
1. Create replicaset.yaml with a ReplicaSet named my-replicaset. The ReplicaSet must run 3 replicas of bitnami/nginx and use the label app=myapp.
2. Apply the manifest and verify the pods are running
3. Check the status of the ReplicaSet
4. Scale the ReplicaSet to 5 replicas using oc scale
5. Scale back down to 2 replicas by editing the manifest
6. Delete one pod manually and verify the ReplicaSet recreates it
7. Create a Deployment named my-app-deployment with 3 replicas using image docker.io/bitnami/nginx
8. Observe the ReplicaSet created by the Deployment
9. Update the image to docker.io/nginx
10. Observe how the Deployment creates a new ReplicaSet and scales down the old one
11. Delete the ReplicaSet directly and observe what happens
12. Delete the Deployment

**Solution**

**Step 1  Create the ReplicaSet manifest**
```bash
vi replicaset.yaml
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: httpd
        image: docker.io/httpd:2.4
```

```bash
# Step 2  Apply and verify
oc apply -f replicaset.yaml
oc get pods -w

# Step 3  Check ReplicaSet status
oc get rs
oc describe rs my-replicaset

# Step 4  Scale to 5 replicas
oc scale rs my-replicaset --replicas=5
oc get pods

# Step 5  Scale down to 2 via manifest
vi replicaset.yaml
# Change: replicas: 3 �� replicas: 2
oc apply -f replicaset.yaml
oc get pods

# Step 6  Delete one pod manually
oc get pods
oc delete pod <pod-name>
oc get pods
# ReplicaSet recreates the pod automatically 

# Step 7  Create a Deployment
oc create deployment my-app-deployment \
  --image=docker.io/bitnami/nginx \
  --replicas=3

# Step 8  Observe the ReplicaSet
oc get rs
oc get all

# Step 9  Update the image
oc set image deployment/my-app-deployment nginx=docker.io/nginx

# Step 10  Observe new ReplicaSet
oc get rs
# New ReplicaSet scales up, old ReplicaSet scales to 0 

# Step 11  Delete the ReplicaSet directly
oc get rs
oc delete rs <replicaset-name>
oc get rs
# Deployment immediately recreates the ReplicaSet 

# Step 12  Delete the Deployment
oc delete deployment my-app-deployment
oc delete rs my-replicaset
```

---

<a id="lab-13-labels-and-selectors"></a>
## Lab 13  Labels and Selectors

**Objective:** Assign labels to resources, query with selectors and manage labels dynamically

**Task details:**
1. Create a Pod manifest named label-pod.yaml with labels app=my-app, environment=production and tier=frontend
2. Deploy the pod and verify the labels are assigned
3. Query pods by label app=my-app
4. Create a Deployment manifest named selector-deployment.yaml that uses a selector for label app=my-app
5. Update the environment label on the pod to staging
6. Verify the updated label
7. Remove the tier label from the pod
8. Verify the tier label is gone

**Solution**

**Step 1  Create the Pod manifest**
```bash
vi label-pod.yaml
```
oc run label-pod \
  --image=nginx \
  --labels="app=my-app,environment=production,tier=frontend" \
  --dry-run=client -o yaml > label-pod.yaml


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-labeled-pod
  labels:
    app: my-app
    environment: production
    tier: frontend
spec:
  containers:
  - name: httpd
    image: docker.io/httpd:2.4
```

```bash
# Step 2  Deploy and verify
oc create -f label-pod.yaml
oc get pod my-labeled-pod --show-labels

# Step 3  Query by label (equality-based)
oc get pod -l app=my-app

# Query by label (set-based)
oc get pod -l 'environment in (production,staging)'
```

**Step 4  Create Deployment with selector**
```bash
vi selector-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: httpd
        image: docker.io/httpd:2.4
```

```bash
oc apply -f selector-deployment.yaml

# Step 5  Update environment label
oc label pod my-labeled-pod environment=staging --overwrite=true

# Step 6  Verify updated label
oc describe pod my-labeled-pod | grep Labels

# Step 7  Remove tier label
oc label pod my-labeled-pod tier-

# Step 8  Verify tier label is gone
oc describe pod my-labeled-pod | grep Labels
```

---

<a id="lab-14-configure-services"></a>
## Lab 14  Configure Services

**Objective:** Deploy an application, expose it as a service and create a route for external access

**Task details:**

1. Create a new project called httpdemo
2. Deploy an application named my-http-app using image docker.io/bitnami/nginx
3. Verify the pods are running
4. Expose the application as a service
5. Verify the service is created
6. Create a route with hostname my-app.httpdemo.apps-crc.testing
7. Verify the route is created
8. Describe the route to see the details
9. Test the route by curling the hostname

**Solution**

```bash
# Step 1  Create the project
oc new-project httpdemo

# Step 2  Deploy the application
oc create deployment my-http-app --image=docker.io/bitnami/nginx

# Step 3  Verify pods
oc get pods -w

# Step 4  Expose the application as a service
oc expose deployment my-http-app --port=8080

# Step 5  Verify the service
oc get service

# Step 6  Create a route
oc expose service my-http-app --hostname=my-app.httpdemo.apps-crc.testing

# Step 7  Verify the route
oc get route

# Step 8  Describe the route
oc describe route my-http-app

# Step 9  Test the route
curl http://my-app.httpdemo.apps-crc.testing
```

---
<a id="lab-15-manage-secrets"></a>
## Lab 15  Manage Secrets

**Objective:** Create, inspect and use OpenShift Secrets to securely manage sensitive application data

**Task details:**
1. Create a new project called `secrets-demo`
2. Create a generic Secret called `my-secret` with `username=myuser` and `password=mypassword`
3. Verify the Secret and inspect its keys
4. Create `username.txt` and `password.txt` and create a Secret called `my-file-secret` from these files
5. Create a Pod called `secret-pod` using the `busybox` image and inject the values from `my-secret` as environment variables
6. Verify the Secret values using the Pod logs
7. Create a Pod called `secret-volume-pod` and mount `my-secret` as files under `/etc/secrets`
8. Verify the Secret values from the mounted files
9. Create a Docker registry Secret called `registry-secret`
10. Create a TLS Secret called `my-tls-secret` using `tls.crt` and `tls.key`
11. Display `my-secret` as YAML and inspect the Base64 encoded values
12. Decode one of the Base64 encoded Secret values

**Solution**

```bash
# Step 1  Create the project
oc new-project secrets-demo

# Step 2  Create a generic Secret
oc create secret generic my-secret \
  --from-literal=username=myuser \
  --from-literal=password=mypassword

# Step 3  Verify the Secret
oc get secret my-secret
oc describe secret my-secret

# Step 4  Create a Secret from files
echo -n 'myuser' > username.txt
echo -n 'mypassword' > password.txt

oc create secret generic my-file-secret \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Verify
oc describe secret my-file-secret

# Step 5  Create a Pod using Secret environment variables
vi secret-pod.yaml
```

`secret-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: secret-container
    image: busybox
    command:
    - sh
    - -c
    - "echo Username: $USERNAME; echo Password: $PASSWORD"
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
```

```bash
# Create the Pod
oc apply -f secret-pod.yaml

# Step 6  Verify the Secret values
oc get pod secret-pod
oc logs secret-pod

# Step 7  Create a Pod with the Secret mounted as files
vi secret-volume-pod.yaml
```

`secret-volume-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: secret-container
    image: busybox
    command:
    - sh
    - -c
    - "cat /etc/secrets/username; echo; cat /etc/secrets/password"
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```

```bash
# Create the Pod
OR

oc run secret-volume-pod --image=bitnami/nginx --dry-run=client -o yaml | oc set volume -f -
--add --name=secret-volume --type=secret --secret-name=my-secret --mount-path=/etc/secrets --local -o yaml > secret-volume-pod.yaml

oc apply -f secret-volume-pod.yaml

# Step 8  Verify the mounted files
oc logs secret-volume-pod

# Step 9  Create a Docker registry Secret
oc create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword

# Verify
oc get secret registry-secret

# Step 10  Create a TLS Secret
oc create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Verify
oc get secret my-tls-secret

# Step 11  Inspect Secret data
oc get secret my-secret -o yaml

# Step 12  Decode a Base64 encoded value
echo 'bXl1c2Vy' | base64 -d
# Output: myuser
```

### Key Concepts

```text
oc create secret generic
    �� Generic Secret

--from-literal
    �� Secret value directly from command line

--from-file
    �� Secret value from file

secretKeyRef
    �� Specific Secret key as environment variable

volumes.secret
    �� Secret mounted as files

oc create secret docker-registry
    �� Registry credentials

oc create secret tls
    �� TLS certificate and key

Base64
    �� Encoding, not encryption
```

---

<a id="lab-16-manage-configuration-maps"></a>
## Lab 16  Manage Configuration Maps

**Objective:** Create and use OpenShift ConfigMaps to manage non-sensitive application configuration

**Task details:**
1. Create a new project called `config-demo`
2. Create a ConfigMap called `my-config` with `APP_ENV=production` and `APP_DEBUG=false`
3. Verify the ConfigMap and inspect its data
4. Create a file called `index.html` containing a custom web page
5. Create a ConfigMap called `nginx-config` from the `index.html` file
6. Verify the ConfigMap contains the `index.html` key
7. Create an nginx Pod called `cm-nginx` using the `bitnami/nginx` image
8. Mount `nginx-config` as a volume at `/app`
9. Verify that `index.html` is available inside the Pod
10. Expose the Pod and use port forwarding to access the custom web page
11. Create a Pod named var-pod using the bitnami/nginx image and configure it to use all key-value pairs from the my-config ConfigMap as environment             variables.
12. Explain the difference between ConfigMaps and Secrets

**Solution**

```bash
# Step 1  Create the project
oc new-project config-demo

# Step 2  Create a ConfigMap
oc create configmap my-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false

# Step 3  Verify the ConfigMap
oc get configmap my-config
oc describe configmap my-config

# Step 4  Create the custom web page
echo '<html><body><h1>Hello from OpenShift ConfigMap</h1></body></html>' > index.html

# Step 5  Create a ConfigMap from the file
oc create configmap nginx-config \
  --from-file=index.html

# Step 6  Verify the ConfigMap
oc get configmap nginx-config
oc describe configmap nginx-config

# Step 7-8  Create the nginx Pod and mount the ConfigMap
oc run cm-nginx --image=bitnami/nginx --dry-run=client -o yaml | oc set volume -f - --add --name=nginx-config-vol --type=configmap --configmap-name=nginx-config --mount-path=/app --local -o yaml > cm-nginx.yaml


```

```bash
# Create the Pod
oc apply -f cm-nginx-pod.yaml

# Step 9  Verify the mounted file
oc get pod cm-nginx
oc exec cm-nginx -- ls -l /app
oc exec cm-nginx -- cat /app/index.html

# Step 10  Port forward to the nginx Pod
oc port-forward pod/cm-nginx 8080:8080
```

Open:

```text
http://localhost:8080
```

```bash
# Step 11  Use the ConfigMap as environment variables
oc run cm-env-pod --image=bitnami/nginx --env-from-configmap=my-config --dry-run=client -o yaml > cm-env-pod.yaml

# Verify
oc logs cm-env-pod

# Step 12  Inspect the ConfigMap
oc get configmap my-config -o yaml
```

### Key Concepts

```text
oc create configmap
    �� Create a ConfigMap

--from-literal
    �� Configuration value directly from command line

--from-file
    �� Configuration from a file

envFrom
    �� Import ConfigMap keys as environment variables

configMap volume
    �� Mount ConfigMap data as files

ConfigMap
    �� Non-sensitive configuration

Secret
    �� Sensitive data such as passwords and credentials

Base64
    �� ConfigMaps store data as plain text
```

---

<a id="lab-17-mount-storage-pv-pvc-deployment-route"></a>
## Lab 17  Mount Storage (PV, PVC, Deployment & Route)

**Objective:** Create and use persistent storage with PersistentVolumes (PV), PersistentVolumeClaims (PVC), Deployments and Routes in OpenShift using an NFS backend.

**Task details:**

1. Inspect the existing NFS-backed StorageClass and determine its details and Reclaim Policy.
2. Create a new project named `webserver`.
3. Create a PersistentVolume named `web-pv` with the following specifications:
   - Access Mode: `ReadOnlyMany`
   - Size: `1Gi`
   - Reclaim Policy: `Retain`
   - NFS Path: `/exports/nfs-share` (or `/exports-ocp4/page`)
4. Create a PersistentVolumeClaim named `web-pvc` in the `webserver` project with:
   - Size: `1Gi`
   - Access Mode: `ReadOnlyMany`
5. Verify that the PVC is successfully bound to the PV.
6. Deploy an application with the following requirements:
   - Project: `webserver`
   - Deployment name: `web-landing`
   - Container image: `nginx:alpine`
   - Replicas: `3`
   - Mount `web-pvc` at `/usr/share/nginx/html`.
7. Expose the Deployment via a Service and a secure Edge Route accessible at:
   `https://exoplanets-declarative-manifests.apps.ocp4.example.com`
8. Verify that all 3 Pods are running and that the web application responds correctly.

**Solution**

```bash
# Step 1  Inspect the StorageClass
oc get storageclass
oc describe storageclass <storageclass-name>

# Step 2  Create the project
oc new-project webserver

# Step 3  Create the PersistentVolume
vi web-pv.yaml
```

**web-pv.yaml**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: web-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: <NFS_SERVER_IP>
    path: /exports/nfs-share
```

```bash
# Apply the PersistentVolume
oc apply -f web-pv.yaml

# Verify
oc get pv web-pv

# Step 4  Create the PersistentVolumeClaim
vi web-pvc.yaml
```

**web-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-pvc
  namespace: webserver
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
```

```bash
# Apply the PersistentVolumeClaim
oc apply -f web-pvc.yaml

# Step 5  Verify the PVC is bound
oc get pvc web-pvc -n webserver
oc get pv web-pv

# Step 6  Create the Deployment
vi deployment.yaml
```

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-landing
  namespace: webserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-landing
  template:
    metadata:
      labels:
        app: web-landing
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html-volume
          persistentVolumeClaim:
            claimName: web-pvc
```

```bash
# Apply the Deployment
oc apply -f deployment.yaml

# Verify the Deployment and Pods
oc get deployment web-landing -n webserver
oc get pods -n webserver

# Step 7  Expose via Service and secure Edge Route
oc expose deployment web-landing --port=80 --target-port=80 -n webserver

oc create route edge web-route \
  --service=web-landing \
  --hostname=exoplanets-declarative-manifests.apps.ocp4.example.com \
  -n webserver

# Step 8  Verify the Route
oc get route web-route -n webserver

# Test application access
curl -k https://exoplanets-declarative-manifests.apps.ocp4.example.com
```

**Expected output:**

```text
HTTP/1.1 200 OK
(Contents of index.html mounted from the NFS volume)
```

**Key Concepts**

- **PersistentVolume (PV):** Cluster-wide storage resource backed by NFS.
- **PersistentVolumeClaim (PVC):** Request for storage by an application in a project.
- **ReadOnlyMany (ROX):** Allows multiple nodes to mount the volume as read-only.
- **Deployment replicas:** Three application Pods sharing the same NFS web root.
- **Edge Route:** Exposes the application externally with TLS termination at the OpenShift Router.


---

<a id="lab-18-storage-classes"></a>
## Lab 18  Storage Classes

**Objective:** Use StorageClasses to dynamically provision persistent storage in OpenShift

**Task details:**
1. Create a new project called `storageclass-demo`
2. List the available StorageClasses
3. Identify the default StorageClass
4. Create a PersistentVolumeClaim called `my-hostpath-claim` using the default StorageClass
5. Request 1Gi of storage with `ReadWriteOnce` access
6. Verify that the PVC is created and becomes bound
7. Verify that OpenShift dynamically created a PersistentVolume
8. Create a Pod called `storage-pod` using the `bitnami/nginx` image
9. Mount the PVC at `/data`
10. Create a file in `/data` and verify that it is stored on the persistent volume
11. Inspect the StorageClass, PVC and PV
12. Explain the difference between a manually created PV and dynamically provisioned storage

**Solution**

```bash
# Step 1  Create the project
oc new-project storageclass-demo

# Step 2  List the available StorageClasses
oc get storageclass

# Step 3  Identify the default StorageClass
oc get storageclass

# Step 4-5  Create a PVC using the default StorageClass
vi pvc.yaml
```

`pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-hostpath-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
# Create the PVC
oc apply -f pvc.yaml

# Step 6  Verify the PVC
oc get pvc

# Step 7  Verify the dynamically created PV
oc get pv

# Step 8-9  Create a Pod using the PVC
vi storage-pod.yaml
```

`storage-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: nginx
    image: bitnami/nginx
    command:
    - sh
    - -c
    - "touch /data/storage-test.txt && sleep 3600"
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-hostpath-claim
```

```bash
# Create the Pod
oc apply -f storage-pod.yaml

# Verify the Pod
oc get pod storage-pod

# Step 10  Create and verify a file
oc exec storage-pod -- sh -c 'echo "StorageClass works" > /data/storage-test.txt'
oc exec storage-pod -- cat /data/storage-test.txt

# Step 11  Inspect the StorageClass, PVC and PV
oc get storageclass
oc describe pvc my-hostpath-claim
oc get pv
```

### Key Concepts

```text
StorageClass
    �� Defines how storage is dynamically provisioned

PVC
    �� Requests storage from a StorageClass

StorageClass
    ��
PVC
    ��
Dynamic PV
    ��
Pod

Static provisioning
    �� Administrator creates PV manually

Dynamic provisioning
    �� PVC requests storage
    �� StorageClass automatically provisions the PV
```

---

<a id="lab-19-statefulsets"></a>
## Lab 19  StatefulSets

**Objective:** Deploy and manage stateful applications using StatefulSets with stable identities and dedicated persistent storage

**Task details:**
**Preparation**

1. Create a project called `stateful-demo`.
2. Create a headless Service called `mysql` with `clusterIP: None`.

**StatefulSet**

3. Create a StatefulSet called `mysql` with the following properties:
   - 2 replicas
   - image `bitnami/mysql`
   - `serviceName: mysql` (uses the headless Service from step 2)
   - a `volumeClaimTemplate` requesting 1Gi of storage for each Pod
4. Apply the Service and the StatefulSet.

**Verification**

5. Verify the StatefulSet, the Pods and the Service.
6. Verify that each Pod has its own PersistentVolumeClaim.

**Scaling and recovery**

7. Scale the StatefulSet to 3 replicas and verify that the new Pod receives its own PersistentVolumeClaim.
8. Delete a Pod and verify that it is recreated with the same name.
**Solution**

```bash
# Step 1  Create the project
oc new-project stateful-demo

# Step 2  Create the headless Service
vi mysql-service.yaml
```

`mysql-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

```bash
# Create the Service
oc apply -f mysql-service.yaml

# Step 3-6  Create the StatefulSet
vi mysql-statefulset.yaml
```

`mysql-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: bitnami/mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /bitnami/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

```bash
# Step 7  Create the StatefulSet
oc apply -f mysql-statefulset.yaml

# Step 8  Verify the StatefulSet, Pods and Service
oc get statefulset
oc get pods
oc get service

# Step 9  Verify the PVCs and PVs
oc get pvc
oc get pv

# Step 10  Scale the StatefulSet
oc scale statefulset mysql --replicas=3

# Step 11  Verify the new Pod and PVC
oc get pods
oc get pvc

# Step 12  Delete a Pod
oc delete pod mysql-0

# Verify that the Pod is recreated with the same name
oc get pods
```

### Key Concepts

```text
Deployment
    �� Stateless applications
    �� Pods have replaceable identities

StatefulSet
    �� Stateful applications
    �� Pods have stable identities

StatefulSet
    ��
mysql-0
mysql-1
mysql-2

volumeClaimTemplates
    ��
One PVC per Pod

Headless Service
    �� clusterIP: None
    �� Stable network identities

StatefulSet
    �� Stable Pod names
    �� Ordered creation/scaling
    �� Dedicated storage per Pod
```

<a id="lab-20-mount-storage-pv-and-pvc"></a>
## Lab 20  Mount Storage  PV and PVC

## Task

An NFS backend StorageClass is already configured.

Create a PersistentVolume (PV):

- Name: `web-pv`
- Size: `1Gi`
- Access Mode: `ReadOnlyMany`
- Reclaim Policy: same as the existing StorageClass
- StorageClass: use the existing NFS StorageClass
- NFS path: `/exports-ocp4/page`

Create a PersistentVolumeClaim (PVC):

- Project: `webserver`
- Name: `web-pvc`
- Size: `1Gi`
- Access Mode: `ReadOnlyMany`
- StorageClass: same as the PV

Deploy:

- Deployment: `web-landing`
- Image: `registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0`
- Replicas: `3`
- Mount PVC at `/usr/share/nginx/html`
- Create a Service and Route
- Hostname:
  `exoplanets-declarative-manifests.apps.ocp4.example.com`

**Solution**

```bash
# Step 1  Create project
oc new-project webserver

# Step 2  Inspect the existing StorageClass
oc get storageclass
oc describe storageclass <nfs-storageclass>

# Step 3  Create a temporary PVC to discover NFS details
# Use the existing/default NFS StorageClass.
# The dynamically created PV reveals:
# - NFS server
# - NFS export path
# - volumeMode
# - reclaimPolicy

oc get pvc
oc get pv
oc get pv <temporary-pv> -o yaml

# Step 4  Create the required PV
vi web-pv.yaml

# Use the NFS server, path, volumeMode and reclaimPolicy
# discovered from the temporary PV.

# Step 5  Apply the PV
oc apply -f web-pv.yaml

# Step 6  Create the PVC
vi web-pvc.yaml

# Step 7  Apply the PVC
oc apply -f web-pvc.yaml

# Step 8  Verify PV/PVC binding
oc get pv
oc get pvc

# Step 9  Create the Deployment
oc create deployment web-landing \
  --image=registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0

# Step 10  Scale to 3 replicas
oc scale deployment web-landing --replicas=3

# Step 11  Mount the PVC
oc set volume deployment/web-landing \
  --add \
  --name=web-content \
  --type=persistentVolumeClaim \
  --claim-name=web-pvc \
  --mount-path=/usr/share/nginx/html \
  --read-only

# Step 12  Create Service
oc expose deployment web-landing

# Step 13  Create Route
oc create route edge web-landing-route \
  --service=web-landing \
  --hostname=exoplanets-declarative-manifests.apps.ocp4.example.com

# Step 14  Verify
oc get pv
oc get pvc
oc get deployment
oc get pods
oc get svc
oc get route
```

<a id="lab-21-serviceaccounts-and-scc"></a>
## Lab 21  ServiceAccounts and SCC

**Objective:** Configure a ServiceAccount with permission to run Pods as any user and deploy an application using that ServiceAccount.

**Task details:**

1. Create a ServiceAccount named `redhat-sa` in the `production` project.
2. Allow the ServiceAccount to use the `anyuid` SCC.
3. Configure the existing application `team` in the `production` project to use the `redhat-sa` ServiceAccount.
4. Do not add or remove any configuration components from the application.
5. Ensure the application produces output when accessed.

**Solution**

```bash
# Create the ServiceAccount
oc create serviceaccount redhat-sa -n production

# Allow the ServiceAccount to use anyuid
oc adm policy add-scc-to-user anyuid -z redhat-sa -n production

# Configure the existing application to use redhat-sa
oc set serviceaccount deployment/team redhat-sa -n production

# Verify the rollout
oc rollout status deployment/team -n production

# Verify the ServiceAccount
oc get deployment team -n production \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'

# Verify the SCC permission
oc auth can-i use scc/anyuid \
  --as=system:serviceaccount:production:redhat-sa

# Check the application resources
oc get pods,svc,routes -n production
```

**Success criteria:** The `redhat-sa` ServiceAccount exists, is allowed to use `anyuid`, and the existing `team` Deployment uses that ServiceAccount without adding or removing application configuration components.

<a id="lab-22-install-and-configure-an-operator-file-integrity-operator"></a>
## Lab 22  Install and Configure an Operator (File Integrity Operator)

**Objective:** Install and configure the File Integrity Operator in OpenShift.

**Task details:**
1. Install the File Integrity Operator.
2. Install the Operator in the `openshift-integrity` project.
3. Set the approval strategy to `Automatic`.
4. Enable cluster monitoring for the `openshift-file-integrity` project.
5. Verify that the Operator is successfully installed.

**Solution**

```bash
# Step 1  Create the Operator project
oc new-project openshift-integrity

# Step 2  Create the project for File Integrity monitoring
oc new-project openshift-file-integrity

# Step 3  Enable cluster monitoring
oc label namespace openshift-file-integrity \
  openshift.io/cluster-monitoring=true
```

**Step 4  Install the File Integrity Operator**

1. Open the OpenShift Web Console.
2. Navigate to **Ecosystem �� Software Catalog (OperatorHub)**.
3. Search for `File Integrity Operator`.
4. Select the Operator and click **Install**.
5. Configure the installation:
   - Installed Namespace: `openshift-integrity`
   - Update Approval: `Automatic`
6. Click **Install**.

**Step 5  Verify the installation**

```bash
# Check the Subscription
oc get subscriptions -n openshift-integrity

# Check the ClusterServiceVersion
oc get csv -n openshift-integrity

# Check Operator Pods
oc get pods -n openshift-integrity

# Verify cluster monitoring label
oc get namespace openshift-file-integrity --show-labels
```

**Key Concepts**

- **OperatorHub:** Catalog for discovering and installing OpenShift Operators.
- **Subscription:** Defines the Operator package, channel and update approval strategy.
- **Automatic approval:** Automatically approves Operator update InstallPlans.
- **ClusterServiceVersion (CSV):** Represents an installed Operator version.
- **Cluster monitoring:** Enables monitoring of user workloads in a namespace.
- **Namespace label:** `openshift.io/cluster-monitoring=true` enables monitoring for the namespace.

<a id="lab-23-networking-expose-applications-through-a-secure-route"></a>
## Lab 23  Networking: Expose Applications through a Secure Route

**Objective:** Deploy an application in OpenShift and secure external access using an Edge Route with a CA-signed SSL certificate.

**Task details:**
1. Create a project named `business`.
2. Create a Deployment named `marketing`.
3. Use the image `nginxinc/nginx-unprivileged:stable`.
4. Expose the Deployment through a Service.
5. Create an unsecured Route named `marketing-route`.
6. Generate an SSL certificate and private key using the provided `cert-manager.sh` script.
7. The certificate must be CA-signed and use the following subject:
   `/C=US/ST=NV/L=California/O=Hat/OU=IT/CN=webapplication.apps.domain1.exampledomain.com`
8. Configure the existing Route `marketing-route` with Edge TLS termination.
9. The application must be accessible only through:
   `https://webapplication.apps.domain1.exampledomain.com`
10. Ensure that client-to-route communication uses HTTPS and router-to-backend communication remains HTTP.

**Solution**

```bash
# Step 1  Create the project
oc new-project business

# Step 2  Create the Deployment
oc create deployment marketing \
  --image=nginxinc/nginx-unprivileged:stable

# Step 3  Expose the Deployment through a Service
oc expose deployment marketing --port=8080 --target-port=8080

# Step 4  Create an unsecured Route
oc create route edge marketing-route \
  --service=marketing \
  --port=8080 \
  --hostname=webapplication.apps.domain1.exampledomain.com

# Step 5  Verify the Route
oc get route marketing-route
```

**Step 6  Generate the SSL certificate**

Use the provided `cert-manager.sh` script to generate a CA-signed certificate and private key.

The certificate must contain the following subject:

```text
/C=US/ST=NV/L=California/O=Hat/OU=IT/CN=webapplication.apps.domain1.exampledomain.com
```

Check the script's usage instructions to determine the required arguments and output filenames.

For this lab, assume the generated files are:

- `tls.crt`  CA-signed certificate
- `tls.key`  Private key

If the script also provides a CA certificate, keep that file available.

**Step 7  Configure the Route with a CA-signed certificate using the Web Console**

1. Log in to the OpenShift Web Console.
2. Select the `business` project.
3. Navigate to **Networking �� Routes**.
4. Find the Route named `marketing-route`.
5. Open the Route and select **Actions �� Edit Route**.
6. Locate the **Security** or **TLS Settings** section.
7. Enable **Secure route**.
8. Configure the following settings:
   - **TLS Termination:** `Edge`
   - **Insecure Traffic:** `Redirect`
9. Open the generated certificate file `tls.crt` on your workstation and copy its complete contents.
10. Paste the certificate into the **Certificate** field.
11. Open the private key file `tls.key` and copy its complete contents.
12. Paste the private key into the **Private Key** field.
13. If the certificate was signed by a separate CA, copy the CA certificate into the **CA Certificate** field.
14. Save the Route configuration.

**Step 8  Verify the Route**

Navigate to **Networking �� Routes** and open `marketing-route`.

Verify:
- The Route is secure.
- TLS termination is set to `Edge`.
- HTTP traffic is redirected to HTTPS.
- The Route uses the correct hostname.

**Step 9  Test HTTPS access**

```bash
curl -k https://webapplication.apps.domain1.exampledomain.com
```
Expected result: The Nginx welcome page is returned over HTTPS.

### Key Concepts

- **Route:** Exposes an application outside the OpenShift cluster.
- **Edge TLS termination:** TLS is terminated at the OpenShift Router.
- **Backend HTTP:** Communication between the Router and application remains unencrypted HTTP.
- **Certificate:** Public certificate used to secure the HTTPS connection.
- **Private key:** Used by the Router for TLS encryption.
- **CA-signed certificate:** Certificate signed by a trusted Certificate Authority.
- **InsecureEdgeTerminationPolicy: Redirect:** Redirects HTTP requests to HTTPS.

<a id="lab-24-automatically-scale-an-application-horizontal-pod-autoscaler"></a>
## Lab 24  Automatically Scale an Application (Horizontal Pod Autoscaler)

**Objective:** Configure horizontal pod autoscaling (HPA) to automatically scale an application based on CPU utilization.

**Task details:**
1. Use an existing OpenShift project or create a new project.
2. Create a Deployment named `titan`.
3. Configure the CPU request to `50m` per Pod.
4. Configure the CPU limit to `250m` per Pod.
5. Configure horizontal pod autoscaling for the `titan` Deployment with:
   - Minimum number of Pods: `2`
   - Maximum number of Pods: `5`
   - Target average CPU utilization: `75%`
6. Verify the HPA configuration and scaling status.

**Solution**

```bash
# Step 1  Select an existing project
oc project <project-name>

# Alternatively, create a new project
oc new-project autoscale-demo

# Step 2  Create the Deployment
oc create deployment titan --image=nginxinc/nginx-unprivileged:stable

# Step 3  Configure CPU requests and limits
oc set resources deployment titan \
  --requests=cpu=50m \
  --limits=cpu=250m

# Step 4  Configure horizontal pod autoscaling
oc autoscale deployment titan \
  --min=2 \
  --max=5 \
  --cpu-percent=75

# Step 5  Verify the Deployment
oc get deployment titan

# Step 6  Verify the HPA
oc get hpa titan

# View detailed HPA information
oc describe hpa titan

# Verify the Pods
oc get pods
```

**Expected result:**

The HPA should show:

```text
NAME    REFERENCE           TARGETS   MINPODS   MAXPODS
titan   Deployment/titan    0%/75%    2         5
```

The CPU utilization may initially show `<unknown>` while metrics are being collected.

### Key Concepts

- **Horizontal Pod Autoscaler (HPA):** Automatically adjusts the number of Pod replicas based on resource utilization.
- **Minimum replicas:** Minimum number of Pods.
- **Maximum replicas:** Maximum number of Pods.
- **CPU request:** CPU amount used as the reference for calculating utilization.
- **CPU limit:** Maximum CPU amount a container can use.
- **CPU utilization target:** Average CPU utilization relative to the configured CPU request.
- **oc autoscale:** Creates an HPA for an existing Deployment.


---

<a id="lab-25-configure-a-project-template-with-a-limitrange"></a>
## Lab 25  Configure a Project Template with a LimitRange

**Objective:** Configure OpenShift so that every newly created project automatically receives a LimitRange with predefined memory constraints.

**Task details:**

1. Generate the default OpenShift Project Template.
2. Save it as `project-template.yaml`.
3. Generate a LimitRange YAML using the Web Console.
4. Add the LimitRange to the Project Template.
5. Set the LimitRange name to `<PROJECT_NAME>-limits`.
6. Apply the Project Template in the `openshift-config` namespace.
7. Configure OpenShift to use the Project Template automatically.
8. Monitor the OpenShift API Server pods while the configuration is being applied.
9. Create a new test project.
10. Verify that the LimitRange is automatically created.

**Solution**

**Step 1  Generate the Project Template**

```bash
oc adm create-bootstrap-project-template -o yaml > project-template.yaml
```

**Step 2  Edit the Project Template**

```bash
vi project-template.yaml
```

Ensure the template name is:

```yaml
metadata:
  name: project-request
```

Do not remove the existing objects or RoleBindings.

**Step 3  Create the LimitRange using the Web Console**

1. Switch to the **Administrator** view.
2. Navigate to **Administration �� Limit Ranges**.
3. Use the YAML editor to generate a LimitRange.
4. Configure the following requirements:

| Setting | Value |
|---|---|
| Type | Container |
| Minimum memory | `128Mi` |
| Maximum memory | `256Mi` |
| Default memory limit | `256Mi` |
| Default memory request | `256Mi` |

The generated LimitRange YAML should look like this:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: default
spec:
  limits:
  - default:
      memory: 256Mi
    defaultRequest:
      memory: 256Mi
    type: Container
    min:
      memory: 128Mi
    max:
      memory: 256Mi
```

Copy this YAML. Do not create the LimitRange directly in the project. It will be added to the Project Template instead.

**Step 4  Add the LimitRange to the Project Template**

Open `project-template.yaml` and find the `objects:` section.

Leave the existing RoleBinding objects untouched. Add the LimitRange as another object in the `objects` list:

```yaml
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: mem-limit-range
    namespace: default
  spec:
    limits:
    - default:
        memory: 256Mi
      defaultRequest:
        memory: 256Mi
      type: Container
      min:
        memory: 128Mi
      max:
        memory: 256Mi
```

**Step 5  Replace the LimitRange name and namespace**

Change the LimitRange metadata to use the project name variable:

```yaml
metadata:
  name: ${PROJECT_NAME}-limits
  namespace: ${PROJECT_NAME}
```

The complete LimitRange object should now look like this:

```yaml
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: ${PROJECT_NAME}-limits
    namespace: ${PROJECT_NAME}
  spec:
    limits:
    - default:
        memory: 256Mi
      defaultRequest:
        memory: 256Mi
      type: Container
      min:
        memory: 128Mi
      max:
        memory: 256Mi
```

OpenShift replaces `${PROJECT_NAME}` with the name of the newly created project.

**Step 6  Save and apply the Project Template**

```bash
oc apply -f project-template.yaml -n openshift-config
```

Verify the template:

```bash
oc get templates -n openshift-config
```

The template should be named `project-request`.

**Step 7  Configure OpenShift to use the Project Template**

Edit the cluster project configuration:

```bash
oc edit projects.config.openshift.io cluster
```

Find:

```yaml
spec: {}
```

Replace it with:

```yaml
spec:
  projectRequestTemplate:
    name: project-request
```

Save and exit.

This instructs OpenShift to use the `project-request` template whenever a new project is created through the project request mechanism.

**Step 8  Monitor the OpenShift API Server pods**

The configuration change may trigger an update of the OpenShift API Server. Monitor the pods:

```bash
oc get pods -n openshift-apiserver -w
```

Wait until the API Server pods are running and ready again. Press `Ctrl+C` to stop monitoring.

**Step 9  Create a new test project**

```bash
oc new-project sample-project
```

**Step 10  Verify the automatically created LimitRange**

```bash
oc get limitrange
```

Expected LimitRange name:

```text
sample-project-limits
```

Inspect the LimitRange:

```bash
oc describe limitrange sample-project-limits
```

Verify the following values:

| Setting | Expected value |
|---|---|
| Minimum memory | `128Mi` |
| Maximum memory | `256Mi` |
| Default memory limit | `256Mi` |
| Default memory request | `256Mi` |

### Key Concepts

| Command / Concept | Purpose |
|---|---|
| `oc adm create-bootstrap-project-template` | Generates the default Project Template. |
| `${PROJECT_NAME}` | Variable containing the name of the newly created project. |
| `oc apply -f project-template.yaml -n openshift-config` | Creates or updates the Project Template. |
| `oc edit projects.config.openshift.io cluster` | Configures the cluster to use the Project Template. |
| `projectRequestTemplate` | Specifies which template OpenShift uses for new project requests. |
| `LimitRange` | Defines minimum, maximum and default resource values. |

**EX280 Exam Tips:**

- Creating the Project Template alone is not sufficient. You must also configure OpenShift to use it.
- The template must be stored in the `openshift-config` namespace.
- The template name must match the name configured in `projectRequestTemplate`.
- The `${PROJECT_NAME}` variable ensures that every new project gets its own LimitRange name.
- Existing projects are not automatically updated. The template applies to new project requests.
- A LimitRange controls individual container or Pod resource constraints; a ResourceQuota controls total resource consumption within a project.

**Remember the sequence:**

1. Generate the bootstrap template.
2. Generate the LimitRange YAML.
3. Add the LimitRange to the template.
4. Apply the template.
5. Configure `projectRequestTemplate`.
6. Create a new project and verify the result.

<a id="lab-26-configure-network-policies"></a>
## Lab 26  Configure Network Policies

**Objective:** Configure network traffic restrictions between OpenShift projects using NetworkPolicies.

**Task details:**

An application is running in two OpenShift projects:

- `frontend`
- `backend`

The backend project contains a MySQL application that must only be accessible from an authorized frontend application.

Configure a NetworkPolicy with the following requirements:

1. Create a NetworkPolicy named `db-allow-conn` in the `backend` project.
2. Use the Pod selector `network.openshift.io/policy-group=backend` to select the backend Pods.
3. Allow incoming traffic only from Pods in namespaces with the label `team=security`.
4. Additionally, restrict access to Pods with the label `deployment=sc`.
5. Allow only TCP traffic on port `3306`.
6. Verify the configuration by checking the application logs in the `frontend` project.

**Preparation:**

Use the separate setup script to prepare the frontend and backend projects and deploy the applications. The script also creates the default-deny NetworkPolicy.

**Solution**

**Step 1  Open the OpenShift Web Console**

1. Log in to the OpenShift Web Console.
2. Switch to the **Administrator** perspective.
3. Navigate to **Networking �� NetworkPolicies**.
4. Select the `backend` project.
5. Click **Create NetworkPolicy**.

**Step 2  Create the NetworkPolicy**

Configure the following:

| Setting | Value |
|---|---|
| Name | `db-allow-conn` |
| Project | `backend` |
| Policy type | Ingress |
| Pod selector | `network.openshift.io/policy-group=backend` |

**Step 3  Configure the Ingress rule**

Configure the source selectors:

Namespace selector:

| Key | Value |
|---|---|
| `team` | `security` |

Pod selector:

| Key | Value |
|---|---|
| `deployment` | `sc` |

Configure the allowed port:

| Setting | Value |
|---|---|
| Protocol | TCP |
| Port | `3306` |

Save the NetworkPolicy.

**Step 4  Verify the configuration**

Navigate to **Networking �� NetworkPolicies** in the `backend` project and inspect `db-allow-conn`.

Verify that the Pod selector, namespace selector, Pod selector and port match the task requirements.

**Step 5  Verify application connectivity**

1. Navigate to **Workloads �� Deployments**.
2. Select project `frontend`.
3. Open Deployment `frontend-checker`.
4. Open its Pod.
5. Select the **Logs** tab.
6. Verify whether the frontend application can connect to MySQL.

Alternatively, use:

```bash
oc logs deployment/frontend-checker -n frontend
```

### Key Concepts

- **NetworkPolicy:** Controls network traffic to and from selected Pods.
- **Pod selector:** Determines which Pods the policy protects.
- **Namespace selector:** Selects namespaces based on labels.
- **Ingress:** Controls incoming network traffic.
- **NetworkPolicies are additive:** Multiple policies can allow traffic to the same Pod.

**EX280 Exam Tips:**

- Namespace and Pod selectors in the same `from` entry are combined with AND logic.
- A NetworkPolicy only applies to Pods selected by its `podSelector`.
- Always verify namespace labels and Pod labels.
- A default-deny policy does not override an explicit allow rule in another applicable policy.

---

### Appendix  Cleanup script

Save as `cleanup-network-policy-lab.sh`.

```bash
#!/bin/bash
set -e

echo "Deleting frontend project..."
oc delete project frontend --ignore-not-found

echo "Deleting backend project..."
oc delete project backend --ignore-not-found

echo "Network Policy lab environment removed."
```

Run:

```bash
chmod +x cleanup-network-policy-lab.sh
./cleanup-network-policy-lab.sh
```
<a id="lab-27-configure-liveness-probes"></a>
## Lab 27  Configure Liveness Probes

**Objective:** Configure a liveness probe for an existing application Deployment in OpenShift.

**Task details:**

An application named `cart` is deployed in the `crt-kubernetesway-dev` project using a single-container setup.

Configure a liveness probe for this container with the following requirements:

1. Use a TCP socket check.
2. Configure the probe to check port `8080`.
3. Set the initial delay to `10` seconds.
4. Set the timeout to `30` seconds.

**Solution**

**Step 1  Select the project**

```bash
oc project crt-kubernetesway-dev
```

**Step 2  Configure the liveness probe**

```bash
oc set probe deployment/cart \
  --liveness=true \
  --initial-delay-seconds=10 \
  --timeout-seconds=30 \
  --open-tcp=8080
```

**Step 3  Verify the configuration**

```bash
oc get deployment cart -o yaml
```

Check the container configuration for:

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 30
```

You can also inspect the Deployment:

```bash
oc describe deployment cart
```

### Key Concepts

| Setting | Purpose |
|---|---|
| `livenessProbe` | Checks whether the container is still healthy. |
| `tcpSocket` | Checks whether a TCP connection can be established on the specified port. |
| `port: 8080` | Port used for the health check. |
| `initialDelaySeconds: 10` | Waits 10 seconds after container startup before the first probe. |
| `timeoutSeconds: 30` | Allows up to 30 seconds for each probe to complete. |

**EX280 Exam Tips:**

- A failed liveness probe can cause Kubernetes to restart the container.
- A liveness probe checks whether the application is still running correctly.
- A readiness probe determines whether the application is ready to receive traffic.
- A startup probe gives applications additional time to start.
- Use `oc set probe` to configure probes directly from the command line.

**Remember the command:**

```bash
oc set probe deployment/NAME --liveness=true \
  --open-tcp=PORT \
  --initial-delay-seconds=SECONDS \
  --timeout-seconds=SECONDS
```
