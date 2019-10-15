---
authors:
- dsharp
- jvigil
- khuddleston
- gtadi
categories:
- Greenplum
- Kubernetes
- KubeBuilder
date: 2019-10-14T17:00:00Z
draft: true
short: |
  Lessons we learned while building a controller with KubeBuilder
title: How we built a controller using KubeBuilder with Test Driven development, Part 1
image: /images/pairing.jpg

---

## Who are we?

We are the Greenplum for Kubernetes team. We're working on a Kubernetes operator
to run [Greenplum](https://greenlpum.org), and connected components of Greenplum
like [PXF](https://gpdb.docs.pivotal.io/5220/pxf/overview_pxf.html) and
[GPText](https://gptext.docs.pivotal.io/330/welcome.html). We started with a
controller for Greenplum, and recently used
[KubeBuilder](https://github.com/kubernetes-sigs/kubebuilder) to add controllers
for PXF and GPText.

In this post, we review some of the key lessons we learned in the course of
developing our KubeBuilder controllers. In
[_Part 2_](/post/gp4k-kubebuilder-tdd), we cover our journey of discovery of how
to unit test our new controllers.

## Building the Controller

The [KubeBuilder](https://book.kubebuilder.io/) book covers in detail [how to
write a simple controller][kb-book-controller-impl]. Despite the thoroughness of
the book, there were some key points that tripped us up.

[kb-book-controller-impl]: https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html

### Controller Basics

Our use case is a simple _CustomResourceDefinition_ to parameterize construction
of a _Deployment_ and _Service_. After the reconciler is registered with the
controller-runtime _Manager_, then its `Reconcile()` method will be called any
time an object needs to be updated. The controller manager takes care of the
basic controller details like listing and watching for changes to the objects,
caching retrieved objects, and queuing and rate limiting reconciliations. When
it determines that a change may need to be made, it calls the registered
reconciler. The `reconcile.Request` parameter to `Reconcile()` contains only the
namespace and name of the object to be reconciled. So, the first step is to Get
the object. Then, based on its current contents, we can construct the desired
state of our dependent objects (Deployment, Service), and ensure the desired
state is set for those objects in the apiserver.

Setting that desired state correctly and succinctly took us a few iterations,
so below are some key details that we learned in the process, and finishing
with an example Reconcile implementation.

### CreateOrUpdate

_CreateOrUpdate_ is a helper method provided by controller-runtime. It does
what it says on the tin: Given an _Object_, it will either _Create_ it or
_Update_ an existing object. Because of this functionality, it forms the core of
many `Reconcile()` functions. However, its API makes it extremely easy to misuse
without understanding some of its internals.

`CreateOrUpdate()` takes a callback, "mutate", which is where all changes to the
object must be performed. This point bears repeating: your mutate callback is
the only place you should enact the contents of your object, aside from the
name and namespace which must be filled in prior. Under the hood,
_CreateOrUpdate_ calls `Get()` on the object. If the object does not exist, then
`Create()` will be called. If it does exist, then `Update()` will be called. In
either case, the mutate callback will be called first.

In the `Create()` path, the `Get()` does not modify your object, so initial
testing may appear to work if you pass a no-op mutate. The object will get
created as it was before CreateOrUpdate was called. However, an issue arises
during `Update()`. In the `Update()` path, the object already exists, so the
`Get()` call overwrites it. The `Update()` will never do anything, even if the
actual object has diverged from the desired state. Therefore proper usage of
"mutate" is essential to reconcile the object.

In your mutate callback, you should surgically modify individual fields of the
object. Don't overwrite large chunks of the object, or the whole object, as we
tried to do initially. Overwriting the object would discard the `metadata`
field, and cause _Update_ to fail, or overwrite the `status` field, losing
state. It could also interfere with other controllers trying to make their own
modifications to the object. Remember, you are only one controller in the whole
Kubernetes cluster&mdash;play nice with others.

### Garbage Collection, OwnerReferences and ContollerRefs

The _OwnerReferences_ field within the metadata field of all Kubernetes objects
declares that if the referred object (the owner) is deleted, then the object
with the reference (the dependent) should also be deleted by [garbage
collection][k8s-gc]. A controller should create a
[ControllerRef][k8s-controllerref] (an OwnerReference with `Controller: true`)
on objects it creates. There can be only one ControllerRef on an object in order
to prevent controllers from fighting over an object.

For example, an `ownerReferences` entry is added to a Deployment that was
created by a CustomResource controller, so that when the given CustomResource
is deleted, the corresponding Deployment is also deleted.

The `controller-runtime` package provides another method,
`SetControllerReference`, to add a ControllerRef to objects that a controller
creates. It takes care of correctly appending to existing OwnerReferences and
checking for an existing ControllerRef.

If we call SetControllerReference() on all objects we create during
Reconcile(), then we can rely on the garbage collector, and we do not need to
detect a deleted CR to delete its sub-resources.

[k8s-gc]: https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/
[k8s-controllerref]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/controller-ref.md

### Returning an error from Reconcile means requeue

Errors returned from Reconcile() will be logged. But returning an error from
Reconcile() also means the reconciliation will be requeued. If that's not
desirable, don't return an error. This means thinking carefully about simply
returning downstream errors.

### Ignore NotFound errors

Case in point for not returning an error: A subtlety in the KubeBuilder book
is how they ignore NotFound errors when `Get()`ting the reconciled object.
Normally NotFound would be returned when the object has been deleted. Since we
have arranged for the garbage collector to clean up our dependent objects,
there is no need to do anything with NotFound errors, so we can ignore them. If
the reconciled object was not deleted, but missing for some other reason, we
should still ignore NotFound errors. We do not want to requeue the
reconciliation by returning the error. If the object were to reappear in the
API, the controller would get another reconciliation at that time. Requeuing
the reconciliation would be a waste of time.

### Example

To put all of the above together, here is a simplified example of Reconcile():

```go
import (
	"github.com/pkg/errors"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

func (r *CustomReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := context.Background()
	// Get CustomResource
	var customResource myApi.CustomResource
	if err := r.Get(ctx, req.NamespacedName, &customResource); err != nil {
		if apierrs.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, errors.Wrap(err, "unable to fetch CustomResource")
	}

	// CreateOrUpdate SERVICE
	var svc corev1.Service
	svc.Name = customResource.Name
	svc.Namespace = customResource.Namespace
	_, err := ctrl.CreateOrUpdate(ctx, r, &svc, func() error {
		ModifyService(customResource, &svc)
		return controllerutil.SetControllerReference(&customResource, &svc, r.Scheme)
	})
	if err != nil {
		return ctrl.Result{}, errors.Wrap(err, "unable to CreateOrUpdate Service")
	}

	// CreateOrUpdate DEPLOYMENT
	var app appsv1.Deployment
	app.Name = customResource.Name + "-app"
	app.Namespace = customResource.Namespace
	_, err = ctrl.CreateOrUpdate(ctx, r, &app, func() error {
		ModifyDeployment(customResource, &app)
		return controllerutil.SetControllerReference(&customResource, &app, r.Scheme)
	})
	if err != nil {
		return ctrl.Result{}, errors.Wrap(err, "unable to CreateOrUpdate Deployment")
	}

	return ctrl.Result{}, nil
}
```

# Up Next

In our [next post](/post/gp4k-kubebuilder-tdd/), we describe our journey of
figuring out how to apply Test Driven Development to a KubeBuilder controller.
