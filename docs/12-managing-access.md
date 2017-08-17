Managing Access
===============

Now that your cluster is running, we need to grant access to users and also introduce
credentials to authenticate to private image repositories for use in the containers.

Private Image Registries
------------------------

How to gain access to a private registry is explained in detail in
[Configuring Nodes to use Private Docker Registry](docs/private-docker-registry.md)

Granting Access for User
------------------------

This user can do almost everything, but only in one namespace

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: ${NAMESPACE}
  name: namespace-owner
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["*"]
  verbs: ["*"]
```

AuthN Options
------------------------
After evaluation of the available options two authentication options stuck out:

1. OpenID Connect. Though OIC would provide a nice way to authenticate against Gitlab
a user would ne to re-issue the commands to retrieve a valid token each time the token looses
its validity

2. Expected to be easier to setup, the kubernetes-ldap Webhook provided by Kismatic
looks promising to act as the AuthN component for the ICC. It should allow to directly
connect to the dir-public LDAP server.



# Setup LDAP AuthN

Users should be able to login via their HAW accounts. To achieve that we utilize an altered version of the
kubernetes-ldap service from Kismatic by Apprenda. The service is written in Golang and provides two endpoints:
* */ldapAuth*: Endpoint to receive login requests from users. Upon retrieval of a request the received credentials
are used to query the LDAP server via bind and if successful a token is created. Finally the service returns a JWT token.
* */authenticate*: Endpoint for the Kubernetes API Server to validate incoming authN attempts by token. Will return
'Success' or 'Failure' depending on the validity of the token.

The service itself is deployed into the *kube-system* namespace of the cluster and exposes
the */ldapAuth* endpoint through an Ingress definition to the outside via https://icc-login.informatik.haw-hamburg.de .
The Ingress server uses a valid SSL certificate to encrypt the communication with the service.

#### The procedure is as follows:

1. The user logs in with a provided 'kubelogin' tool. That tool - essentially a Python Script - fetches the HAW credentials
of the user and sends them to the /ldapAuth endpoint.
2. When successful the *kubelogin* script receives the JWT token and uses that to write a kubeconfig file in the users' home
directory.
3. The user may now use *kubectl* to access the cluster. When issuing a request *kubectl* will send the token alongside with the
request and the Kubernetes API Server is configured to fire a webhook event, which get's caught by the kubernetes-ldap service.
The service then checks the validity of the token and accepts or denies to authenticate the user accordingly.  

#### Webhook Yaml
For the API Server to use the kubernetes-ldap service a webhook config yaml file needs to be specified:

##### /etc/kubernetes/ssl/ldap-webhook-config.yaml
```yaml
# Docs here: https://kubernetes.io/docs/admin/authentication/#webhook-token-authentication
# clusters refers to the remote service.
clusters:
  - name: ldap-auth-webhook
    cluster:
      certificate-authority: /etc/kubernetes/ssl/ca.pem      # Telekom or HAW chain cert goes here!
      server: https://10.32.0.15:8443/authenticate # URL of remote service to query. Must use 'https'.

# users refers to the API Server's webhook configuration.
users:
  - name: ldap-auth-webhook-client
    user:
      client-certificate: /etc/kubernetes/ssl/kubernetes-ldap-client.pem # cert for the webhook plugin to use
      client-key: /etc/kubernetes/ssl/kubernetes-ldap-client.key          # key matching the cert

# kubeconfig files require a context. Provide one for the API Server.
current-context: webhook
contexts:
- context:
    cluster: ldap-auth-webhook
    user: ldap-auth-webhook-client
  name: webhook

```


After a *restart* the API-Server now uses the kubernetes-ldap service via webhook.

### Running the kubernetes-ldap service

The kubernetes-ldap service itself is simply deployed as a K8s Deployment:

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: icc-k8s-ldap
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        service: icc-k8s-ldap
    spec:
      volumes:
        - name: authn-tls-secret
          secret:
            secretName: kubernetes-ldap-server-informatik-haw-hamburg-de
      containers:
      - name: icc-k8s-ldap
        image: nexus.informatik.haw-hamburg.de/icc-k8s-ldap:8014
        args: ["--ldap-host", "YOUR_LDAP_SERVER",
               "--ldap-port", "636",
               "--ldap-base-dn", "ou=users,ou=dep-inf,o=haw-hamburg",
               "--ldap-user-attribute", "uid",
               "--ldap-search-user-dn", "$(SECRET_LDAP_USERNAME)",
               "--ldap-search-user-password", "$(SECRET_LDAP_PASSWORD)",
               "--authn-tls-cert-file", "/etc/kubernetes/ssl/tls.crt",
               "--authn-tls-private-key-file", "/etc/kubernetes/ssl/tls.key"]
        ports:
        - containerPort: 8080
        - containerPort: 8443
        resources:
          limits:
            cpu: 50m
            memory: 90Mi
          requests:
            cpu: 50m
            memory: 90Mi
        env:
        - name: SECRET_LDAP_USERNAME
          valueFrom:
            secretKeyRef:
              name: k8s-ldap-credentials
              key: username
        - name: SECRET_LDAP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: k8s-ldap-credentials
              key: password
        volumeMounts:
          - name: authn-tls-secret
            mountPath: /etc/kubernetes/ssl
            readOnly: true
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3

```

Also a service is required to expose it:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: icc-k8s-ldap
  namespace: kube-system
  labels:
    service: icc-k8s-ldap
spec:
  ports:
  -   name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  -   name: https
      protocol: TCP
      port: 8443
      targetPort: 8443
  selector:
    service: icc-k8s-ldap
  clusterIP: 10.32.0.15

```

And finally an Ingress definition for the */ldapAuth* endpoint to be exposed to the world:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
  name: icc-kube-system-ingress
  namespace: kube-system
spec:
  tls:
  - hosts:
    - login.YOUR_DOMAIN
    secretName: icc-login-informatik-haw-hamburg-de
  backend:
        serviceName: icc-k8s-ldap
        servicePort: http
  rules:
  - host: login.YOUR_DOMAIN
    http:
      paths:
      - path: /ldapAuth
        backend:
            serviceName: icc-k8s-ldap
            servicePort: http
```


Pod Security
------------------------
Carefully consider Pod Security Policies in order to manage what users are allowed to do
with their pods (run priviled etc.)
--> https://kubernetes.io/docs/concepts/policy/pod-security-policy/
