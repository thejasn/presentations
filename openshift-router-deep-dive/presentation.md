# Openshift Router Deep Dive

Thejas N

07 June 2023

thn@redhat.com

## What is `openshift-router`?

- Generic housing for deploying routing components (nginx, haproxy,F5) on edge
- Purpose built for openshift Route API
- Deployment managed by cluster-ingress-operator
- HAProxy is current reference implementation
  - A default HAProxy config template is bundled along with the router image ([config](https://github.com/openshift/router/blob/master/images/router/haproxy/conf/haproxy-config.template))
  - Config path is passed as ENV VARIABLE in the Dockerfile during build

## Where/How does openshift-router run?

- The router pods run in `openshift-ingress` namespace
- Each router pod is linked to a `ingresscontroller`
  - OCP creates a `default` ingresscontroller
  - Every `ingresscontroller` spawns a new set of router pods
  - Currently by default replicas are set to 2 and **NOT** 3 (weird quirk due to [this](https://github.com/openshift/cluster-ingress-operator/blob/master/pkg/operator/controller/ingress/replicas.go#L12))
- Ingresscontroller config controls a bunch of router configurations
  - Tuning Options
  - HTTPHeaders
  - HTTP Error codes
- Anti-affinity and affinity policy is crucial to reduce downtime during rolling updates
  - New router pods are colocated with a pod from old replica set
- Makes use of `TopologySpreadConstraints` to spread replicas across availability zones

> oc get ingresscontroller default -n openshift-ingress-operator

```terminal3
oc get ingresscontroller default -n openshift-ingress-operator
```

> oc get pods -n openshift-ingress

```terminal4
oc get pods -n openshift-ingress
```

> oc get deployment router-default -n openshift-ingress -o=jsonpath={.spec.template.spec.affinity.podAffinity}

```terminal4
oc get deployment router-default -n openshift-ingress -o=jsonpath={.spec.template.spec.affinity.podAffinity}
```

> oc get deployment router-default -n openshift-ingress -o=jsonpath={.spec.template.spec.topologySpreadConstraints}

```terminal4
oc get deployment router-default -n openshift-ingress -o=jsonpath={.spec.template.spec.topologySpreadConstraints}
```

## Life Cycle of openshift-router

- The starting point for the `openshift-router` is the `cluster-ingress-operator`.
- The creation of `ingresscontroller's` trigger the creation of router pods.

### Openshift `cluster-ingress-operator`

- The default `ingresscontroller` is created by the CIO
- If all prerequisites (SA, roles, configmaps, certs) for the router are satisfied, router deployment is created

### Router deployment

#### Loads all router plugins

- Router is designed using the [chain of responsibility pattern](https://refactoring.guru/design-patterns/chain-of-responsibility)
- This enables adding customization via plugin interface
  ```go
    // Plugin is the interface the router controller dispatches watch events
    // for the Routes and Endpoints resources to.
    type Plugin interface {
        HandleRoute(watch.EventType, *routev1.Route) error
        HandleEndpoints(watch.EventType, *kapi.Endpoints) error
        // If sent, filter the list of accepted routes and endpoints to this set
        HandleNamespaces(namespaces sets.String) error
        HandleNode(watch.EventType, *kapi.Node) error
        Commit() error
    }
  ```
  - List of plugins that are chained
  ```go
    pkg/router/controller/extended_validator.go|18 col 6| type ExtendedValidator struct {
    pkg/router/controller/factory/factory_endpointslices_test.go|32 col 6| type endpointSlicesTestPlugin struct {
    pkg/router/controller/host_admitter.go|72 col 6| type HostAdmitter struct {
    pkg/router/controller/status.go|38 col 6| type StatusAdmitter struct {
    pkg/router/controller/status_test.go|48 col 6| type fakePlugin struct {
    pkg/router/controller/unique_host.go|30 col 6| type UniqueHost struct {
    pkg/router/template/configmanager/haproxy/blueprint_plugin.go|14 col 6| type BlueprintPlugin struct {
    pkg/router/template/plugin.go|31 col 6| type TemplatePlugin struct {
    pkg/router/template/plugin_test.go|670 col 6| type fakePlugin struct {
  ```

#### Starts global controller which executes plugin chain

- Global controller abstracts common funcs and resources used for watching
  resources like Routes, Endpoints, etc
- This controller is built using a common controller factory
  - Abstracts creation of required informers
  - Enables sharing code with route selection controller

#### Start zombie process reaper for cleanup

- Starts a goroutine to reap processes periodically if called from a pid 1 process.
- Polls the `procfs` from `/proc` for zombie process IDs

#### Wait for termination hook

- Your standard termination hook for router cleanup
- Stops invoking the reload method, and waits until no further reloads will occur.
- Invoke the reload script one final time with the ROUTER_SHUTDOWN environment variable set with true.

### Load Balancer Service (entry point for routing)

- The CIO creates a LB service in the `openshift-ingress` namespace
- The owner reference of the router's deployment is added by the CIO

```terminal5
oc get svc -n openshift-ingress

```

## Openshift Router Internals

### Plugins

- **HostAdmitter**
  - Used for admission checks
  - Runs `RouteAdmissionFunc()` which checks against blacklists (ROUTER_ALLOWED_DOMAINS) and wildcards policies
- **UniqueHost**
  - Also validates hostname
- **ExtendedValidator**
  - Additional validations done before admitting the routes
  - These are run in addition to the validations done on the openshift-apiserver ([ref](https://github.com/openshift/openshift-apiserver/blob/aac3dd5bf0547e928103a0f718ca104b1bb13930/pkg/route/apis/route/validation/validation.go#L21))
  - There is overlap between the to validations
- **StatusAdmitter**
  - High up in the chain of plugins
  - Reflects failures in other plugins if any
  - Maintains an LRU for conflicting route updates
- **TemplatePlugin**
  - Template plugin implements the plugin interface
  - Template plugin bootstraps the template router

***Note:***
***Another plugin called BlueprintPlugin exists for dynamic haproxy config manager, which out of scope.***

## Secret Monitor 

### Secret Monitor Plugin 

Jump to code walk through. 

## Questions?


