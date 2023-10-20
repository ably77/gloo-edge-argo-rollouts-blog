# Install the argo-rollouts controller
Install the kubectl argo-rollouts plugin as described [here](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)

Instructions for brew
```
brew install argoproj/tap/kubectl-argo-rollouts
```

Create argo-rollouts namespace
```
kubectl create namespace argo-rollouts
```

Save the following as `kustomization.yaml`
```
cat > kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argo-rollouts
resources:
- https://raw.githubusercontent.com/argoproj/argo-rollouts/v1.5.1/manifests/install.yaml
patchesJson6902:
  - target:
      kind: ClusterRole
      name: argo-rollouts
      version: v1
    patch: |
      - op: add
        path: /rules/-
        value:
          apiGroups:
          - gateway.solo.io
          resources:
          - virtualservices
          - routetables
          verbs:
          - '*'
  - target:
      kind: ConfigMap
      name: argo-rollouts-config
      version: v1
    patch: |
      - op: add
        path: /data
        value:
          trafficRouterPlugins: |
            - name: "solo-io/glooedge"
              location: "https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-glooedge/releases/download/v0.1.0-beta1/glooedge-plugin-linux-amd64"
EOF
```

Now apply the kustomize using `kubectl apply -k` to deploy the argo-rollouts controller into the `argo-rollouts` namespace
```
kubectl apply -k .
```

Check to see if your argo-rollouts controller has been deployed:
```
% kubectl get pods -n argo-rollouts
NAME                            READY   STATUS    RESTARTS   AGE
argo-rollouts-84995876f-whzv6   1/1     Running   0          92m
```

You can see in the logs of the argo-rollouts pod that the `solo-io/glooedge` plugin was loaded in the output below
```
% kubectl logs -n argo-rollouts deploy/argo-rollouts
time="2023-10-17T23:03:26Z" level=info msg="Argo Rollouts starting" version=v1.2.0+5597ae6
time="2023-10-17T23:03:26Z" level=info msg="Creating event broadcaster"
time="2023-10-17T23:03:26Z" level=info msg="Setting up event handlers"
time="2023-10-17T23:03:26Z" level=info msg="Setting up experiments event handlers"
time="2023-10-17T23:03:26Z" level=info msg="Setting up analysis event handlers"
time="2023-10-17T23:03:26Z" level=info msg="Downloading plugin solo-io/glooedge from: https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-glooedge/releases/download/v0.1.0-beta1/glooedge-plugin-linux-amd64"
time="2023-10-17T23:03:31Z" level=info msg="Download complete, it took 4.224423168s"
time="2023-10-17T23:03:31Z" level=info msg="Leaderelection get id argo-rollouts-84995876f-hmksr_db4aeb8c-ef8e-45f3-8e3e-d4419306078e"
time="2023-10-17T23:03:31Z" level=info msg="attempting to acquire leader lease argo-rollouts/argo-rollouts-controller-lock...\n"
time="2023-10-17T23:03:31Z" level=info msg="Starting Healthz Server at 0.0.0.0:8080"
time="2023-10-17T23:03:31Z" level=info msg="Starting Metric Server at 0.0.0.0:8090"
time="2023-10-17T23:03:31Z" level=info msg="successfully acquired lease argo-rollouts/argo-rollouts-controller-lock\n"
time="2023-10-17T23:03:31Z" level=info msg="I am the new leader: argo-rollouts-84995876f-hmksr_db4aeb8c-ef8e-45f3-8e3e-d4419306078e"
time="2023-10-17T23:03:31Z" level=info msg="Starting Controllers"
time="2023-10-17T23:03:31Z" level=info msg="New leader elected: argo-rollouts-84995876f-hmksr_db4aeb8c-ef8e-45f3-8e3e-d4419306078e"
time="2023-10-17T23:03:31Z" level=info msg="invalidated cache for resource in namespace: argo-rollouts with the name: argo-rollouts-notification-secret"
time="2023-10-17T23:03:31Z" level=info msg="Waiting for controller's informer caches to sync"
time="2023-10-17T23:03:31Z" level=info msg="Started controller"
time="2023-10-17T23:03:31Z" level=info msg="Starting Service workers"
time="2023-10-17T23:03:31Z" level=info msg="Starting Experiment workers"
time="2023-10-17T23:03:31Z" level=info msg="Started Service workers"
time="2023-10-17T23:03:31Z" level=info msg="Started Experiment workers"
time="2023-10-17T23:03:31Z" level=info msg="Starting Ingress workers"
time="2023-10-17T23:03:31Z" level=info msg="Started Ingress workers"
time="2023-10-17T23:03:31Z" level=info msg="Starting analysis workers"
time="2023-10-17T23:03:31Z" level=warning msg="Controller is running."
time="2023-10-17T23:03:31Z" level=info msg="Started 30 analysis workers"
time="2023-10-17T23:03:31Z" level=info msg="Starting Rollout workers"
time="2023-10-17T23:03:31Z" level=info msg="Started rollout workers"
```

