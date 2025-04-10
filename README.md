# Red Hat build of OpenTelemetry with a .NET App

![GitHub](https://img.shields.io/github/license/kevchu3/openshift4-grafana?color=blue&style=plastic)

## Background

Red Hat build of OpenTelemetry is based on the upstream [OpenTelemetry] CNCF project.  One of the major benefits of OpenTelemetry is its extensibility, and there are many supported collectors, processors, and exporters that can be used, resulting in many design patterns that fit different use cases.  This tutorial will explore the Kubernetes Sidecar design pattern which deploys an agent sidecar container within an application pod that sends traces to a centralized Grafana Tempo.  We will also use Auto Instrumentation to inject and configure OpenTelemetry libraries for our .NET application.

## Prerequisites

This tutorial walks through instrumentation and distributed tracing of a .NET application using Red Hat build of OpenTelemetry.  The following are assumed to have been installed already:

* OpenShift 4 (tested on 4.18)
* [Red Hat build of OpenTelemetry] operator
* [Tempo Operator] (distributed tracing platform based on Tempo)

## Distributed Tracing

Distributed tracing records the path of a request through various microservices of a containerized application.  Red Hat OpenShift distributed tracing platform is based on the upstream [Grafana Tempo project].  Refer to the docs for more reading about the [Tempo architecture].

As a cluster administrator, you will need to set up required object storage by a supported provider.  For more information, see [Object storage setup].

Install a central TempoStack instance using the [tempostack.yaml] manifest.  In this example, we've configured AWS S3 object storage with a Secret named `tempo-bucket` alongside the TempoStack instance, replace the secret with your own configuration.
```
$ oc apply -f manifests/tempostack.yaml
```

## Sidecar

The application will send tracing data to a collector agent (sidecar) that offloads responsibility by sending the data to a storage backend, in our case, the central Grafana Tempo instance.  Deploy this sidecar with the [sidecar.yaml] manifest:

```
$ oc apply -f manifests/sidecar.yaml -n dotnet-project
```

## Auto Instrumentation

Instrumentation is the process of adding observability code to an application.  Auto instrumentation creates this tracing framework without significant code changes by automatically injecting and configuring auto-instrumentation libraries in a variety of supported languages.

We'll start by deploying a sample .NET application.  From the OpenShift web console, navigate to the Developer Perspective, create a new project for the application (we'll use `dotnet-project` for this example).  Navigate to the Developer Catalog using `+Add`, select the Basic .NET catalog item, and press `Create`, which creates a deployment named `devfile-sample-dotnet-60-basic-git`.

Now, we'll create an `Instrumentation` custom resource defined in [instrumentation.yaml] in the project that will be used by the OpenTelemetry operator.  Auto instrumentation of .NET sends data on port 4318 via the OTLP/HTTP protocol.
```
$ oc apply -f manifests/instrumentation.yaml -n dotnet-project
```

We'll configure our application deployment with annotations for [.NET auto instrumentation] to enable injection and sidecar (see the following section):

```
$ oc edit deployment devfile-sample-dotnet-60-basic-git -n dotnet-project

spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-dotnet: "true"
        sidecar.opentelemetry.io/inject: sidecar
```

The following commands can be run instead to perform the edits above:
```
$ oc patch deployment devfile-sample-dotnet-60-basic-git -n dotnet-project --patch '{"spec": {"template": {"metadata": {"annotations": {"instrumentation.opentelemetry.io/inject-dotnet": "true"}}}}}'
$ oc patch deployment devfile-sample-dotnet-60-basic-git -n dotnet-project --patch '{"spec": {"template": {"metadata": {"annotations": {"sidecar.opentelemetry.io/inject": "sidecar"}}}}}'
```

## Jaeger UI

The `TempoStack` resource we deployed provides a query frontend based on Jaeger UI, and we can navigate to it as follows:
```
$ oc get route tempo-sample-query-frontend -n observability -o 'jsonpath={.spec.host}'
```

If you are logging in with OpenShift RBAC and receive a 403 Permission Denied, add the following permissions:
```
$ oc adm policy add-role-to-user view <account name> -n observability
```

Now it is time to run our test traces.  Generate spans (a unit of work in Tempo that has an operation name, start time, duration, and tags) and traces (data/execution path through the system consisting of one or more spans) to Grafana Tempo by navigating to the .NET application's route and clicking around.  The spans and traces should now show up in Jaeger.

If you are having difficulty generating data (perhaps some instructions above were misconfigured), you can create the following test job ([testjob.yaml]):
```
$ oc apply -f manifests/testjob.yaml
```


[OpenTelemetry]: https://opentelemetry.io/
[Red Hat build of OpenTelemetry]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/red_hat_build_of_opentelemetry/index
[Tempo Operator]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/distributed_tracing/distributed-tracing-platform-tempo
[Grafana Tempo project]: https://grafana.com/oss/tempo/
[Tempo architecture]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/distributed_tracing/distributed-tracing-architecture#distr-tracing-architecture_distributed-tracing-architecture
[Object storage setup]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/distributed_tracing/distributed-tracing-platform-tempo#distr-tracing-tempo-install-tempomonolithic-cli_dist-tracing-tempo-installing
[tempostack.yaml]: manifests/tempostack.yaml
[installing TempoStack]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/distributed_tracing/distributed-tracing-platform-tempo#installing-a-tempostack-instance
[instrumentation.yaml]: manifests/instrumentation.yaml
[.NET auto instrumentation]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/red_hat_build_of_opentelemetry/index#otel-configuration-of-apache-http-server-auto-instrumentation_otel-configuration-of-instrumentation
[sidecar.yaml]: manifests/sidecar.yaml
[testjob.yaml]: manifests/testjob.yaml
