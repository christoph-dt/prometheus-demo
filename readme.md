# Prometheus Demo 

Creating a demo environment to monitor metric ingestions from Prometheus to Dynatrace. With this example, you will be able to deploy an NGINX pod which will be monitored by a Prometheus exporter and the metrics will be ingested into your Dynatrace environment.

## Prerequisites
- Kubernetes Cluster
- Dynatrace [Kubernetes API Monitoring](https://docs.dynatrace.com/docs/setup-and-configuration/setup-on-k8s/guides/operation/k8s-api-monitoring) enabled
- Dynatrace, go to your Kubernetes cluster settings page and enable
  - Monitor Kubernetes namespaces, services, workloads, and pods
  - Monitor annotated Prometheus exporters

## Prometheus Concept
Prometheus is using exporters that provide metrics to an API endpoint. This metrics can be scraped via the Dynatrace Kubernetes API Integration and provided in the Data Explorer. 

## Deploy Nginx Server with Prometheus Exporter
To deploy all our resources needed the namespace "prometheus-demo" will be used throughout this demo. The nginx server will expose its website on Port 80 and NodePort 30007 and the prometheus metrics on port 9113 and NodePort 30008.

In order to create the namespace the following command can be used.
`kubectl create namespace prometheus-demo`

In the following steps the NGINX Server deployment file will be created step by step:

### Create NGINX ConfigMap

Before deploying the NGINX server we will need to provide a configuration to expose the metrics from the server. The configuration needs to be stored at '/etc/nginx/conf.d/nginx.conf' within the nginx pod.

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


To provide the configuration a ConfigMap name "nginx-conf" will be created. It will store the configuration mentioned above in nginx.conf. After mounting this configMap in the Pod the file nginx.conf will be available as read-only file in the mounted path.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: prometheus-demo
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

### Create NGINX Pod yaml

To beginn a standard deployment file will be created and stored locally To create the base file structure you can use the following command:
 ``kubectl create deployment nginx-server --image=nginx -n prometheus-demo --replicas=1 --port=80 --dry-run=client -o yaml > nginx.yaml``
Here is the output that is stored in the nginx.yaml file:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-server
  name: nginx-server
  namespace: prometheus-demo
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

*We will now adapt this file and add additional configurations step by step.*

### Mount ConfigMap
As a next step thethe NGINX Config from the ConfigMap above needs to be mounted: 

add the following lines in `.spec.template.spec.containers[nginx]`
```yml
            volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/
```
add the following lines in`.spec.template.spec`
```yml        
      volumes:
      - name: nginx-config
        configMap: 
          name: nginx-conf
```
### Add Prometheus Exporter
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
### Add Dynatrace Annotations

As a last step the [dynatrace annotations](https://docs.dynatrace.com/docs/shortlink/monitor-prometheus-metrics#annotate-prometheus-exporter-pods) need to be added. For more details check out the [Dynatrace Documentation](https://docs.dynatrace.com/docs/shortlink/monitor-prometheus-metrics)

In this configuration the metrics are provided at http://localhost:9113/metrics and a filter to deny metrics that contain "build".

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
Below you can find the entire file combined. Now you can apply this yaml file with `kubectl apply -f nginx.yaml` to your cluster to deploy the nginx server
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  namespace: prometheus-demo
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

### [Optional] Expose Website and Prometheus metrics



## Solution

You can find all yaml files described above in the subfolder `solution`.  
