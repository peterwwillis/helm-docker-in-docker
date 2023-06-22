# Docker-in-Docker

## ABOUT

This helm chart deploys the Docker-in-Docker daemon as a Kubernetes service.

Use this if you want a Docker daemon in a Kubernetes cluster - for example, if you have developers who want
to use familiar Docker tooling to do development, without needing a Docker install on their client machines.


## USAGE


### Install the helm chart

From the root of this Git repository:
```
$ helm install dind ./charts/docker-in-docker/
```


### Configure your Docker client

This assumes you are on your local machine.

1. First download the TLS certificates from the K8s secret:

```
$ mkdir -p ~/.docker/contexts/docker-in-docker
$ kubectl get secret dind-cert-client \
    --template='{{index .data "ca.crt" | base64decode}}' \
    > ~/.docker/contexts/docker-in-docker/ca.pem
$ kubectl get secret dind-cert-client \
    --template='{{index .data "tls.crt" | base64decode}}' \
    > ~/.docker/contexts/docker-in-docker/cert.pem
$ kubectl get secret dind-cert-client \
    --template='{{index .data "tls.key" | base64decode}}' \
    > ~/.docker/contexts/docker-in-docker/key.pem
$ chmod 0600 ~/.docker/contexts/docker-in-docker/*.pem
```

2. If you are on your local machine, open a port-forward to the Docker service:

```
$ kubectl port-forward service/dind 12345:docker &
[1] 9481
$ Forwarding from 127.0.0.1:12345 -> 2376
Forwarding from [::1]:12345 -> 2376
```

3. Create and configure a Docker context. If you are on a Pod in the K8s cluster, you can use `DIND_HOST=dind` and `DIND_PORT=2376`:

```
$ DIND_HOST=localhost
$ DIND_PORT=12345
$ docker context create \
    --docker "host=tcp://$DIND_HOST:$DIND_PORT,ca=$HOME/.docker/contexts/docker-in-docker/ca.pem,cert=$HOME/.docker/contexts/docker-in-docker/cert.pem,key=$HOME/.docker/contexts/docker-in-docker/key.pem" \
    --description="docker-in-docker" \
    docker-in-docker
docker-in-docker
Successfully created context "docker-in-docker"
$ 
$  docker context use docker-in-docker
docker-in-docker
Current context is now "docker-in-docker"
```

4. Test that the connection works:

```
$ docker ps
Handling connection for 12345
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
$ 
```


## Uninstall the Helm chart

```
$ helm uninstall dind
```


# NOTES

 - This is not designed for production use; buyer beware.
 - You probably want to set resource limits before deploying.
 - I have not tested the Ingress. It's likely to interfere with the default TLS configuration, so you may need to disable the `secureDocker` value.
 - I have not tested the HPA.


# CREDITS

Thanks to the Drone.io crew for the [basis of this chart](https://github.com/drone/charts/tree/master/charts/drone-runner-docker).


# ERRATA

 - With the current secure mode, the Drone GC doesn't seem to do TLS handshakes correctly.
   I'm not sure why, but I suspect it's because Helm doesn't have a way to support the
   'extendedKeyUsage' x509 extension (https://docs.docker.com/engine/security/protect-access/#create-a-ca-server-and-client-keys-with-openssl).
   You can try to re-generate the keys yourself and put them into the secrets and see if
   that fixes it.


# LINKS
 - https://docs.docker.com/engine/security/protect-access/
 - https://docs.docker.com/engine/extend/plugins_authorization/
 - https://docs.docker.com/engine/security/protect-access/#create-a-ca-server-and-client-keys-with-openssl

