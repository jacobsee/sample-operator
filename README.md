# Sample-Operator

This is an extremely simple operator to use as a learning guide. It accepts `Webserver` custom resources, and deploys an Apache pod with a `service` and a `route` on OpenShift to access the server.

This operator was generated with

```bash
operator-sdk init --domain redhat.com --repo github.com/jacobsee/sample-operator --skip-go-version-check
```

The controller & API were generated with

```bash
operator-sdk create api --group servers --version v1alpha1 --kind Webserver --resource --controller
```
