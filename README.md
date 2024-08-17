# Deploy a Sample to a Cluster

## Development

Add to the hello world application.

https://fastapi.tiangolo.com/#create-it

Bonus points if you add an NGINX sidecar and serve up a frontend.  


## Build

Build a container image and tag it with `sample_app:v1`

```bash
cd sample_app
docker build -t sample_app:v1 .
```


## Run Dockerized Container

Test the containerized application

```bash
docker run -it -p 8000:8000 sample_app:v1
```

You should see the server running at port 8000

```
INFO     Importing module app
INFO     Found importable FastAPI app

 ╭─ Importable FastAPI app ─╮
 │                          │
 │  from app import app     │
 │                          │
 ╰──────────────────────────╯

INFO     Using import string app:app

 ╭─────────── FastAPI CLI - Production mode ───────────╮
 │                                                     │
 │  Serving at: http://0.0.0.0:8000                    │
 │                                                     │
 │  API docs: http://0.0.0.0:8000/docs                 │
 │                                                     │
 │  Running in production mode, for development use:   │
 │                                                     │
 │  fastapi dev                                        │
 │                                                     │
 ╰─────────────────────────────────────────────────────╯

INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Check that your application responds on `:8000`

```bash
curl http://localhost:8000
```

Then modify the application, build it with a new image and tag (not `sample_app:v1`) and run the container again.

Use the running container to execute a command.  Find the container

```bash
docker ps
```

Hop into the container

```bash
docker exec ${container_id}
```

Inside the container try something, like kill the fastapi process

```bash
kill 1
```

Is it still running?


## Deploy to Kubernetes

### Connect to the cluster

Open up Docker Desktop settings and "Enable Kubernetes"

Make sure you have the `kubectl` command if it wasn't installed with Docker Desktop use `brew install kubectl`

If you are using Mac OSX Docker Desktop will include a menu with "Kubernetes Context"

Check your local context, open up a terminal window and try

```bash
kubectl config get-contexts
```

You should see a star next to `docker-desktop`


### Deploy the application

Add `watch` with `brew install watch`

Open up a new terminal and check for running pods with

```bash
watch kubectl get pods
```

Open up the `manifests/deployments.yaml`

Does that manifest look right to you?

Once you've entered in the image and tag apply the configuration, deploy to the cluster:

```bash
kubectl apply -f manifests
```

You should see pods spin up the application

What port is the service running on?

List the services and find out the name and port it is running on

```bash
kubectl get services
```

Find out more about the deployment

```
kubectl describe sample-app-deployment
```

Find out about the pods

```
kubectl describe ${name_of_a_pod}
```


### Simulate OOMKilled

Kubernetes will automatically deploy with a round-robin blue-green deployment and attempt to keep a service running by managing resources, e.g. - setup new nodes if configured and allocate resources to pods deploying them.

In this next part we'll simulate an outage caused by OOMKilled - out of memory, an event where a process is killed by the kernel for utilizing too much memory and not releasing it.

Change the deployment resources to 1Mi and add replicas 3 - 10.

Apply your changes

```
kubectl apply -f manifests
```

Confirm that your pods are running and no restarts are happening.

Add an endpoint to `app.py` and write a method that will consume more memory than you have.  Deploy the new change.

If everything looks good continue but if you don't see the endpoint, did you remember to build a new image and to change `deployment.yaml`?  Check the manifest is correct

```bash
kubectl get deployments sample-app-deployment -o yaml
```

Hop on one of the pods and monitor the resources

```bash
kubectl exec -it ${name_of_a_pod} top
```

Trigger the issue, you might not hit the pod you are monitoring until you try a few times.

```bash
curl http://localhost:8000/name-of-problematic-endpoint
```

Shut down your cluster

```
kubectl delete -f manifests
```



## Helm Charts

Helm charts are the "package manager" for infrastructure pre-packages for you.  That is, since many of them are running in production they follow best practices for setting up all kinds of services.

Bitnami Redis chart

(https://github.com/bitnami/charts/tree/main/bitnami/redis)[https://github.com/bitnami/charts/tree/main/bitnami/redis]

Install Redis as a Helm chart

```bash
helm install redis-sample oci://registry-1.docker.io/bitnamicharts/redis
```

You should see pods for Redis come up

Follow their instructions to connect as a client using `redis-cli`

```
To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 -d)

To connect to your Redis&reg; server:

1. Run a Redis&reg; pod that you can use as a client:

   kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:7.4.0-debian-12-r1 --command -- sleep infinity

   Use the following command to attach to the pod:

   kubectl exec --tty -i redis-client \
   --namespace default -- bash

2. Connect using the Redis&reg; CLI:
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-master
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-replicas

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/redis-master 6379:6379 &
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379
```

Forward ports to connect

```
kubectl port-forward --namespace default svc/redis-master 6379:6379
```

Redis is a key-value pair database with O(1) complexity access.

Try a few examples.  This would be a naive chat room application:

1. Write a key-value

```
> set sampleapp:notification:message:room:1:message:123 "Hello world"
> set sampleapp:notification:message:room:1:message:124 "Lorem ipsum"
```

2. Get values from a set of keys

```
> keys sampleapp:notification:message:room:1:message:*
```

Uninstall the Helm chart and remove the pod you were using as a client

```
helm uninstall redis
kubectl delete pod redis-client
```


## FAQ

### I cannot curl my service even though it is running

`EXTERNAL-IP` is mapped to `<none>`

Service's `type` must be `type: LoadBalancer`

### What should I be looking for in OOMKilled

Depending on your set up it could have happened very quickly

```
kubectl describe pod sample-app-deployment-56f5467dc-rrxtc

...

State:          Running
      Started:      Fri, 16 Aug 2024 22:56:37 -0400
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Fri, 16 Aug 2024 22:55:11 -0400
      Finished:     Fri, 16 Aug 2024 22:56:36 -0400
    Ready:          True
```

And

```
kubectl get pods

...

sample-app-deployment-56f5467dc-rrxtc   1/1     Running   1 (109s ago)   3m15s
```
