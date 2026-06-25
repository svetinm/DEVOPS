### LO5 — Solve problems with application shipping by using containers
## For each command: identify the mistake, fix it, and explain what was wrong.

## 1. podman run -d --name web -p 80:8080 docker.io/library/nginx
(nginx listens on 80; the site isn't reachable on the host port you expected — what's reversed?)

podman run -d --name web -p 8080:80 docker.io/library/nginx
podman ps
podman port web
podman logs web
podman exec web ss -tln


2. podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql
(the container exits during init — inspect podman logs.)

podman run -d --name db -e MYSQL_ROOT_PASSWORD=MySecret123 docker.io/library/mysql
podman ps -a
podman logs db


3. podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
(why is the -p flag effectively ignored here?)

podman run -d --name app --network host docker.io/library/nginx
podman run -d --name app -p 8080:80 docker.io/library/nginx
Because --network host bypasses network isolation and port forwarding


4. podman run -d --name c1 docker.io/library/busybox
(the container goes straight to Exited (0) — why, and how do you keep it running?)

podman run -d --name c1 docker.io/library/busybox sleep 1d
podman run -d --name c1 docker.io/library/busybox tail -f /dev/null
podman ps -a
podman logs c1


5. podman run --rm -d --name job docker.io/library/alpine echo hello
(then podman logs job fails — explain the --rm + detached gotcha.)

The command echo hello finishes immediately. Since --rm is used, Podman automatically removes the container after it exits. Therefore, podman logs job fails because the container no longer exists.
podman run -d --name job docker.io/library/alpine echo hello


6. podman run -d -p 8080:80 --memory 8m docker.io/library/mysql
(the database never becomes healthy — what limit is the problem?)

The memory limit (--memory 8m) is too low. MySQL requires much more than 8 MB to start, so the container cannot initialize and never becomes healthy.

podman run -d --memory 512m docker.io/library/mysql or podman run -d docker.io/library/mysql

7. podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b
(name resolution fails — why, on the default network, and how do you fix it?)

Containers on the default network cannot resolve each other's names. DNS resolution for container names works only on a user-defined network.

podman network create mynet

podman run -d --name a --network mynet docker.io/library/alpine sleep 1d
podman run -d --name b --network mynet docker.io/library/alpine sleep 1d

podman exec a ping b

8. podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx
(the files aren't visible in the container, or SELinux denies access — what mount flag is missing?)

podman run -d --name web \
-v ./html:/usr/share/nginx/html:Z \
docker.io/library/nginx

Explanation
:Z → Private SELinux label (used by one container).
:z → Shared SELinux label (used by multiple containers).

Without these flags, the container may not be able to read or write the mounted files.


9. For each Containerfile/Dockerfile: identify the mistake, fix it, and explain what was wrong.

10. FROM debian:12
RUN apt-get install -y nginx
(the build fails to find the package — what's missing before install?)

apt-get install is executed without first updating the package index. As a result, the build may fail because the package lists are outdated or missing.

FROM debian:12
RUN apt-get update && apt-get install -y nginx

11. FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD python app.py
(the app starts but doesn't handle signals / can't be stopped cleanly — shell vs exec form?)

FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD ["python", "app.py"]

CMD python app.py uses the shell form. The shell becomes PID 1, so Python does not receive signals (e.g., SIGTERM) properly, making the container harder to stop cleanly.

12. FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
(every source change triggers a full npm install — reorder for layer caching.)

COPY . /app copies all source files before npm install. Any change in the source code invalidates the Docker cache, causing npm install to run again.

FROM node:20
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

package.json changes less often than the application code.
Docker reuses the cached npm install layer if dependencies haven't changed.
Only the final COPY . . layer is rebuilt when source files change

13. FROM alpine:3.20
ENV PATH=/app/bin
RUN apk add --no-cache curl
(after this, curl/sh aren't found — what did setting PATH break?)

PATH is overwritten with /app/bin. System directories like /usr/bin and /bin are removed, so commands such as curl and sh cannot be found.

FROM alpine:3.20
ENV PATH="/app/bin:${PATH}"
RUN apk add --no-cache curl

14. FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]


The application listens on port 3000, but EXPOSE declares port 8080. This mismatch can cause confusion when mapping ports (e.g., -p 8080:8080 won't work).

FROM ubuntu:24.04
EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]


15. (you map -p 8080:8080 but nothing answers — what mismatch is there, and what does EXPOSE actually
do?)

Documents which port the application is expected to use.
Helps with readability and some container tools.
Does not publish or open the port


16. FROM debian:12
RUN apt-get update && apt-get install -y build-essential
(the image is huge — how do you cut the size in the same layer?)

FROM debian:12
RUN apt-get update && \
    apt-get install -y build-essential && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

The image is large because the APT package cache is left inside the image after installation.


17. FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
(permission denied errors — what's wrong with the USER/COPY ordering and ownership?)

USER appuser is set before copying files and installing packages. The non-root user may not have permission to write to /app or install Python packages, causing Permission denied errors.

FROM python:3.11

COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt

USER appuser


18. FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
(the final image is ~1 GB — how would a multi-stage build fix it?)

The final image contains the Go compiler, build tools, source code, and dependencies, making it very large (~1 GB).

# Build stage
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o app .

# Runtime stage
FROM alpine:3.20
COPY --from=builder /src/app /app
CMD ["/app"]



19. A pod is stuck Pending. Walk through the diagnosis with kubectl describe pod and identify the reason from
the events (scheduling / resources / PVC / nodeSelector).

Which command is used to diagnose a Pending Pod?

A:

kubectl describe pod <pod-name>

Q: Where do you find the cause?

A: In the Events section of kubectl describe pod.

Q: Name three common reasons for a Pod being Pending.

A:

Insufficient CPU or memory.
Missing or unbound PVC.
nodeSelector does not match any node.



20. Find a deployment from older tasks, remove a couple of spaces and try to deploy it. Identify the cause in the
error messages and fix it.

You accidentally remove a few spaces from a Deployment YAML file. Since YAML is indentation-sensitive, Kubernetes cannot parse the manifest.

YAML uses spaces to define structure. Missing or incorrect indentation changes the hierarchy, making the file invalid. YAML needs good spacing

21. A pod is in ImagePullBackOff. Diagnose the cause (image typo, missing tag, private registry without pull
secret) and fix it.

A Pod is in ImagePullBackOff because Kubernetes cannot pull the container image.

Image name typo.
Invalid or missing image tag.
Private registry without an imagePullSecret.


22. A pod is in CrashLoopBackOff. Use kubectl logs --previous to read the last crash and fix the root cause.

The Pod is in CrashLoopBackOff because the container starts, crashes, and Kubernetes keeps restarting it.

kubectl logs --previous <pod-name>

CrashLoopBackOff means the application inside the container keeps crashing. Use kubectl logs --previous to identify the error, fix the underlying issue (configuration, command, environment variables, missing files, etc.), and redeploy.

23. A pod's last state shows OOMKilled. Confirm it from kubectl get pod -o yaml / describe, then fix it (raise the
memory limit or fix the app).

kubectl describe pod <pod>
kubectl get pod <pod> -o yaml

Last State: Terminated
Reason: OOMKilled

Fix

Increase memory limit.
Optimize the application.

24. A Deployment's pods never become Ready. Trace it to a failing readiness probe and correct the probe or
the app.

kubectl describe pod <pod>
kubectl logs <pod>

Correct the readiness probe.
Ensure the application listens on the correct port/path.


25. A Service returns nothing. Diagnose a Service/pod label mismatch using kubectl get endpoints.

kubectl get svc
kubectl get pods --show-labels
kubectl get endpoints

If:

ENDPOINTS: <none>

→ labels don't match.

Fix

Match Service selector with Pod labels.




26. Given a manifest where spec.selector.matchLabels doesn't match spec.template.metadata.labels, explain
the error kubectl apply returns and fix it.

Problem

selector:
  matchLabels:
    app: web

template:
  metadata:
    labels:
      app: nginx

Error

selector does not match template labels

Fix
Both labels must be identical.


27. Given a manifest whose resources.requests exceed any node's capacity, explain why the pod is Pending
(Insufficient cpu/memory) and fix it.

Diagnose

kubectl describe pod <pod>

Example:

Insufficient memory

Fix

Lower resources.requests
Add node resources


28. Given a pod mounting a PVC that doesn't exist, diagnose the FailedMount/Pending and create the PVC.

Diagnose

kubectl describe pod <pod>
kubectl get pvc

Example:

FailedMount
PersistentVolumeClaim not found

Fix
Create the PVC or reference the correct PVC.

29. A container based on ubuntu with no long-running command shows Completed/CrashLoopBackOff. Explain
why and add a proper command.

Problem
Ubuntu has no long-running process.

Fix

sleep infinity

or

tail -f /dev/null


30. Use kubectl get events --sort-by=.lastTimestamp (or kubectl events) to triage everything that happened to a
pod over time.

kubectl get events --sort-by=.lastTimestamp

or

kubectl events

Shows the complete event history for troubleshooting


31. Use kubectl exec -it to get a shell in a pod and debug DNS with nslookup and cat /etc/resolv.conf.

Enter the Pod:

kubectl exec -it <pod> -- sh

Check DNS:

nslookup service-name
cat /etc/resolv.conf


32. Use an ephemeral debug container (kubectl debug -it <pod> --image=busybox --target=<container>) to
troubleshoot a no-shell/distroless container.

kubectl debug -it <pod> \
--image=busybox \
--target=<container>

Adds a temporary debug container.


33. Spin up a throwaway kubectl run tmp --rm -it --image=busybox pod to test connectivity to a Service from
inside the cluster.

kubectl run tmp \
--rm -it \
--image=busybox

Inside:

wget service-name

or

nslookup service-name

34. A node shows NotReady. List what you'd check (kubelet, disk pressure, CNI) and inspect it on minikube with
kubectl describe node and sudo minikube logs.

Check

kubectl describe node <node>

Look for:

kubelet
DiskPressure
MemoryPressure
PIDPressure
Network/CNI

Minikube:

sudo minikube logs

35. A pod is stuck Terminating. Explain finalizers and the grace period, then force-delete it (--grace-period=0 --
force) and state the risks.

Cause

Finalizers
Long grace period

Force delete

kubectl delete pod <pod> \
--grace-period=0 \
--force

Risk

Application may not shut down cleanly and data may be lost.


36. Given a manifest with the wrong apiVersion/kind pairing (e.g. apps/v1 + Pod), explain the validation error
and correct the group/version.

Wrong:

apiVersion: apps/v1
kind: Pod

Correct:

apiVersion: v1
kind: Pod

or

apiVersion: apps/v1
kind: Deployment

37. A pod fails with CreateContainerConfigError because a configMapKeyRef key is missing. Diagnose and fix.

Cause

Missing ConfigMap key.

Diagnose:

kubectl describe pod <pod>
kubectl get configmap

Fix:

Add missing key.
Correct configMapKeyRef.

38. A pod can't mount a Secret that lives in a different namespace. Explain namespace scoping and fix it.

Secrets are namespace-scoped.

A Pod cannot mount a Secret from another namespace.

Fix

Create the Secret in the same namespace.
Move the Pod to the correct namespace.


39. A rollout is stuck with ProgressDeadlineExceeded. Use kubectl describe deployment / rollout status to find
the cause and remediate.

Diagnose:

kubectl rollout status deployment <deployment>
kubectl describe deployment <deployment>

Common causes:

Pods not Ready
ImagePullBackOff
CrashLoopBackOff

Fix the root cause and redeploy.

40. Catch indentation/field typos (e.g. imagePullpolicy, misnested ports) before applying with kubectl apply --
dry-run=server / --validate=true, then fix them.

Before applying:

kubectl apply \
--dry-run=server \
-f deployment.yaml

or

kubectl apply \
--validate=true \
-f deployment.yaml

Finds:

indentation errors
unknown fields
spelling mistakes

41. An app's logs say it can't reach its database Service. Verify in order: pod running → Service exists →
endpoints populated → DNS resolves → correct port — and identify the broken link.

Check in this order:

kubectl get pods

↓

kubectl get svc

↓

kubectl get endpoints

↓

nslookup database-service

↓

Verify Service port matches container port.


42. Read a container's state/lastState and restartCount with kubectl get pod -o jsonpath to determine the root
cause of repeated restarts.

Find Restart Cause
kubectl get pod <pod> \
-o jsonpath='{.status.containerStatuses[*].restartCount}'

Last state:

kubectl get pod <pod> \
-o jsonpath='{.status.containerStatuses[*].lastState}'

Useful for identifying:

OOMKilled
Error
CrashLoopBackOff

43. Compare OpenShift Route to Ingress

Route	Ingress
OpenShift only	Standard Kubernetes
Exposes Service externally	Exposes Service externally
Simpler in OpenShift	Requires Ingress Controller
Supports TLS	Supports TLS

Summary

Route = OpenShift-specific way to expose a Service.
Ingress = Standard Kubernetes resource for exposing HTTP/HTTPS services.