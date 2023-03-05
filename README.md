# Microservices Deployment Pattern

## Overview

This shows a sample deployment template of how two microapps would be deployed to Kubernetes cluster:

- User
- Shift

> User application runs code that runs a small API that returns users from a database.

> Shift application runs code that runs a small API that returns shifts from a database.

## Template structure

```bash
├── k8s-configs
│   ├── rbac
│   │   ├── developer.yaml
│   │   └── role-binding.yaml
│   ├── shift-app
│   │   └── shift-deployment.yaml
│   └── user-app
│       ├── user-deployment.yaml
│       └── user-hpa.yaml
└── README.md
```

## Auto-scaling

This config makes sure that the User application has daily bell-curve scaling. So auto-scaling is set to kick in when the CPU reaches 70%. It is done by adding `HorizontalPodAutoscaler` concept into the [groove](./k8s-configs/user-app/user-hpa.yaml) as per Kubernetes Horizontal Pod Autoscaler (HPA). 

```yaml
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

Another tweak also was done to the deployment.yaml to add the below resources:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 100Mi
  limits:
    cpu: 500m
    memory: 200Mi
```

## Rolling updates and Auto-rollbacks

### Rolling Updates

To enable rolling updates, we have specified the strategy field in the [deployment yaml](/k8s-configs/user-app/user-deployment.yaml). We have set the type to RollingUpdate and specified the maximum number of pods that can be unavailable during the update as below:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
```

### Auto-rollbacks

We have set the `progressDeadlineSeconds` field in the [deployment yaml](/k8s-configs/user-app/user-deployment.yaml) to specify the maximum amount of time that the deployment can make progress during the update. If the deployment fails to make progress within this time, k8s will automatically rollback the update addressing this need.

```yaml
spec:
  progressDeadlineSeconds: 600
```

## Control over developer-interuptions on deployed pods

To restrict the development team from running certain commands on the Kubernetes cluster, we have used Kubernetes Role-Based Access Control (RBAC) to set up access controls. We have [Kubernetes Role](/k8s-configs/rbac/developer.yaml) in place that grants the development team the ability to deploy and rollback deployments, but restricts their access to certain resources or commands. This would ensure that developers are not having tied hands, but have enough access on their deployed pods/images within the target cluster/namespace such as to create, get, watch, update, patch, delete, and rollback Deployments, but only allows them to get, watch, and list Pods and Services. This also restricts their access to other resources, such as ConfigMaps or Secrets, especially handy when application is developed under a multi-vendor perspective with added security on the infrastructure itself.

There is also an associated binding rule that is available [here](/k8s-configs/rbac/role-binding.yaml).

## Support for multi-environment deployment [Bonus]

To apply the same Kubernetes resources to multiple environments, such as staging and production, we can use Kubernetes Namespaces. Namespaces provide a way to create a virtual cluster within a physical cluster, allowing you to logically separate different environments, applications, or teams from each other.

We can add `namespace` attribute under `metadata` section of the deploy configuration as:

```yaml
namespace: staging
```

Another approach is to use Helm Charts with jinja2 templatization. That way we can templatize a lot of the variables like namespace, replicas, resources (CPU, Memory) based on the environment it is being deployed to from a CI/CD perspective.
We can have variables like *target_namespace, total_replicas, cpu_req, mem_req, cpu_limit, mem_limit*, etc...

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-deployment
  namespace: {{ target_namespace }}
spec:
  replicas: {{ total_replicas }}
  selector:
    matchLabels:
      app: user
  strategy:
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
  type: RollingUpdate
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
      - name: user
        image: php:7.4-apache
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: {{ cpu_req }}
            memory: {{ mem_req }}
          limits:
            cpu: {{ cpu_limit }}
            memory: {{ mem_limit }}
        livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          imagePullSecrets:
          - name: my-registry-secret
        terminationGracePeriodSeconds: 30
        progressDeadlineSeconds: 600
      restartPolicy: Always
```

Then during run-time, use jinja2 cli to convert the yaml replacing all variable values.

```bash
j2 user-deployment.yaml.j2 -o user-deployment.yaml
```

> You can read more about **Jinja2** templatization [here](https://jinja.palletsprojects.com/en/3.1.x/templates/#synopsis).

## Auto-scaling based on custom Network Latency [Bonus]

To achieve this we can create a custom metric adapter that provides network latency metrics to Kubernetes. This can be done using tools like Prometheus or Grafana, which can be configured to scrape metrics from the containers. Once we have that metric in place, then the application needs to have HPA that scales based on Latency:

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: user-hpa-latency
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: network-latency
        selector:
          matchLabels:
            app: user-deployment
      target:
        type: AverageValue
        averageValue: 100ms
```
