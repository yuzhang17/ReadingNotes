## KubeBuilder

### Who is this for

#### Users of Kubernetes

Including:

- The structure of Kubernetes APIs and Resources
- API versioning semantics
- Self-healing
- Garbage Collection and Finalizers
- Declarative vs Imperative APIs
- Level-Based vs Edge-Base APIs
- Resources vs Subresources

#### Kubernetes API extension developers

Including:

- How to batch multiple events into a single reconciliation call
- How to configure periodic reconciliation
- Forthcoming
  - When to use the lister cache vs live lookups
  - Garbage Collection vs Finalizers
  - How to use Declarative vs Webhook Validation
  - How to implement API versioning



#### What’s in a basic project?

##### Build Infrastructure

##### Launch Configuration

[`config/default`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/default) contains a [Kustomize base](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/config/default/kustomization.yaml) for launching the controller in a standard configuration.

Each other directory contains a different piece of configuration, refactored out into its own base:

- [`config/manager`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/manager): launch your controllers as pods in the cluster
- [`config/rbac`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/rbac): permissions required to run your controllers under their own service account



#### Every journey needs a start, every program a main

At this point, our main function is fairly simple:

- We set up some basic flags for metrics.
- We instantiate a [*manager*](https://godoc.org/sigs.k8s.io/controller-runtime/pkg/manager#Manager), which keeps track of running all of our controllers, as well as setting up shared caches and clients to the API server (notice we tell the manager about our Scheme).
- We run our manager, which in turn runs all of our controllers and webhooks. The manager is set up to run until it receives a graceful shutdown signal. This way, when we’re running on Kubernetes, we behave nicely with graceful pod termination.

Note that the Manager can restrict the namespace that all controllers will watch for resources 



Also, it is possible to use the MultiNamespacedCacheBuilder to watch a specific set of namespaces:



#### Groups and Versions and Kinds

When we talk about APIs in Kubernetes, we often use 4 terms: *groups*, *versions*, *kinds*, and *resources*.

##### Kinds and Resources

Each API group-version contains one or more API types, which we call *Kinds*. 
You’ll also hear mention of resources on occasion. A resource is simply a use of a Kind in the API.

Notice that resources are always lowercase, and by convention are the lowercase form of the Kind.



#####  how does that correspond to Go?

When we refer to a kind in a particular group-version, we’ll call it a *GroupVersionKind*, or GVK for short. Same with resources and GVR. As we’ll see shortly, each GVK corresponds to a given root Go type in a package.



#####  what’s that Scheme thing?  ????:grey_question:

The `Scheme` we saw before is simply a way to keep track of what Go type corresponds to a given GVK (don’t be overwhelmed by its [godocs](https://godoc.org/k8s.io/apimachinery/pkg/runtime#Scheme)).



#### Adding a new API

```
kubebuilder create api --group batch --version v1 --kind CronJob
```

We start out simply enough: we import the `meta/v1` API group, which is not normally exposed by itself, but instead contains metadata common to all Kubernetes Kinds.

we define types for the Spec and Status of our Kind.

we define the types corresponding to actual Kinds, `CronJob` and `CronJobList`.

That little `+kubebuilder:object:root` comment is called a marker. 

Finally, we add the Go types to the API group. This allows us to add the types in this API group to any [Scheme](https://godoc.org/k8s.io/apimachinery/pkg/runtime#Scheme).

#### Designing an API

In Kubernetes, we have a few rules for how we design APIs. Namely, all **serialized fields *must* be `camelCase`,** so we use JSON struct tags to specify this. We can also use the **`omitempty` struct tag to mark that a field should be omitted from serialization when empty**.



Fields may use most of the primitive types. Numbers are the exception: for API compatibility purposes, we accept three forms of numbers: `int32` and `int64` for integers, and `resource.Quantity` for decimals.

##### groupversion_info.go

First, we have some *package-level* markers that denote that there are Kubernetes objects in this package, and that this package represents the group `batch.tutorial.kubebuilder.io`.

Then, we have the commonly useful variables that help us set up our Scheme. 

##### zz_generated.deepcopy.go

`zz_generated.deepcopy.go` contains the autogenerated implementation of the aforementioned `runtime.Object` interface, which marks all of our root types as representing Kinds.



#### What’s in a controller?

It’s a controller’s job to ensure that, for any given object, the actual state of the world (both the cluster state, and potentially external state like running containers for Kubelet or loadbalancers for a cloud provider) matches the desired state in the object.



First, we start out with some standard imports. As before, we need the **core controller-runtime library**, as well as **the client package**, and **the package for our API types.**

```
package controllers

import (
    "context"

    "github.com/go-logr/logr"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```

Next, kubebuilder has scaffolded a basic reconciler struct for us. Pretty much every reconciler needs to log, and needs to be able to fetch objects, so these are added out of the box.



Most controllers eventually end up running on the cluster, so they need RBAC permissions, which we specify using controller-tools [RBAC markers](https://book.kubebuilder.io/reference/markers/rbac.html). These are the bare minimum permissions needed to run. As we add more functionality, we’ll need to revisit these.



Finally, we add this reconciler to the manager, so that it gets started when the manager is started.

```
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Complete(r)
}
```

#### Implementing a controller

The basic logic of our CronJob controller is this:

1. Load the named CronJob
2. List all active jobs, and update the status
3. Clean up old jobs according to the history limits
4. Check if we’re suspended (and don’t do anything else if we are)
5. Get the next scheduled run
6. Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy
7. Requeue when we either see a running job (done automatically) or it’s time for the next scheduled run.



We’ll start out with some imports. You’ll see below that we’ll need a few more imports than those scaffolded for us. We’ll talk about each one when we use it.

```
package controllers

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/go-logr/logr"
    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batch "tutorial.kubebuilder.io/project/api/v1"
)
```

Next, we’ll need a Clock, which will allow us to fake timing in our tests.

```
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
    Clock
}
```

We’ll mock out the clock to make it easier to jump around in time while testing, the “real” clock just calls `time.Now`.

```
type realClock struct{}

func (_ realClock) Now() time.Time { return time.Now() }

// clock knows how to get the current time.
// It can be used to fake out timing for testing.
type Clock interface {
    Now() time.Time
}
```

Notice that we need a few more RBAC permissions -- since we’re creating and managing jobs now, we’ll need permissions for those, which means adding a couple more [markers](https://book.kubebuilder.io/reference/markers/rbac.html).

```
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get
```

Now, we get to the heart of the controller -- the reconciler logic.

```

```

Question:

1. 基本类型为什么用指针

2. k8s api 的校验在哪里实现的

3. init container 

   ```
   https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
   ```

4. what is schema
5. Who involks reconcile method
6. controller for watch usage
