# Sample-Operator

This is an extremely simple operator to use as a learning guide. It accepts `Webserver` custom resources, and deploys an Apache pod with a `service` and a `route` on OpenShift to access the server.

## Setup

We should be starting out with an empty project folder.

From that empty folder, scaffold out an operator with

```bash
operator-sdk init --domain redhat.com --repo github.com/jacobsee/sample-operator --skip-go-version-check
```

and take a look around - there are some standard Go project files, and some Kubernetes manifests in the `config` directory.

To do any real work though, we need to create an API resource with

```bash
operator-sdk create api --group servers --version v1alpha1 --kind Webserver --resource --controller
```

And now there are some really interesting things to look through!
Since we've created a `Webserver` resource type, there are two new notable files to look at - the API definition in `api/v1alpha1/webserver_types.go`, and the controller logic in `controllers/webserver_controller.go`.

The API definition is where we will declare the structure of our resources' `spec` and `status` blocks. This is important because Go needs to know the types to expect elsewhere in our code, and also because the `operator-sdk` will scaffold out OpenAPI specs for OpenShift to enforce at runtime.

The controller is the _implementation_ of our custom logic. More on that later - but if you're familiar with a language like C++ or similar, you can think of the `*_types.go` files as header files and the `*_controller.go` files as the implementation.

Let's make `replicaCount` a field of our new custom resource. We'll do that by adding it to the `WebserverSpec` struct.

```go
type WebserverSpec struct {
    ReplicaCount int `json:"replicaCount,omitempty"`
}
```

Note that it is convention in Go to use PascalCase (or, CamelCase with the first letter capitalized) for resources that need to be accessible across packages. Since our `types` file is in the package `v1alpha1` and our controller is in the package `controllers` - we will need to do that here and name it `ReplicaCount`. However, since it is convention in YAML to use camelCase with a lowercase first letter, we will add a JSON annotation with the parameter name `replicaCount` so that we can use it like that there. We've also added `omitempty` so that this field is not required.

We will now re-run the `make` command to have the `operator-sdk` do the appropriate scaffolding for our new property. Our `ReplicaCount` property is now available for use in our controller!

## Deployment Reconciliation

It's time now to make use of the controller at `controllers/webserver_controller.go`.

As mentioned before, this is where we'll write the implementation logic for our operator. Specifically, we're paying the most attention to the `Reconcile` function. The reconcile function is automatically called when an instance of a resource type that it is watching is created, modified, or deleted.

Add some setup logic to the `Reconcile` function. It's only passed the _name_ of a modified resource, so we need to do a `Get` request with the client in order to fetch all the information we need:

```go
logger := log.FromContext(ctx)
logger.Info("Reconciling Webserver")

instance := &serversv1alpha1.Webserver{}
err := r.Client.Get(ctx, req.NamespacedName, instance)
if err != nil {
    if errors.IsNotFound(err) {
        return ctrl.Result{}, nil
    }
    return ctrl.Result{}, err
}
```

Next, let's create an object representing the `Deployment` that we wish to create:

```go
labels := map[string]string{"app": instance.Name}
deployment := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name:      instance.Name,
        Namespace: instance.Namespace,
    },
    Spec: appsv1.DeploymentSpec{
        Replicas: &instance.Spec.Count,
        Selector: &metav1.LabelSelector{
            MatchLabels: labels,
        },
        Template: corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: labels,
            },
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{
                    {
                        Name:  "webserver",
                        Image: "registry.access.redhat.com/rhscl/httpd-24-rhel7:latest",
                        Ports: []corev1.ContainerPort{
                            {
                                Name:          "http",
                                ContainerPort: 8080,
                            },
                        },
                    },
                },
            },
        },
    },
}
```

This should look somewhat familiar for someone who has written a deployment manifest in YAML before. It is essentially the same thing with added type information required in Go. At this point, we have our object defined but it has not yet been applied to the cluster. Just a little more work to make that happen:

```go
err = r.Client.Create(context.TODO(), deployment)
if err != nil {
    err = r.Client.Update(context.TODO(), deployment)
    if err != nil {
        return ctrl.Result{}, err
    }
}
```

This is the _minimal_ amount of code required to make our operator deploy an object to OpenShift/k8s when it receives a custom resource. However, there is probably one more thing we want to take care of at the same time, which is cleanup after the custom resource is deleted. Kubernetes provides a way to link resources together, so that when one is deleted, the other is cleaned up as well. In this case, we'll want the deletion of our `Webserver` resource to clean up the deployment as well, so we'll need to add the following **controller reference** just before creating it:

```go
if err := controllerutil.SetControllerReference(instance, deployment, r.Scheme); err != nil {
    return ctrl.Result{}, err
}
```

The reconciler function should look something like this:

```go
func (r *WebserverReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Reconciling Webserver")

    instance := &serversv1alpha1.Webserver{}
    err := r.Client.Get(ctx, req.NamespacedName, instance)
    if err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    labels := map[string]string{"app": instance.Name}
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      instance.Name,
            Namespace: instance.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &instance.Spec.Count,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "webserver",
                            Image: "registry.access.redhat.com/rhscl/httpd-24-rhel7:latest",
                            Ports: []corev1.ContainerPort{
                                {
                                    Name:          "http",
                                    ContainerPort: 8080,
                                },
                            },
                        },
                    },
                },
            },
        },
    }

    if err := controllerutil.SetControllerReference(instance, deployment, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }

    err = r.Client.Create(context.TODO(), deployment)
    if err != nil {
        err = r.Client.Update(context.TODO(), deployment)
        if err != nil {
            return ctrl.Result{}, err
        }
    }

    return ctrl.Result{}, nil
}
```
