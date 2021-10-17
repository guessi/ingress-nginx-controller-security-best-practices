# Nginx Ingress Controller With Security Protections

Sample script for deployment [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/) in Kubernetes cluster with security protections

## Disclaimer

!!! This is demo purpose scripts, review changes before apply, **DO NOT** apply to production directly !!!

## Prerequisites

- Kubernetes 1.21+
- Kubernetes CLI 1.21+
- Kubernetes Helm 3.7+

## Why I Create this Repository?

Tons of sample scripts for nginx-ingress-controller, but few of them were security by default.

## Let's Get Started

### Get Helm prepared, and don't forget to check your helm version

    $ helm version --short

    v3.7.1+g1d11fcb

### Ensure helm-repo is up to date

    $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

    "ingress-nginx" has been added to your repositories

    $ helm repo update

    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "ingress-nginx" chart repository
    Update Complete. ⎈Happy Helming!⎈

### Install Nginx Ingress Controller

    $ helm -n kube-system install ingress-nginx ingress-nginx/ingress-nginx

### Verify Installation

    $ helm -n kube-system ls

    NAME         	NAMESPACE  	REVISION	UPDATED                             	STATUS  	CHART              	APP VERSION
    ingress-nginx	kube-system	1       	2021-10-17 10:57:47.876629 +0800 CST	deployed	ingress-nginx-4.0.6	1.0.4

    $ kubectl -n kube-system get services ingress-nginx-controller -o wide

    NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
    ingress-nginx-controller   LoadBalancer   10.102.122.36   localhost     80:31249/TCP,443:30223/TCP   26s   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx

### Detect Installed Version

    $ POD_NAME=$(kubectl get pods -n kube-system -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

    $ kubectl -n kube-system exec -it $POD_NAME -- /nginx-ingress-controller --version

    -------------------------------------------------------------------------------
    NGINX Ingress controller
    Release:       v1.0.4
    Build:         9b78b6c197b48116243922170875af4aa752ee59
    Repository:    https://github.com/kubernetes/ingress-nginx
    nginx version: nginx/1.19.9
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

    NAME                                     CLASS   HOSTS   ADDRESS   PORTS   AGE
    ingress.networking.k8s.io/demo-ingress   nginx   *                 80      17s

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

Wait for ingress endpoint "ADDRESS" field become ready

    $ kubectl get ingress
    NAME           CLASS   HOSTS   ADDRESS     PORTS   AGE
    demo-ingress   nginx   *       localhost   80      20m

Let's check the response for the `/v1` endpoint

    $ curl -i -u 'user:mysecretpassword' "http://localhost/v1"

    HTTP/1.1 200 OK
    Date: Sun, 17 Oct 2021 03:44:53 GMT
    Content-Type: text/plain; charset=utf-8
    Content-Length: 34
    Connection: keep-alive
    X-App-Name: http-echo
    X-App-Version: 0.2.3

    "this page is served by backend1"

Let's check the response for the `/v2` endpoint

    $ curl -i -u 'user:mysecretpassword' "http://localhost/v2"

    HTTP/1.1 200 OK
    Date: Sun, 17 Oct 2021 03:44:53 GMT
    Content-Type: text/plain; charset=utf-8
    Content-Length: 34
    Connection: keep-alive
    X-App-Name: http-echo
    X-App-Version: 0.2.3

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

    $ helm -n kube-system uninstall nginx-ingress

# Reference

- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Nginx Full Configurations Example](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [ModSecurity Web Application Firewall](https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/)
- [Role-Based Access Control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

# License

[GPLv2](LICENSE)
