# Insights Operator

This cluster operator gathers anonymized system configuration and reports it to Red Hat Insights. It is a part of the standard OpenShift distribution. The data collected allows for debugging in the event of cluster failures or unanticipated errors.

# Table of Contents

- [Building](#building)
- [Testing](#testing)
- [Documentation](#documentation)
- [Getting metrics from Prometheus](#getting-metrics-from-prometheus)
    - [Generate the certificate and key](#generate-the-certificate-and-key)
    - [Prometheus metrics provided by Insights Operator](#prometheus-metrics-provided-by-insights-operator)
    - [Getting the data directly from Prometheus](#getting-the-data-directly-from-prometheus)
    - [Debugging Prometheus metrics without valid CA](#debugging-prometheus-metrics-without-valid-ca)
- [Debugging](#debugging)
    - [Using the profiler](#using-the-profiler)
- [Changelog](#changelog)
    - [Updating the changelog](#updating-the-changelog)
- [Reported data](#reported-data)
- [Contributing](#contributing)
- [Support](#support)
- [License](#license)

# Building

To build the operator, install Go 1.11 or above and run:

```
$ make build
```

To test the operator against a remote cluster, run:

```sh
$ bin/insights-operator start --config=config/local.yaml --kubeconfig=$KUBECONFIG
```

where `$KUBECONFIG` has sufficiently high permissions against the target cluster.

# Testing

Unit tests can be started by the following command:

```sh
$ make test
```

It is also possible to specify CLI options for Go test. For example, if you need to disable test results caching, use the following command:

```sh
$ VERBOSE=-count=1 make test
```

> Integration (e2e) tests are not part of this repository, you can find it [here](https://gitlab.cee.redhat.com/ccx/insights-operator-tests).

# Documentation


The document [docs/gathered-data](docs/gathered-data.md) contains the list of collected data and the API that is used to collect it. This documentation is generated by the command bellow, by collecting the comment tags located above each Gather method.

To start generating the document run:

```sh
$ make docs
```

# Getting metrics from Prometheus

## Generate the certificate and key

Certificate and key are required to access Prometheus metrics (instead 404 Forbidden is returned). It is possible to generate these two files from Kubernetes config file. Certificate is stored in `users/admin/client-cerfificate-data` and key in `users/admin/client-key-data`. Please note that these values are encoded by using Base64 encoding, so it is needed to decode them, for example by `base64 -d`.

There's a tool named `gen_cert_key.py` that can be used to automatically generate both files. It is stored in `tools` subdirectory.

```sh
$ gen_cert_file.py kubeconfig.yaml
```

## Prometheus metrics provided by Insights Operator

It is possible to read Prometheus metrics provided by Insights Operator. Example of metrics exposed by Insights Operator can be found at [metrics.txt](docs/metrics.txt)

Depending on how or where the IO is running you may have different ways to retrieve the metrics. Here is a list of some options, so you can find the one that fits you:

### Running IO locally

If the IO runs locally, the following command migth be used:

```sh
$ curl --cert k8s.crt --key k8s.key -k https://localhost:8443/metrics
```

### Running IO on K8s

Get the token

```sh
$ oc whoami -t
```

Read metrics from Pod

```sh
$ oc exec \
    -it deployment/insights-operator \
    -n openshift-insights -- \
    curl -k -H "Authorization: Bearer YOUR-TOKEN-HERE" 'https://localhost:8443/metrics'
```

## Getting the data directly from Prometheus

```sh
$ sudo kubefwd svc -n openshift-monitoring -d openshift-monitoring.svc -l prometheus=k8s
$ curl --cert k8s.crt --key k8s.key  -k 'https://prometheus-k8s.openshift-monitoring.svc:9091/metrics'
```

## Debugging Prometheus metrics without valid CA

Get the token

```sh
$ oc sa get-token prometheus-k8s -n openshift-monitoring
```

Change in `pkg/controller/operator.go` after creating `metricsGatherKubeConfig` (about line #86)

```ini
metricsGatherKubeConfig.Insecure = true
metricsGatherKubeConfig.BearerToken = "YOUR-TOKEN-HERE"
# by default CAFile is /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
metricsGatherKubeConfig.CAFile = "" 
metricsGatherKubeConfig.CAData = []byte{}
```

# Debugging

## Using the profiler

### Starting IO with the profiler

IO starts a profiler if given the correct environment.
Set the `OPENSHIFT_PROFILE` env variable to "web".

```sh
$ export OPENSHIFT_PROFILE=web
```

### Collect profiling data

After IO starts the profiling can be accessed at `http://localhost:6060`, you can use the `pprof` tool to connect to it.

Some profiling examples:

```sh
# CPU profiling for 30 seconds
$ go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

```sh
# heap profiling
$ go tool pprof http://localhost:6060/debug/pprof/heap
```

These commands will create a compressed file that can be visualized using a variety of tools, one of them is the `pprof` tool.

### Analyzing profiling data

Starting a web ui at `localhost:8080` to visualize/analyze the profiling data:

```sh
$ go tool pprof -http=:8080 /path/to/profiling.out
```

For extra info: [check this link](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)

# Changelog

You can find the project changelog by [clicking here](CHANGELOG.md).

## Updating the changelog

At `./cmd/changelog/main.go` there is a script that can update the changelog for you.

It uses both the local git and GitHub`s API to update the file so:

- To get info from GitHub you will need to set the `GITHUB_TOKEN` envvar to a GitHub access-token.
- Make sure that you have a local, up-to-date copy of each release-branch that might be in the changelog.

It can be used 2 ways:

1. Providing no command line arguments the script will update the current `CHANGELOG.md` with the latest changes according to the local git state.

> 🚨 IMPORTANT: It will only work with changelogs created with this script

```sh
$ go run cmd/changelog/main.go
```

2. Providing 2 command line arguments, `AFTER` and `UNTIL` dates the script will generate a new `CHANGELOG.md` within the provided time frame.

```sh
$ go run cmd/changelog/main.go 2021-01-10 2021-01-20
```

# Reported data

* ClusterVersion
* ClusterOperator objects
* All non-secret global config (hostnames and URLs anonymized)

The list of all collected data with description, location in produced archive and link to Api and some examples is at [docs/gathered-data.md](docs/gathered-data.md)

The resulting data is packed in `.tar.gz` archive with folder structure indicated in the document. Example of such archive is at [docs/insights-archive-sample](docs/insights-archive-sample).

## Insights Operator Archive

### Sample IO archive

There is a sample IO archive maintained in this repo to use as a quick reference. (can be found at [docs/insights-archive-sample](https://github.com/openshift/insights-operator/tree/master/docs/insights-archive-sample))

To keep it up-to-date it is **required** to update this manually when developing a new data enhancement.

Make sure the `.json` files are in a humanly readable format in the sample archive.
By doing this its easier to review a data enhancement PR, and rule developers can easily check what data it collects. 

### Generating a sample archive

Run the insights-operator on a test cluster (from `cluster-bot` or `Quicklab` or etc).

### Formatting archive json files

This formats `.json` files from folder with extracted archive.

```sh
$ find . -type f -name '*.json' -print | while read line; do cat "$line" | jq > "$line.tmp" && mv "$line.tmp" "$line"; done
```

# Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for workflow & convention details.

See [STYLEGUIDE](STYLEGUIDE.md) for file format and coding style guide.

# Support

Insights Operator is part of Red Hat OpenShift Container Platform. For product-related issues, please
file a ticket [in Red Hat Bugzilla](https://bugzilla.redhat.com/enter_bug.cgi?product=OpenShift%20Container%20Platform&component=Insights%20Operator) for "Insights Operator" component.

# License

This project is licensed by the Apache License 2.0. For more information check the LICENSE file.