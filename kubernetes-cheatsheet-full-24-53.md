# Kubernetes Deployments - LO4

## Create a Deployment

**Q:** Create a Deployment named `web` running `nginx:1.25` with 3 replicas using a single imperative `kubectl create deployment` command, then export its generated YAML to a file.

**A:**

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web-deployment.yaml
```

---

## DockerHub Authentication

**Q:** Enable pulling images from DockerHub as an authenticated user to lower the chance of hitting the pull limit. What kind of resource do you need to create?

**A:**

Create a Kubernetes Secret of type `docker-registry` (`kubernetes.io/dockerconfigjson`) image pull secret.

---

## Scale a Deployment

**Q:** Scale `web` from 3 to 5 replicas two different ways — once with `kubectl scale`, once by editing the manifest.

**A:**

```bash
kubectl scale deployment web --replicas=5
```

or edit the manifest:

```yaml
spec:
  replicas: 5
```

Apply:

```bash
kubectl apply -f web-deployment.yaml
```

---

## Rolling Update

**Q:** Perform a rolling update of `web` from `nginx:1.25` to `nginx:1.27` and watch it with `kubectl rollout status`.

**A:**

```bash
kubectl set image deployment/web nginx=docker.io/library/nginx:1.27
kubectl rollout status deployment/web
```

---

## Rollout History and Rollback

**Q:** View a Deployment's rollout history and roll back to the previous revision. Explain what the `CHANGE-CAUSE` column shows and how to populate it.

**A:**

View history:

```bash
kubectl rollout history deployment/web
```

Add change cause:

```bash
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade nginx 1.25 -> 1.27" \
  --overwrite
```

Rollback:

```bash
kubectl rollout undo deployment/web
```

`CHANGE-CAUSE` shows the reason for the revision. If it shows `<none>`, no annotation was added.

---

## RollingUpdate Strategy

**Q:** Set the update strategy to `RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 0`, and explain the effect on availability during an update.

**A:**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

`maxSurge: 1` allows one extra Pod during updates, while `maxUnavailable: 0` ensures no Pods are unavailable, enabling zero-downtime updates.

---

## Recreate Strategy

**Q:** Switch a Deployment's strategy to `Recreate` and describe a concrete scenario where this is required instead of `RollingUpdate`.

**A:**

```yaml
spec:
  strategy:
    type: Recreate
```

All old Pods are terminated before new Pods are created. Useful when old and new application versions cannot run simultaneously (e.g., incompatible database schema changes).

---

## Resource Requests and Limits

**Q:** Add CPU/memory requests and limits to a Deployment's container and verify they appear in the running Pod spec.

**A:**

```bash
kubectl set resources deployment/web \
  -c nginx \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi
```

Verify:

```bash
kubectl describe pod <pod-name>
```

---

## Revision History Limit

**Q:** Set `revisionHistoryLimit: 3` and explain how it affects your ability to roll back and how many old ReplicaSets are kept.

**A:**

```yaml
spec:
  revisionHistoryLimit: 3
```

Only the three most recent previous revisions are retained. Rollback is only possible to revisions that still exist.

---

## Failed Image Rollout

**Q:** Set a Deployment's image to a non-existent tag, observe the rollout get stuck, and explain why the old Pods keep serving traffic.

**A:**

```bash
kubectl set image deployment/web nginx=nginx:does-not-exist
```

The new Pods cannot start because the image tag does not exist. The old Pods continue serving traffic until replacement Pods become Ready.

---

## Label Selectors

**Q:** Use a label selector to list only the Pods belonging to one Deployment, and explain the Deployment → ReplicaSet → Pod selector relationship.

**A:**

```bash
kubectl get pods -l app=web
```

Deployment → ReplicaSet → Pods. The Deployment manages a ReplicaSet using label selectors, and the ReplicaSet manages Pods with matching labels.

---

## Expose a Deployment

**Q:** Expose a Deployment with `kubectl expose`, then explain what object was created and how its selector was derived.

**A:**

```bash
kubectl expose deployment web --port=80 --target-port=80
```

Creates a Service (default type: ClusterIP). The selector is automatically derived from the Deployment's Pod labels.

---

## Sidecar Container

**Q:** Add a sidecar (second) container to a Deployment's Pod template and explain how the two containers share the Pod network and volumes.

**A:**

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.27

    - name: sidecar
      image: busybox
      command:
        - sh
        - -c
        - while true; do sleep 30; done
```

Containers in the same Pod share the same IP address, network namespace, and any mounted volumes.

---

## Deployment with nodeSelector

