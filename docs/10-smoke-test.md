# Smoke Test

This lab walks you through a quick smoke test to make sure things are working.

## Test

```
kubectl run nginx --image=nginx --port=80 --replicas=3
```

```
kubectl get pods -o wide
```
```
NAME                    READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-158599303-7k8p9   1/1       Running   0          13s       10.200.2.3   worker2
nginx-158599303-h0zcs   1/1       Running   0          13s       10.200.1.2   worker1
nginx-158599303-rfhm3   1/1       Running   0          13s       10.200.0.2   worker0
```

```
kubectl expose deployment nginx --type NodePort
```

> Note that --type=LoadBalancer will not work because we did not set up the haproxy properly. Watch this space for updates.

Grab the `NodePort` that was setup for the nginx service:

```
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```



Test the nginx service using cURL:

```
curl http://icc-node-1.ful:${NODE_PORT}
curl http://icc-node-2.ful:${NODE_PORT}
curl http://icc-node-3.ful:${NODE_PORT}
```

Each should result in the following text being returned

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
