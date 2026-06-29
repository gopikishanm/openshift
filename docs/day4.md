## Day 4

Today, the target is to deploy nginx and understand routes, services

```sh
~$ oc new-project day4-nginx
Now using project "day4-nginx" on server "https://api.crc.testing:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.43 -- /agnhost serve-hostname

~$ oc new-app nginx-example
--> Deploying template "day4-nginx/nginx-example" to project day4-nginx

     Nginx HTTP server and a reverse proxy
     ---------
     An example Nginx HTTP server and a reverse proxy (nginx) application that serves static content. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/nginx-ex/blob/master/README.md.

     The following service(s) have been created in your project: nginx-example.

     For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/nginx-ex/blob/master/README.md.

     * With parameters:
        * Name=nginx-example
        * Namespace=openshift
        * NGINX Version=1.20-ubi8
        * Memory Limit=512Mi
        * Git Repository URL=https://github.com/sclorg/nginx-ex.git
        * Git Reference=
        * Context Directory=
        * Application Hostname=
        * GitHub Webhook Secret=1YxiAvACYYigdIRPyl34hVNi8fsRKwMO1QDoXulM # generated
        * Generic Webhook Secret=Mg5PLmch8BH07X4rKMYywBAcHOqLEYGFs6eaHaiF # generated

--> Creating resources ...
    service "nginx-example" created
    route.route.openshift.io "nginx-example" created
    imagestream.image.openshift.io "nginx-example" created
    buildconfig.build.openshift.io "nginx-example" created
    deployment.apps "nginx-example" created
--> Success
    Access your application via route 'nginx-example-day4-nginx.apps-crc.testing'
    WARNING: No container image registry has been configured with the server. Automatic builds and deployments may not function.
    Build scheduled, use 'oc logs -f buildconfig/nginx-example' to track its progress.
    Run 'oc status' to view your app.

~$ oc logs -f buildconfig/nginx-example
error: no builds found for "nginx-example"

~$ oc status
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
In project day4-nginx on server https://api.crc.testing:6443

http://nginx-example-day4-nginx.apps-crc.testing (svc/nginx-example)
  deployment/nginx-example deploys istag/nginx-example:latest <-
    bc/nginx-example source builds https://github.com/sclorg/nginx-ex.git on openshift/nginx:1.20-ubi8
      not built yet
    deployment #1 running for 31 seconds - 0/1 pods growing to 1

Errors:
  * bc/nginx-example is pushing to istag/nginx-example:latest, but the administrator has not configured the integrated container image registry.

1 error, 1 warning identified, use 'oc status --suggest' to see details.

~$ oc status --suggest
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
In project day4-nginx on server https://api.crc.testing:6443

http://nginx-example-day4-nginx.apps-crc.testing (svc/nginx-example)
  deployment/nginx-example deploys istag/nginx-example:latest <-
    bc/nginx-example source builds https://github.com/sclorg/nginx-ex.git on openshift/nginx:1.20-ubi8
      not built yet
    deployment #1 running for about a minute - 0/1 pods growing to 1

Errors:
  * bc/nginx-example is pushing to istag/nginx-example:latest, but the administrator has not configured the integrated container image registry.
    try: oc registry -h

Warnings:
  * istag/nginx-example:latest needs to be imported or created by a build.
    try: oc start-build nginx-example

View details with 'oc describe <resource>/<name>' or list resources with 'oc get all'.

~$ oc get all
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/nginx-example   ClusterIP   10.217.4.190   <none>        8080/TCP   2m22s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-example   0/1     0            0           2m22s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-example-69984c487c   1         0         0       2m22s

NAME                                           TYPE     FROM   LATEST
buildconfig.build.openshift.io/nginx-example   Source   Git    0

NAME                                           IMAGE REPOSITORY   TAGS   UPDATED
imagestream.image.openshift.io/nginx-example

NAME                                     HOST/PORT                                   PATH   SERVICES        PORT    TERMINATION   WILDCARD
route.route.openshift.io/nginx-example   nginx-example-day4-nginx.apps-crc.testing          nginx-example   <all>                 None

~$ oc set image deploy/nginx-example nginx-example=bitnami/nginx:sha256-cdf9e347e44ecd304efb7ce5e1802f3f0ebf1d8eb52efdf1315576f95ce13915
deployment.apps/nginx-example image updated

~$ oc get pods
NAME                             READY   STATUS                 RESTARTS   AGE
nginx-example-57f448b887-qvjfd   0/1     CreateContainerError   0          70s

~$ oc describe pod nginx-example-57f448b887-qvjfd
Name:             nginx-example-57f448b887-qvjfd
Namespace:        day4-nginx

Events:
  Type     Reason          Age               From               Message
  ----     ------          ----              ----               -------
  Normal   Scheduled       81s               default-scheduler  Successfully assigned day4-nginx/nginx-example-57f448b887-qvjfd to crc
  Normal   AddedInterface  80s               multus             Add eth0 [10.217.0.20/23] from ovn-kubernetes
  Normal   Pulling         80s               kubelet            Pulling image "bitnami/nginx:sha256-cdf9e347e44ecd304efb7ce5e1802f3f0ebf1d8eb52efdf1315576f95ce13915"
  Normal   Pulled          70s               kubelet            Successfully pulled image "bitnami/nginx:sha256-cdf9e347e44ecd304efb7ce5e1802f3f0ebf1d8eb52efdf1315576f95ce13915" in 10.218s (10.218s including waiting). Image size: 7948 bytes.
  Warning  Failed          70s               kubelet            Error: reference "[overlay@/var/lib/containers/storage+/run/containers/storage:overlay.skip_mount_home=true]docker.io/bitnami/nginx@sha256:4f1850ca2bb183d4a24c2be40fc983c1e4e3bf83543bbfa8bc42e618340db5a8" does not resolve to an image ID
  Normal   Pulled          5s (x6 over 70s)  kubelet            Container image "bitnami/nginx:sha256-cdf9e347e44ecd304efb7ce5e1802f3f0ebf1d8eb52efdf1315576f95ce13915" already present on machine
  Warning  Failed          5s (x6 over 70s)  kubelet            Error: reading image "010a981ff7bfa8416bac963b62ec8eaf862c54d6cac54a1d6b2141f2e94b0717": locating image with ID "010a981ff7bfa8416bac963b62ec8eaf862c54d6cac54a1d6b2141f2e94b0717": image not known

# Add adm policy
~$ oc adm policy add-scc-to-user anyuid -z default -n day4-nginx
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "default"

# Added below yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: bitnami-nginx-app
  namespace: day4-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bitnami-nginx
  template:
    metadata:
      labels:
        app: bitnami-nginx
    spec:
      containers:
      - name: nginx
        image: bitnami/nginx:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: day4-nginx
spec:
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: bitnami-nginx

~/day4$ oc get pods
NAME                                 READY   STATUS    RESTARTS   AGE
bitnami-nginx-app-6cbd4d7f64-6mzdh   1/1     Running   0          25s

~/day4$ oc logs bitnami-nginx-app-6cbd4d7f64-6mzdh --tail 10
subject=CN=example.com
nginx 08:53:19.85 INFO  ==> No custom scripts in /docker-entrypoint-initdb.d
nginx 08:53:19.85 INFO  ==> Initializing NGINX
realpath: /bitnami/nginx/conf/vhosts: No such file or directory
nginx 08:53:19.88 INFO  ==> ** NGINX setup finished! **

nginx 08:53:19.90 INFO  ==> ** Starting NGINX **

~/day4$ oc get all
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
NAME                                     READY   STATUS    RESTARTS   AGE
pod/bitnami-nginx-app-6cbd4d7f64-6mzdh   1/1     Running   0          64s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/nginx-example   ClusterIP   10.217.4.190   <none>        8080/TCP   25m
service/nginx-service   ClusterIP   10.217.5.196   <none>        8080/TCP   64s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/bitnami-nginx-app   1/1     1            1           64s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/bitnami-nginx-app-6cbd4d7f64   1         1         1       64s

NAME                                           TYPE     FROM   LATEST
buildconfig.build.openshift.io/nginx-example   Source   Git    0

NAME                                           IMAGE REPOSITORY   TAGS   UPDATED
imagestream.image.openshift.io/nginx-example

NAME                                     HOST/PORT                                   PATH   SERVICES        PORT    TERMINATION   WILDCARD
route.route.openshift.io/nginx-example   nginx-example-day4-nginx.apps-crc.testing          nginx-example   <all>                 None

# The url is not reachable from host after updating /etc/hosts file. 
# Application is not available error

# Deployed a route to point to nginx service
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-route
  namespace: day4-nginx
spec:
  host: day4-nginx.apps-crc.testing
  to:
    kind: Service
    name: nginx-service
  port:
    targetPort: 8080

# Tried curl from ubuntu vm

~/day4$ curl -k http://day4-nginx.apps-crc.testing
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

From laptop

![day 4 output | 300](./images/day4_output.png)

### Questions

- skip_mount_home: true This is specific to underlying applications that are running containers like Podman or crio