**Q:** Write a Deployment manifest for `httpd:2.4` with 2 replicas, a named container port, and a `nodeSelector`; explain why it stays Pending if no node matches.

**A:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: httpd

  template:
    metadata:
      labels:
        app: httpd

    spec:
      nodeSelector:
        disktype: ssd

      containers:
        - name: httpd
          image: httpd:2.4

          ports:
            - name: http
              containerPort: 80
```

If no node has the label `disktype=ssd`, the scheduler cannot place the Pods and they remain in the `Pending` state.

---

## Bare Pod vs Deployment-Managed Pod

**Q:** Create a bare Pod (no controller) running BusyBox that sleeps, then explain what happens when you delete it versus a Deployment-managed Pod.

**A:**

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: busybox-sleep

spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3600"
```

Create:

```bash
kubectl apply -f busybox-pod.yaml
```

A bare Pod is permanently removed when deleted. A Deployment-managed Pod is automatically recreated by its ReplicaSet.

# Kubernetes Exam Cheat Sheet (24–53)

24. Mount a ConfigMap as a volume so each key becomes a file, and verify the contents inside the pod.


```bash
kubectl create configmap app-config \
  --from-literal=config.txt="Hello ConfigMap" \
  --from-literal=db.txt="mysql"
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: busybox
    image: docker.io/library/busybox:latest
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Verify:
```bash
kubectl exec configmap-pod -- ls /etc/config
kubectl exec configmap-pod -- cat /etc/config/config.txt
```

Mount a single ConfigMap key to a specific path using subPath; explain when it's needed and its update
caveat

```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/myapp/config.txt
  subPath: config.txt
```

Verify:
```bash
kubectl exec pod-name -- cat /etc/myapp/config.txt
```

Explanation: updates to ConfigMap are not automatically reflected when using subPath.

26. Mount a Secret as a volume and verify the files contain decoded values with restrictive permissions.


```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secret
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
```

Verify:
```bash
kubectl exec secret-pod -- cat /etc/secret/username
kubectl exec secret-pod -- ls -l /etc/secret
```

Show that scaling a StatefulSet creates one PVC per replica, then explain what happens to those PVCs
when the StatefulSet is deleted

```bash
kubectl scale sts redis --replicas=3
kubectl get pvc
```

Expected:
```text
data-redis-0
data-redis-1
data-redis-2
```

Delete StatefulSet:

```bash
kubectl delete sts redis
kubectl get pvc
```

PVCs remain.

28. Use an initContainer to pre-populate data into a shared volume before the main container reads it.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}

  initContainers:
  - name: init
    image: docker.io/library/busybox:latest
    command:
    - sh
    - -c
    - |
      echo "Hello from initContainer!" > /data/message.txt
    volumeMounts:
    - name: shared-data
      mountPath: /data

  containers:
  - name: main
    image: docker.io/library/busybox:latest
    command:
    - sh
    - -c
    - |
      echo "Reading file..."
      cat /data/message.txt
      sleep 3600
    volumeMounts:
    - name: shared-data
      mountPath: /data
```

Verify:
```bash
kubectl exec pod-name -- cat /data/file.txt
```

29. Set readOnly: true on a volume mount and prove writes are rejected.

```yaml
volumeMounts:
- name: config
  mountPath: /config
  readOnly: true

apiVersion: v1
kind: Pod
metadata:
  name: readonly-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}

  containers:
  - name: busybox
    image: docker.io/library/busybox:latest
    command:
    - sh
    - -c
    - sleep 3600

    volumeMounts:
    - name: shared-data
      mountPath: /data
      readOnly: true
```

Verify:

```bash
echo test > /data/test
```

Result:
```text
Read-only file system
```

30. Set a sizeLimit on an emptyDir and explain what happens if the container exceeds it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-limit
spec:
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 10Mi

  containers:
  - name: busybox
    image: docker.io/library/busybox:latest
    command:
    - sh
    - -c
    - sleep 3600

    volumeMounts:
    - name: cache
      mountPath: /cache
```

If exceeded:

```text
dd if=/dev/zero of=/cache/bigfile bs=1M count=20
No space left on device
The emptyDir volume is configured with a sizeLimit of 10Mi. If the runtime enforces the limit, writes beyond 10 MiB fail with "No space left on device". On some environments, such as certain Minikube or CRI-O configurations, the limit may be visible in the Pod specification but not strictly enforced by the runtime.
```

## Create a generic Secret from literals for a DB username/password, then show the stored values are base64-encoded, not encrypted.

```bash
kubectl create secret generic db --from-literal=username=admin --from-literal=password=secret

