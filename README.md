# Nginx Ingress Controller With Security Protections

Sample script for deployment [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/) in Kubernetes cluster with security protections

## Disclaimer

> :warning: This is demo purpose scripts, review changes before apply, **DO NOT** apply to production directly :warning:

## Prerequisites

- Kubernetes 1.21+
- Kubernetes CLI 1.21+
- Kubernetes Helm 3.9+

## Why I Create this Repository?

Tons of sample scripts for nginx-ingress-controller, but few of them were security by default.

## Let's Get Started

### Get Helm prepared, and don't forget to check your helm version

    $ helm version --short
    v3.9.1+ga7c043a

### Ensure helm-repo is up to date

    $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    "ingress-nginx" has been added to your repositories

    $ helm repo update ingress-nginx
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "ingress-nginx" chart repository
    Update Complete. ⎈Happy Helming!⎈

### Install Nginx Ingress Controller

    $ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace kube-system
    Release "ingress-nginx" does not exist. Installing it now.
    NAME: ingress-nginx
    LAST DEPLOYED: Sun Jul 17 22:17:38 2022
    NAMESPACE: kube-system
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None

### Verify Installation

    $ helm list --filter ingress-nginx --namespace kube-system
    NAME         	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART              	APP VERSION
    ingress-nginx	kube-system	1       	2022-07-17 22:17:38.136514 +0800 CST	deployed	ingress-nginx-4.2.0	1.3.0

    $ kubectl -n kube-system get services ingress-nginx-controller
    NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP               PORT(S)                      AGE
    ingress-nginx-controller   LoadBalancer   10.100.160.249  XXXXX.elb.amazonaws.com   80:30951/TCP,443:30575/TCP   26s

### Detect Installed Version

    $ POD_NAME=$(kubectl get pods -n kube-system -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

    $ echo ${POD_NAME}
    ingress-nginx-controller-55dcf56b68-72m8v

    $ kubectl -n kube-system exec -it ${POD_NAME} -- /nginx-ingress-controller --version
    -------------------------------------------------------------------------------
    NGINX Ingress controller
    Release:       v1.3.0
    Build:         2b7b74854d90ad9b4b96a5011b9e8b67d20bfb8f
    Repository:    https://github.com/kubernetes/ingress-nginx
    nginx version: nginx/1.19.10

    -------------------------------------------------------------------------------

### Deploy

Deploy sample scripts via `kubectl apply`

    $ kubectl apply -f ./deployment.yaml -f ./service.yaml -f ./ingress.yaml
    deployment.apps/demo-basic-auth created
    deployment.apps/demo-backend-1 created
    deployment.apps/demo-backend-2 created
    service/demo-basic-auth created
    service/demo-backend-1 created
    service/demo-backend-2 created
    ingress.networking.k8s.io/demo-ingress created

Check deployment status

    $ kubectl get ingress,service,deployment
    NAME                                     CLASS   HOSTS   ADDRESS                   PORTS   AGE
    ingress.networking.k8s.io/demo-ingress   nginx   *       XXXXX.elb.amazonaws.com   80      17s

    NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    service/demo-backend-1    ClusterIP   10.101.79.176    <none>        8088/TCP   17s
    service/demo-backend-2    ClusterIP   10.103.188.232   <none>        8088/TCP   17s
    service/demo-basic-auth   ClusterIP   10.96.102.32     <none>        80/TCP     17s
    service/kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP    34h

    NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/demo-backend-1    2/2     2            2           17s
    deployment.apps/demo-backend-2    2/2     2            2           17s
    deployment.apps/demo-basic-auth   2/2     2            2           17s

### Verification

Expose backend service entries directly with port-forward

    $ kubectl port-forward service/demo-backend-1 18088:8088
    Forwarding from 127.0.0.1:18088 -> 5678
    Forwarding from [::1]:18088 -> 5678

    $ kubectl port-forward service/demo-backend-2 28088:8088
    Forwarding from 127.0.0.1:28088 -> 5678
    Forwarding from [::1]:28088 -> 5678

Check backend service returns via proxy

    $ curl -i -u 'user:mysecretpassword' "http://localhost:18088/v1"
    HTTP/1.1 200 OK
    X-App-Name: http-echo # <--------------------- Service information exposed.
    X-App-Version: 0.2.3 # <--------------------- Running version information exposed.
    Date: Sun, 17 Jul 2022 14:25:47 GMT
    Content-Length: 34
    Content-Type: text/plain; charset=utf-8

    "this page is served by backend1"

    $ curl -i -u 'user:mysecretpassword' "http://localhost:28088/v2"
    HTTP/1.1 200 OK
    X-App-Name: http-echo # <--------------------- Service information exposed.
    X-App-Version: 0.2.3 # <--------------------- Running version information exposed.
    Date: Sun, 17 Jul 2022 14:25:48 GMT
    Content-Length: 34
    Content-Type: text/plain; charset=utf-8

    "this page is served by backend2"

Wait until ingress endpoint become ready (ADDRESS fieled should show ELB address)

    $ kubectl get ingress
    NAME           CLASS   HOSTS   ADDRESS                   PORTS   AGE
    demo-ingress   nginx   *       XXXXX.elb.amazonaws.com   80      42s

Let's check the response again with ELB endpoint

    $ curl -i -u 'user:mysecretpassword' "http://${LOAD_BALANCER}/v1"
    HTTP/1.1 200 OK
    Date: Sun, 17 Jul 2022 14:27:27 GMT
    Content-Type: text/plain; charset=utf-8
    Content-Length: 34
    Connection: keep-alive # <--------------------- No sensitive information expose.

    "this page is served by backend1"

    $ curl -i -u 'user:mysecretpassword' "http://${LOAD_BALANCER}/v2"
    HTTP/1.1 200 OK
    Date: Sun, 17 Jul 2022 14:27:48 GMT
    Content-Type: text/plain; charset=utf-8
    Content-Length: 34
    Connection: keep-alive # <--------------------- No sensitive information expose.

    "this page is served by backend2"

Try to modify `ingress.yaml`, and see what's the difference

In this example, response header for the http requests:

- `/v1` and `/v2` are routed to different backend services
- Nginx version is not exposed
- Server information is hidden
- Protected by [ModSecurity](https://modsecurity.org/)
- Protected by Basic DoS Protection

## Cleanup

Cleanup sample scripts via `kubectl delete`

    $ kubectl delete -f ./deployment.yaml -f ./service.yaml -f ./ingress.yaml

Cleanup Nginx Ingress Controller

    $ helm uninstall nginx-ingress --namespace kube-system

# Reference

- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Nginx Full Configurations Example](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [ModSecurity Web Application Firewall](https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/)
- [Role-Based Access Control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

# License

[GPLv2](LICENSE)