# Install Gloo Edge
Install `glooctl` with brew:
```
# with brew
brew install glooctl

# manually
export GLOO_VERSION=v1.15.12
curl -sL https://run.solo.io/gloo/install | sh
export PATH=$HOME/.gloo/bin:$PATH
```

Install Gloo Edge with `glooctl`:
```
glooctl install gateway --version 1.15.12 --create-namespace
```

Install Gloo Edge with Helm
```
helm repo add gloo https://storage.googleapis.com/solo-public-helm
helm repo update
helm upgrade --install gloo gloo/gloo --namespace gloo-system --version 1.15.12 --create-namespace
```

Check to see if Gloo Edge has been deployed
```
% kubectl get pods -n gloo-system
NAME                                READY   STATUS      RESTARTS   AGE
gateway-certgen-jzwws               0/1     Completed   0          56s
svclb-gateway-proxy-pd47g           2/2     Running     0          48s
gateway-proxy-5d78cdcb85-nvbj2      1/1     Running     0          48s
discovery-54cbf678c9-2pz6s          1/1     Running     0          48s
gloo-f445dd58b-8lszg                1/1     Running     0          48s
gloo-resource-rollout-check-c45vn   0/1     Completed   0          47s
```

# Blue-green Rollout Strategy

A Blue Green Deployment allows users to reduce the amount of time multiple versions running at the same time. This means that in a blue/green rollout that the traffic will be shifted from blue > green as soon as the green application is ready to take on traffic if `autoPromotionEnabled: true`. Additionally, a user can set the following [Configurable Features](https://argo-rollouts.readthedocs.io/en/stable/features/bluegreen/#configurable-features) listed on the upstream docs such as `autoPromotionSeconds: 20` for example

Deploy the rollouts-demo application. The following rollout has `autoPromotionEnabled: false` to demonstrate a manual promotion process
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: rollouts-demo
---
apiVersion: v1
kind: Service
metadata:
  name: rollouts-demo-active
  namespace: rollouts-demo
  labels:
    app: rollouts-demo
    service: rollouts-demo
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: rollouts-demo
---
apiVersion: v1
kind: Service
metadata:
  name: rollouts-demo-preview
  namespace: rollouts-demo
  labels:
    app: rollouts-demo
    service: rollouts-demo
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: rollouts-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rollouts-demo
  namespace: rollouts-demo
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollouts-demo
  namespace: rollouts-demo
spec:
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: rollouts-demo
  strategy:
    blueGreen:
      activeService: rollouts-demo-active 
      previewService: rollouts-demo-preview
      autoPromotionEnabled: false
  template:
    metadata:
      labels:
        app: rollouts-demo
        version: stable
    spec:
      containers:
      - name: rollouts-demo
        image: argoproj/rollouts-demo:blue
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        resources:
          requests:
            cpu: 5m
            memory: 32Mi
EOF
```

Check your rollout status:
```
kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo
```

Example output below:
```
% kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo
Name:            rollouts-demo
Namespace:       rollouts-demo
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (stable, active)
Replicas:
  Desired:       1
  Current:       1
  Updated:       1
  Ready:         1
  Available:     1

NAME                                       KIND        STATUS     AGE  INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy  22s  
└──# revision:1                                                        
   └──⧉ rollouts-demo-6ccb95ffd5           ReplicaSet  ✔ Healthy  22s  stable,active
      └──□ rollouts-demo-6ccb95ffd5-g2xtn  Pod         ✔ Running  22s  ready:1/1```
```

Now we can expose the rollout demo UI using Gloo Edge:
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: rollouts-demo-active
  namespace: rollouts-demo
spec:
  kube:
    selector:
      app: rollouts-demo
    serviceName: rollouts-demo-active
    serviceNamespace: rollouts-demo
    servicePort: 8080
---
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: rollouts-demo-preview
  namespace: rollouts-demo
spec:
  kube:
    selector:
      app: rollouts-demo
    serviceName: rollouts-demo-preview
    serviceNamespace: rollouts-demo
    servicePort: 8080
---
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: rollouts-demo
  namespace: gloo-system
spec:
  virtualHost:
    domains:
    - '*'
    routes:
    - matchers:
      - prefix: /
      routeAction:
        single:
          upstream:
            name: rollouts-demo-active
            namespace: rollouts-demo
EOF
```

Access our application:
```
open $(glooctl proxy url)
```

We should see the Argo Rollouts Demo with blue squares loading across the screen
![rollouts-ui-blue](.images/rollouts-ui-blue.png)

Now we can promote the image from the `blue` image tag to the `green` image tag
```
kubectl argo rollouts set image rollouts-demo -n rollouts-demo rollouts-demo=argoproj/rollouts-demo:green
```

Since we have set `autoPromotionEnabled: false` we can see that the rollout status is `Paused` and waiting for manual approval
```
% kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo
Name:            rollouts-demo
Namespace:       rollouts-demo
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (stable, active)
                 argoproj/rollouts-demo:green (preview)
Replicas:
  Desired:       1
  Current:       2
  Updated:       1
  Ready:         1
  Available:     1

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   15m    
├──# revision:4                                                          
│  └──⧉ rollouts-demo-5b8f48f456           ReplicaSet  ✔ Healthy  11m    preview
│     └──□ rollouts-demo-5b8f48f456-hsnc5  Pod         ✔ Running  36s    ready:1/1
└──# revision:3                                                          
   └──⧉ rollouts-demo-6ccb95ffd5           ReplicaSet  ✔ Healthy  15m    stable,active
      └──□ rollouts-demo-6ccb95ffd5-r9wjk  Pod         ✔ Running  9m57s  ready:1/1
```

Run the following command to promote the rollout:
```
kubectl argo rollouts promote rollouts-demo -n rollouts-demo
```

We should quickly see in the UI that traffic was shifted immediately from blue to green
![rollouts-ui-blue-green](.images/rollouts-ui-blue-green.png)

# Bonus:
If you set the Rollout strategy as follows, the promotion will happen automatically 20 seconds after the pod is marked healthy
```
strategy:
  blueGreen:
    activeService: rollouts-demo-active 
    previewService: rollouts-demo-preview
    autoPromotionEnabled: true
    autoPromotionSeconds: 20
```

# Cleanup blue/green rollouts demo
```
kubectl delete vs rollouts-demo -n gloo-system
kubectl delete upstream rollouts-demo-preview -n rollouts-demo
kubectl delete upstream rollouts-demo-active -n rollouts-demo
kubectl delete rollout rollouts-demo -n rollouts-demo
kubectl delete serviceaccount rollouts-demo -n rollouts-demo
kubectl delete service rollouts-demo-preview -n rollouts-demo
kubectl delete service rollouts-demo-active -n rollouts-demo
kubectl delete ns rollouts-demo
```

# Canary Rollout Strategy
A Canary rollout is a deployment strategy where the operator releases a new version of their application to a small percentage of the production traffic. We can use weights, pause durations, and manual promotion in order to control how our application is rolled out across the stable and canary services

First we can deploy the v1 of our rollouts demo which uses the `blue` image tag
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Service
metadata:
  name: rollouts-demo-stable
  namespace: rollouts-demo
  labels:
    app: rollouts-demo
    service: rollouts-demo
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: rollouts-demo
---
apiVersion: v1
kind: Service
metadata:
  name: rollouts-demo-canary
  namespace: rollouts-demo
  labels:
    app: rollouts-demo
    service: rollouts-demo
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: rollouts-demo
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollouts-demo-rollout
  namespace: rollouts-demo
spec:
  selector:
    matchLabels:
      app: rollouts-demo
  template:
    metadata:
      labels:
        app: rollouts-demo
    spec:
      containers:
      - name: rollouts-demo
        image: argoproj/rollouts-demo:blue
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        resources:
          requests:
            cpu: 5m
            memory: 32Mi
  strategy:
    canary:
      canaryService: rollouts-demo-canary
      stableService: rollouts-demo-stable
      trafficRouting:
        plugins:
          solo-io/glooedge:
            routeTable:
              name: rollouts-demo-routes
              namespace: rollouts-demo
      steps:
        - setWeight: 10
        - pause: {duration: 5}
        - setWeight: 25
        - pause: {duration: 5}
        - setWeight: 50
        - pause: {duration: 5}
        - setWeight: 75
        - pause: {duration: 5}
        - setWeight: 100
        - pause: {duration: 5}
EOF
```

Check your rollout status:
```
kubectl argo rollouts get rollout rollouts-demo-rollout -n rollouts-demo
```

Output should look similar to below:
```
% kubectl argo rollouts get rollout rollouts-demo-rollout -n rollouts-demo
Name:            rollouts-demo-rollout
Namespace:       rollouts-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          10/10
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:blue (stable)
Replicas:
  Desired:       1
  Current:       1
  Updated:       1
  Ready:         1
  Available:     1

NAME                                              KIND        STATUS     AGE  INFO
⟳ rollouts-demo-rollout                           Rollout     ✔ Healthy  86s  
└──# revision:1                                                               
   └──⧉ rollouts-demo-rollout-f7c568d5d           ReplicaSet  ✔ Healthy  86s  stable
      └──□ rollouts-demo-rollout-f7c568d5d-dv8fd  Pod         ✔ Running  86s  ready:1/1
```

Now we can expose the rollout demo UI using Gloo Edge:
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: rollouts-demo-stable
  namespace: rollouts-demo 
spec:
  kube:
    selector:
      app: rollouts-demo
    serviceName: rollouts-demo-stable
    serviceNamespace: rollouts-demo
    servicePort: 8080
---
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: rollouts-demo-canary
  namespace: rollouts-demo
spec:
  kube:
    selector:
      app: rollouts-demo
    serviceName: rollouts-demo-canary
    serviceNamespace: rollouts-demo
    servicePort: 8080
---
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: rollouts-demo
  namespace: gloo-system
spec:
  virtualHost:
    domains:
    - '*'
    routes:
    - matchers:
      - prefix: /
      delegateAction:
        ref:
          name: rollouts-demo-routes
          namespace: rollouts-demo
---
apiVersion: gateway.solo.io/v1
kind: RouteTable
metadata:
  name: rollouts-demo-routes
  namespace: rollouts-demo
spec:
  routes:
    - matchers:
      - prefix: /
      routeAction:
        multi:
          destinations:
          - destination:
              upstream:
                name: rollouts-demo-stable
                namespace: rollouts-demo
            weight: 100
EOF
```

Now lets promote the image from the `blue` image tag to the `green` image tag and watch the traffic pattern in the UI
```
kubectl argo rollouts set image rollouts-demo -n rollouts-demo-rollout rollouts-demo=argoproj/rollouts-demo:green
```

This time, instead of immediately switching from blue to green, we will see a more gradual shift in traffic as defined in our rollout strategy
```
steps:
- setWeight: 10
- pause: {duration: 5}
- setWeight: 25
- pause: {duration: 5}
- setWeight: 50
- pause: {duration: 5}
- setWeight: 75
- pause: {duration: 5}
- setWeight: 100
- pause: {duration: 5}
```

In the rollouts demo UI we should be able to see the steps and the respective weights pretty clearly in the shift from blue to green.
![rollouts-ui-canary](.images/rollouts-ui-canary.png)

# Bonus:
If you set the Rollout strategy steps as follows, the promotion will happen automatically every 5 seconds except for the second step (at 25%) which requires a manual promotion
```
steps:
- setWeight: 10
- pause: {duration: 5}
- setWeight: 25
- pause: {}
- setWeight: 50
- pause: {duration: 5}
- setWeight: 75
- pause: {duration: 5}
- setWeight: 100
- pause: {duration: 5}
```

Run the following command to promote the rollout:
```
kubectl argo rollouts promote rollouts-demo-rollout -n rollouts-demo
```

In the rollouts demo UI we should be able to see that the rollout pauses at the 25% weight until promoted, with the rest of the rollout automatically promoted after a 5 second duration between steps.
![rollouts-ui-canary-with-man](.images/rollouts-ui-canary-with-man.png)

# Conclusion
This tutorial demonstrates the ease of integrating progressive delivery workflows to your application deployments using Argo Rollouts and Gloo Edge but only scratches the surface! Many standard options exist in the Argo Rollouts Documentation such as [BlueGreen Deployment Strategy](https://argo-rollouts.readthedocs.io/en/stable/features/bluegreen/) and [Canary Strategy](https://argo-rollouts.readthedocs.io/en/stable/features/canary/). Take a look at other strategies and routing examples in the plugin [github repo examples](https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-glooedge/tree/main/examples) for more examples to get you started!