kubectl get secret db -o yaml
```

Output:

```yaml
username: YWRtaW4=
password: c2VjcmV0
```

Base64 encoded, not encrypted.

## 32. Create a Secret from files (--from-file) holding a certificate and key, and identify the resulting keys.

```bash
echo "dummy certificate" > tls.crt
echo "dummy private key" > tls.key
kubectl create secret generic tls-files --from-file=tls.crt --from-file=tls.key
```
kubectl get secret  tls-files -o yaml
Keys:

```text
tls.crt
tls.key
```

## 33. Create a kubernetes.io/tls typed Secret from a cert/key pair and explain where it's consumed.


```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=example.com"


kubectl create secret tls my-tls \
 --cert=tls.crt \
 --key=tls.key
```

Used by Ingress controllers.

## 34. Secret volume vs env vars

kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret

apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: busybox
    image: docker.io/library/busybox:latest
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret

kubectl apply -f secret-volume.yaml

kubectl exec secret-demo -- ls /etc/secret

kubectl exec secret-demo -- cat /etc/secret/username
kubectl exec secret-demo -- cat /etc/secret/password

Volume:
- Better security
- Can update automatically

Env vars:
- Easier access
- Visible in process environment

## 35. envFrom

kubectl create secret generic db-secret \
  --from-literal=USERNAME=admin \
  --from-literal=PASSWORD=secret123

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-demo
spec:
  containers:
  - name: busybox
    image: docker.io/library/busybox:latest
    command: ["sh","-c","sleep 3600"]

    envFrom:
    - secretRef:
        name: db-secret
```

Verify:

```bash
kubectl exec pod-name -- env
```

## 36. 35. Use envFrom with a secretRef to load every key of a Secret as env vars at once.

```bash
kubectl create secret docker-registry regcred \
 --docker-server=docker.io \
 --docker-username=user \
 --docker-password=pass
```
kubectl get secret regcred


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image
spec:
  imagePullSecrets:
  - name: regcred

  containers:
  - name: nginx
    image: docker.io/<username>/my-private-image:latest
```

## 37. Create a Secret with several keys and selectively mount only one of them into a pod.
kubectl create secret generic app-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123 \
  --from-literal=token=abcd1234

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-items-demo
spec:
  containers:
  - name: busybox
    image: docker.io/library/busybox:latest
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
      items:
      - key: username
        path: username
```

##  Create a ClusterIP Service for a Deployment and resolve it by DNS from another pod (nslookup <svc>.<ns>.svc.cluster.local)

```bash
kubectl expose deployment web --port=80 --target-port=80
```

Verify:

```bash
kubectl run tmp --rm -it --image=busybox -- sh
nslookup web.default.svc.cluster.local
wget -qO- http://web
```

## 39. Create a NodePort Service and reach it via minikube service <svc> --url and $(minikube ip):<nodePort>.

kubectl create deployment web --image=docker.io/library/nginx:1.27

kubectl expose deployment web \
  --type=NodePort \
  --port=80 \
  --target-port=80

kubectl get svc web

minikube service web --url
minikube ip

```yaml
spec:
  type: NodePort
```

Verify:

```bash
minikube service web --url
curl http://$(minikube ip):<nodePort>
```

## Create a LoadBalancer Service, explain why EXTERNAL-IP is <pending> on minikube, and obtain access with minikube tunnel.

kubectl create deployment web --image=docker.io/library/nginx:1.27

kubectl expose deployment web \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80

kubectl get svc web
minikube tunnel
kubectl get svc web
curl http://<EXTERNAL-IP>

```yaml
spec:
  type: LoadBalancer
```

```bash
kubectl get svc
```

EXTERNAL-IP shows pending.

```bash
minikube tunnel
```

##  Create a headless Service (clusterIP: None) for a StatefulSet and show it returns per-pod A records instead of a single VIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```
kubectl apply -f headless-service.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: docker.io/library/redis:7
        ports:
        - containerPort: 6379

kubectl get pods

kubectl get svc redis
kubectl run dns-test --rm -it --image=docker.io/library/busybox:latest -- sh

Verify:

```bash
nslookup redis
```

Returns pod DNS records.

## 42. 42. Inspect a Service's Endpoints/EndpointSlice and explain how they're populated from the pod selector.


kubectl create deployment web --image=docker.io/library/nginx:1.27

kubectl expose deployment web --port=80 --target-port=80

```bash
kubectl get endpoints web
kubectl get endpointslices
```
A Service uses its selector to find matching Pods. Kubernetes automatically creates and updates the Endpoints and EndpointSlices with the IP addresses of all Pods whose labels match the Service selector. If no Pods match the selector, the Service has no endpoints and cannot forward traffic

