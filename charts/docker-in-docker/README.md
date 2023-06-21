# Docker-in-Docker

## ABOUT

This helm chart deploys the Docker-in-Docker daemon as a Kubernetes service.

Use this if you want a Docker daemon in a Kubernetes cluster - for example, if you have developers who want
to use familiar Docker tooling to do development, without needing a Docker install on their client machines.

## USAGE


### Install the helm chart

```
helm install dind ./charts/docker-in-docker/
```


### Configure your Docker client

Assuming you are in a Pod in the Kubernetes cluster where "dind" has been deployed:

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

2. Then create and configure a Docker context to use them:

```
$ DIND_HOST=dind
$ DIND_PORT=2376
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

3. Test that the connection works:

```
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
$ 
```


## Uninstall the Helm chart

```
helm uninstall dind
```


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

