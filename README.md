# Docker-in-Docker

## ABOUT

This helm chart deploys the Docker-in-Docker daemon as a Kubernetes service.

Use this if you want a Docker daemon in a Kubernetes cluster - for example, if you have developers who want
to use familiar Docker tooling to do development, without needing a Docker install on their client machines.


## USAGE


### Install the helm chart

Add the helm repo, update it, and install from it, generating a new random name for the new install.

You can repleace `--generate-name` with a shorter static name (like "dind") if this is the only install in your namespace.

```
$ helm repo add dind https://peterwwillis.github.io/helm-docker-in-docker/
"dind" has been added to your repositories
$ helm search repo dind
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
dind/docker-in-docker	0.0.2        	24.0.2-dind	A Helm chart to deploy Docker-in-Docker (dind)
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "dind" chart repository
Update Complete. ⎈Happy Helming!⎈
$ helm install dind/docker-in-docker --generate-name
NAME: docker-in-docker-1687405617
LAST DEPLOYED: Wed Jun 21 23:47:01 2023
NAMESPACE: my-namespace
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Docker-in-Docker is deployed!

To connect to the Docker daemon:
  1. Set the following environment variables for your Docker client:
       DOCKER_HOST=tcp://docker-in-docker-1687405617:2376
       DOCKER_TLS_CERTDIR=/certs
       DOCKER_CERT_PATH=/certs/client
       DOCKER_TLS_VERIFY=1
  2. Download the kubernetes secret 'docker-in-docker-1687405617-cert-client' and create the
     following files for the Docker client:
       'ca.crt' -> /certs/client/ca.pem
       'tls.crt' -> /certs/client/cert.pem
       'tls.key' -> /certs/client/key.pem
  3. Use your Docker client as normal. It should automatically authenticate via TLS.
```


### Write down the generated service name

Check the `NAME: ` line from the Helm install and fill in below at `DIND_NAME=`.
This will make subsequent steps a little easier.

```
$ DIND_NAME=docker-in-docker-1687405617
$ 
```


### Configure your Docker client

This assumes you are on your local machine.

1. First download the TLS certificates from the K8s secret:

```
$ mkdir -p ~/.docker/contexts/docker-in-docker
$ kubectl get secret "${DIND_NAME}-cert-client" --template='{{index .data "ca.crt" | base64decode}}' > ~/.docker/contexts/docker-in-docker/ca.pem
$ kubectl get secret "${DIND_NAME}-cert-client" --template='{{index .data "tls.crt" | base64decode}}' > ~/.docker/contexts/docker-in-docker/cert.pem
$ kubectl get secret "${DIND_NAME}-cert-client" --template='{{index .data "tls.key" | base64decode}}' > ~/.docker/contexts/docker-in-docker/key.pem
$ chmod 0600 ~/.docker/contexts/docker-in-docker/*.pem
```

2. If you are on your local machine, open a port-forward to the Docker service:

```
$ kubectl port-forward "service/${DIND_NAME}" 12345:docker &
[1] 9481
$ Forwarding from 127.0.0.1:12345 -> 2376
Forwarding from [::1]:12345 -> 2376
```

3. Create and configure a Docker context.
   (If you are on a Pod in the K8s cluster, you can use `DIND_HOST=${DIND_NAME}` and `DIND_PORT=2376`)

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


### Uninstall the Helm chart

```
$ helm uninstall "${DIND_NAME}"
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

