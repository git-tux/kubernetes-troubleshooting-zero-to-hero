# Troubleshooting CrashLoopBackOff

## 01-wrong-cmd-crashloop manifest 

### Deploying

I started by deploying the first exercise [01-wrong-cmd-crashloop](https://github.com/iam-veeramalla/kubernetes-troubleshooting-zero-to-hero/blob/main/02-CrashLoopBackOff/01-wrong-cmd-crashloop.yml) which as it was expected it creates a deployment with pods in **CrashLoopBackOff** status:

```
kubectl apply -f 01-wrong-cmd-crashloop.yml
kubectl get deployment crashloop-example
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
crashloop-example   0/1     1            0           8m7s

kubectl get pods
NAME                                 READY   STATUS             RESTARTS        AGE
crashloop-example-5c6cdc7d47-7jg6b   0/1     CrashLoopBackOff   6 (2m16s ago)   8m28s
```

### Troubleshooting

First I checked the events of the failed pod:

```
kubectl describe pod crashloop-example-5c6cdc7d47-7jg6b

Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  10m                 default-scheduler  Successfully assigned default/crashloop-example-5c6cdc7d47-7jg6b to kind-worker
  Normal   Pulling    10m                 kubelet            Pulling image "abhishekf5/crashlooptest:v1"
  Normal   Pulled     10m                 kubelet            Successfully pulled image "abhishekf5/crashlooptest:v1" in 8.31s (8.31s including waiting). Image size: 47542370 bytes.
  Normal   Created    4m5s (x7 over 10m)  kubelet            Created container: crashlooplearning
  Normal   Pulled     4m5s (x6 over 10m)  kubelet            Container image "abhishekf5/crashlooptest:v1" already present on machine
  Normal   Started    4m4s (x7 over 10m)  kubelet            Started container crashlooplearning
  Warning  BackOff    5s (x47 over 10m)   kubelet            Back-off restarting failed container crashlooplearning in pod crashloop-example-5c6cdc7d47-7jg6b_default(bf2a3736-e703-4732-b17a-efd95d032350)
```

The events showed me that the image has been successfully downloaded. Next, I wanted to check the console logs of the running pod:

```
kubectl logs crashloop-example-5c6cdc7d47-7jg6b
python3: can't open file '/app/app1.py': [Errno 2] No such file or directory
```

This showed me that one init command of the image fails because a required file has not been added in the container. 
I built a new image locally and uploaded to Github Packages:

```
cd 02-CrashLoopBackOff/simple-python-app

docker build -t ghcr.io/git-tux/crashlooptest:v1 .
docker login --username git-tux --password $GITHUB_TOKEN ghcr.io
docker push ghcr.io/git-tux/crashlooptest:v1

```

I changed my github package permissions so the new image will be available publicly. Then I edited the deployment and replaced the broken image with the new one:

```
sed -i s/abhishekf5/ghcr.io\\/git-tux/ 01-wrong-cmd-crashloop.yml

kubectl replace -f 01-wrong-cmd-crashloop.yml
```

Which ended up with a running pod:

```
kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
crashloop-example-79b94bf8c4-hc2cm   1/1     Running   0          52s
```

## 02-livenessprobe-crashloop.yml

### Deploying

After fixing the previous senario, I deployed the next manifest [02-livenessprobe-crashloop](https://github.com/iam-veeramalla/kubernetes-troubleshooting-zero-to-hero/blob/main/02-CrashLoopBackOff/02-livenessprobe-crashloop.yml)

```
kubectl replace -f 02-CrashLoopBackOff/02-livenessprobe-crashloop.yml
```

### Troubleshooting

Checking the events of the running pod, I saw that the container is continiously being restarted which ended up in a **crashloopBackOff** state after multiple attempts:

```
kubectl describe  pod crashloop-example-868b476898-b6vb6
  Normal   Killing    13m (x6 over 17m)     kubelet            Container crashlooplearning failed liveness probe, will be restarted
  Warning  Unhealthy  12m (x20 over 17m)    kubelet            Liveness probe failed: cat: /tmp/healthy: No such file or directory
  Normal   Pulled     3m59s (x9 over 17m)   kubelet            Container image "abhishekf5/crashlooptest:v2" already present on machine
  Warning  BackOff    2m11s (x47 over 14m)  kubelet            Back-off restarting failed container crashlooplearning in pod crashloop-example-868b476898-b6vb6_default(864ad3f4-5279-4f4e-95df-7c93ca12b20b)
  ```

The reason here seems to be a wrong liveness probe command. I changed the livenessProbe to use httpGet instead of command:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashloop-example
  labels:
    app: crashlooplearning
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crashlooplearning
  template:
    metadata:
      labels:
        app: crashlooplearning
    spec:
      containers:
      - name: crashlooplearning
        image: abhishekf5/crashlooptest:v2
        ports:
        - containerPort: 8000
        livenessProbe:
          httpGet:
            path: /
            port: 8000
          initialDelaySeconds: 0
          periodSeconds: 1
```

And applied it in the cluster:

```
kubectl replace -f 02-livenessprobe-crashloop.yml

kubectl get pod crashloop-example-6f5c476fd8-jd5f5

  Normal  Scheduled  3m12s  default-scheduler  Successfully assigned default/crashloop-example-6f5c476fd8-jd5f5 to kind-worker3
  Normal  Pulled     3m12s  kubelet            Container image "abhishekf5/crashlooptest:v2" already present on machine
  Normal  Created    3m12s  kubelet            Created container: crashlooplearning
  Normal  Started    3m12s  kubelet            Started container crashlooplearning
```

After this change, I watched the event logs for a few minutes and I noticed that the error was fixed and the container was successfully running

## 03-out-of-memory-crashloop manifest

In this senario, I'm deploying the [03-out-of-memory-crashloop.yml](https://github.com/iam-veeramalla/kubernetes-troubleshooting-zero-to-hero/blob/main/02-CrashLoopBackOff/03-out-of-memory-crashloop.yml) manifest

### Deploying

I will apply the manifest by replacing the already deployed 'crashloop-example'

```
kubectl replace -f 03-out-of-memory-crashloop.yml
```

### Troubleshooting

After applying the configuration, I'm checking the status of the running pods. The pod is in **running** state but it has already been restarted multiple times:

```
kubectl get pods
NAME                                 READY   STATUS    RESTARTS       AGE
crashloop-example-7f99469c89-z7vj6   1/1     Running   7 (7m5s ago)   19m

```

Checking the pod logs and the cluster event logs, I couldn't find anything to indicate the reason of restarting.
By using running kubectl describe, I can see that the pod is being restarted because of the **OOMKiller**. This means that the resource limits of the pod are set too low:

```
kubectl describe pod crashloop-example-7f99469c89-zctf6

Containers:
  crashlooplearning:
    Container ID:   containerd://dd60a1690c7b7c5f20fc584d335b4d4719d5816823dae148147fc6c7735f3c14
    Image:          abhishekf5/crashlooptest:v2
    Image ID:       docker.io/abhishekf5/crashlooptest@sha256:48cf69ef51bbc9ea42944afbabbe011aff6f0fd9e9fdd12f6b84a987ca3e4fe3
    Port:           8000/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 17 May 2025 09:46:04 +0300
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Sat, 17 May 2025 09:04:36 +0300
      Finished:     Sat, 17 May 2025 09:45:58 +0300
    Ready:          True
    Restart Count:  65
```

The above command also shows the cpu/memory limits for this pod:

```
    Limits:
      cpu:     25m
      memory:  25Mi
    Requests:
      cpu:        25m
      memory:     25Mi
```

I will try to raise the memory limits by editing the deployment crashloop-example:

```
    Limits:
      cpu:     25m
      memory:  125Mi
    Requests:
      cpu:        25m
      memory:     125Mi
```

And after some time, I verified that the pod is running without any restarts:

```
kubectl describe pod crashloop-example-58d987647c-d6xnw

    State:          Running
      Started:      Sat, 17 May 2025 10:06:40 +0300
    Ready:          True
    Restart Count:  0
```