## Demonstrate cross-namespace access using the FQDN, and show the short name fails from another
namespace
kubectl create namespace test

kubectl create deployment web --image=docker.io/library/nginx:1.27
kubectl expose deployment web --port=80 --target-port=80

kubectl run dns-test \
  -n test \
  --rm -it \
  --image=docker.io/library/busybox:latest -- sh

wget -qO- http://web
nslookup web

Fails:

```bash
wget web
```

Works:

```bash
wget web.default.svc.cluster.local
```

## 44. 44. Compare kubectl port-forward to a Service versus to a Pod — explain the difference and when each fits.

Pod:

```bash
kubectl port-forward pod/web-xxx 8080:80
```

Service:

```bash
kubectl port-forward svc/web 8080:80
```
Pod

Connects directly to one specific Pod.
Useful for debugging or testing a single Pod.

Service

Connects through the Service.
Uses the Service's selector/endpoints to reach the application.
Better for testing the application as it is exposed inside the cluster.

## 45. port targetPort nodePort

```yaml
ports:
- port: 80
  targetPort: 8080
  nodePort: 30080
```
port – The port exposed by the Service inside the cluster.
targetPort – The port on the Pod/container where the application is listening.
nodePort – The port exposed on every Kubernetes node for external access (only for NodePort and LoadBalancer Services).

## Verify connectivity to a ClusterIP Service from a throwaway debug pod (kubectl run tmp --rm -it --image=busybox -- sh) with wget/nc.

```bash
kubectl create deployment web --image=docker.io/library/nginx:1.27
kubectl expose deployment web --port=80 --target-port=80


kubectl run tmp --rm -it --image=docker.io/library/busybox:latest -- sh
wget -qO- http://web
nc -vz web 80
```

## Create two Deployments and one Service that load-balances across both by sharing a common label; prove requests hit both.

Deployment labels:
kubectl create deployment web1 --image=docker.io/library/nginx:1.27
kubectl create deployment web2 --image=docker.io/library/nginx:1.27

kubectl label deployment web1 app=demo --overwrite
kubectl label deployment web2 app=demo --overwrite

kubectl expose deployment web1 --name=demo-service --port=80 --target-port=80

kubectl edit svc demo-service

```yaml
labels:
  app: demo
```

Service:

```yaml
selector:
  app: demo
```

Verify:

```bash
kubectl get endpoints
```

## 48. Diagnose Service issues

```bash
# 1. Check DNS
nslookup <service-name>

# 2. Check Service endpoints
kubectl get endpoints <service-name>

# 3. Check Service selector
kubectl describe svc <service-name>

# 4. Check Pod labels
kubectl get pods --show-labels

# 5. Check Pod readiness and container ports
kubectl describe pod <pod-name>
```
To troubleshoot Service connectivity, first verify DNS resolution, then check that the Service has endpoints. Next, confirm that the Service selector matches the Pod labels, ensure the Pod is Ready, and finally verify that the Service port and targetPort match the container's listening port.

## 49. kubectl create deployment web --image=docker.io/library/nginx:1.27

```bash
kubectl create deployment web --image=docker.io/library/nginx:1.27 
kubectl expose deployment web --type=NodePort --port=80  --target-port=80
kubectl get svc web

minikube service web --url
```

## 50. Add an HTTP livenessProbe hitting / on port 80 to an nginx Deployment and verify via kubectl describe pod.

kubectl create deployment web --image=docker.io/library/nginx:1.27

containers:
- name: nginx
  image: docker.io/library/nginx:1.27

  livenessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 10

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5

kubectl apply -f web.yaml

```

Verify:

```bash
kubectl describe pod <pod>
```

## 51. Add a tcpSocket readiness probe to a redis container on port 6379 and verify it.


kubectl create deployment redis --image=docker.io/library/redis:7
```yaml
containers:
- name: redis
  image: docker.io/library/redis:7

  readinessProbe:
    tcpSocket:
      port: 6379
    initialDelaySeconds: 5
    periodSeconds: 10
```
kubectl apply -f redis.yaml

kubectl describe pod <pod-name>

```yaml
readinessProbe:
  tcpSocket:
    port: 6379
  periodSeconds: 5
```

## Configure a startup probe with a failureThreshold * periodSeconds budget for a ~2-minute boot and justify the numbers
kubectl create deployment web --image=docker.io/library/nginx:1.27

kubectl edit deployment web

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 12
```

12 x 10 = 120 seconds startup budget.

## 53. No probes

- Pod becomes ready when container starts.
- Kubernetes does not check application health.
- Hung applications are not restarted automatically.
