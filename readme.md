# Prometheus Demo 

Creating a demo environment to monitor metric ingestions from Prometheus to Dynatrace. In this example we will create a Nginx Pod which will be monitored

## Prerequisites
- Kubernetes Cluster
- Dynatrace [Kubernetes API Monitoring](https://docs.dynatrace.com/docs/setup-and-configuration/setup-on-k8s/guides/operation/k8s-api-monitoring) enabled
- Dynatrace, go to your Kubernetes cluster settings page and enable
  - Monitor Kubernetes namespaces, services, workloads, and pods
  - Monitor annotated Prometheus exporters

## Prometheus Concept
Prometheus is using exporters that provide metrics to an API endpoint. This metrics can be scraped via the Dynatrace Kubernetes API Integration and provided in the Data Explorer. 

## Deploy Nginx Server with Prometheus Exporter
In this demo we will create a nginx server in the namespace 'nginx' which will expose its website on Port 80 and NodePort 30007 and the prometheus metrics on port 9113 and NodePort 30008.

Before deploying the NGINX server we will need to provide a configuration to expose the metrics from the server. The configuration needs to be stored at '/etc/nginx/conf.d/nginx.conf' within the pod.

```
server {
      listen       80;
      server_name  localhost;

      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }
      location /metrics {
          stub_status;
      }
    }  
```

### Create NGINX Config 
To provide the configuration we will create ConfigMap name "nginx-conf"

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: nginx
data:
  nginx.conf: |
    server {
      listen       80;
      server_name  localhost;

      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }
      location /metrics {
          stub_status;
      }
    }
EOF
```

### Create NGINX Pod Config
In the following steps we will create the deployment file step by step for the NGINX Server deployment:

We begin with a standard deployment file. To create the structure for the file you can use ``kubectl create deployment nginx-server --image=nginx --replicas=1 --port=80 --dry-run=client -o yaml > nginx.yaml``

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-server
  name: nginx-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-server
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-server
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

As a next step we will mount the NGINX Config from the ConfigMap we created above: 
```yml
            volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/
        
      volumes:
      - name: nginx-config
        configMap: 
          name: nginx-conf
```

Now the metrics are exposed within the Pod an can be used from prometheus exporter at `http://localhost:80/metrics`. Therefore a NGINX Exporter sidecar container can be added in the `.spec.template.spec.containers` section:

```yml
        - name: nginx-exporter
          image: 'nginx/nginx-prometheus-exporter'
          args:
            - '-nginx.scrape-uri=http://localhost:80/metrics'
        resources:
            limits:
              memory: 128Mi
              cpu: 500m
          ports:
            - containerPort: 9113
```
as a last step the [dynatrace annotations](https://docs.dynatrace.com/docs/shortlink/monitor-prometheus-metrics#annotate-prometheus-exporter-pods) need to be added. For more details check out the [Dynatrace documentation](https://docs.dynatrace.com/docs/shortlink/monitor-prometheus-metrics)

```yml
    annotations:
        metrics.dynatrace.com/port: '9113'
        metrics.dynatrace.com/scrape: 'true'
        metrics.dynatrace.com/secure: 'false'
        metrics.dynatrace.com/path: '/metrics'
        metrics.dynatrace.com/filter: | 
          {
          "mode":"exclude",
          "names":["*build*"]
          }
```
Below you can find the entire file combined. Now you can apply this yaml file to your cluster to deploy the nginx server
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  namespace: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        metrics.dynatrace.com/port: '9113'
        metrics.dynatrace.com/scrape: 'true'
        metrics.dynatrace.com/secure: 'false'
        metrics.dynatrace.com/path: '/metrics'
        metrics.dynatrace.com/filter: | 
          {
          "mode":"exclude",
          "names":["*build*"]
          }

    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/
        - name: nginx-exporter
          image: 'nginx/nginx-prometheus-exporter'
          args:
            - '-nginx.scrape-uri=http://localhost:80/metrics'
          resources:
            limits:
              memory: 128Mi
              cpu: 500m
          ports:
            - containerPort: 9113
      volumes:
      - name: nginx-config
        configMap: 
          name: nginx-conf
```

DONE! Now the Prometheus metrics should appear in the Data Explorer within your Dynatrace Environment.