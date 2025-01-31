## Running Moov ACH Server

The [Moov ACH Project](https://github.com/moov-io/ach) offers an HTTP Server for REST based ACH file construction, validation and tabulation. Any language which can use HTTP and JSON can use the ACH Server. We offer examples of using the HTTP interface in [Go](https://github.com/moov-io/ach/blob/master/examples/http/main.go) and [Ruby](https://github.com/moov-io/ruby-ach-demo/blob/master/main.rb).

### Getting Started

To start using Moov's ACH service you'll first need to obtain a binary of the project. These can be downloaded [from the releases page](https://github.com/moov-io/ach/releases) or from [Docker Hub](https://hub.docker.com/r/moov/ach).

#### Running ACH Server from a binary

First download the [latest Moov ACH server release](https://github.com/moov-io/ach/releases) and run it! You'll see some log output but the service will be running on port `8080` (with an admin port on `9090`).

```
# Download latest ACH binary from https://github.com/moov-io/ach/releases
# Example: https://github.com/moov-io/ach/releases/download/v1.0.0/ach-darwin-amd64

# Run Moov's ACH server
$ ach-linux-amd64
ts=2019-02-21T17:03:23.782592Z caller=main.go:69 startup="Starting ach server version v1.0.0"
ts=2019-02-21T17:03:23.78314Z caller=main.go:129 transport=HTTP addr=:8080
ts=2019-02-21T17:03:23.783252Z caller=main.go:119 admin="listening on :9090"
```

Once the ACH server is started you can ping or get the list of ACH files.

```
$ curl http://localhost:8080/ping
PONG

$ curl http://localhost:8080/files
{"files":[],"error":null}
```

You can also send [an example PPD ACH file we have](https://github.com/moov-io/ach/blob/master/test/testdata/ppd-valid.json) to any ACH service or read through HTTP examples in [Ruby](https://github.com/moov-io/ruby-ach-demo) and [Go](https://github.com/moov-io/ach/blob/master/examples/http/main.go).

#### Running ACH Server on a local Docker host

First, please [read over their getting started guide](https://docs.docker.com/get-started/) then to run the ACH Server locally:

```
$ docker run moov/ach:v1.0.0 # Use the latest released version
ts=2019-02-21T17:03:23.782592Z caller=main.go:69 startup="Starting ach server version v1.0.0"
ts=2019-02-21T17:03:23.78314Z caller=main.go:129 transport=HTTP addr=:8080
ts=2019-02-21T17:03:23.783252Z caller=main.go:119 admin="listening on :9090"
```
(On OS X, you'll need to use [port forwarding](https://docs.docker.com/docker-for-mac/networking/#known-limitations-use-cases-and-workarounds):
`$ docker run -p 8080:8080 -p 9090:9090 moov/ach:v1.0.0`)

Then, (in a new terminal/shell) you can start making HTTP requests towards the ACH Server:

```
$ curl http://localhost:8080/ping
PONG

$ curl http://localhost:8080/files
{"files":[],"error":null}
```

We have posted [Ruby](https://github.com/moov-io/ruby-ach-demo) and [Go](https://github.com/moov-io/ach/blob/master/examples/http/main.go) HTTP examples for creating an entire ACH file.

#### Running ACH Server on Kubernetes

The following snippet runs the ACH Server on [Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) in the `apps` namespace. You could reach the ach instance at the following URL from inside the cluster.

```
# Needs to be ran from inside the cluster
$ curl http://ach.apps.svc.cluster.local:8080/ping
PONG

$ curl http://localhost:8080/files
{"files":[],"error":null}
```

Kubernetes manifest - save in a file (`ach.yaml`) and apply with `kubectl apply -f ach.yaml`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apps
---
apiVersion: v1
kind: Service
metadata:
  name: ach
  namespace: apps
spec:
  type: ClusterIP
  selector:
    app: ach
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
    - name: metrics
      protocol: TCP
      port: 9090
      targetPort: 9090
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ach
  namespace: apps
  labels:
    app: ach
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ach
  template:
    metadata:
      labels:
        app: ach
    spec:
      containers:
      - image: moov/ach:v1.0.0
        imagePullPolicy: Always
        name: ach
        args:
          - -http.addr=:8080
          - -admin.addr=:9090
        env:
          - name: ACH_FILE_TTL
            value: 30m # 30 minutes
        ports:
          - containerPort: 8080
            name: http
            protocol: TCP
          - containerPort: 9090
            name: metrics
            protocol: TCP
        resources:
          limits:
            cpu: 0.1
            memory: 50Mi
          requests:
            cpu: 25m
            memory: 10Mi
        readinessProbe:
          httpGet:
            path: /ping
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /ping
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
      restartPolicy: Always
```
