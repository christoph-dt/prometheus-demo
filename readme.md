# Prometheus Demo 

Creating a demo environment to monitor metric ingestions from Prometheus to Dynatrace. In this example we will create a Nginx Pod which will be monitored

## Prerequisites
- Kubernetes Cluster
- Dynatrace [Kubernetes API Monitoring](https://docs.dynatrace.com/docs/setup-and-configuration/setup-on-k8s/guides/operation/k8s-api-monitoring) enabled

## Prometheus Concept
Prometheus is using exporters that provide metrics to an API endpoint. This metrics can be scraped via the Dynatrace Kubernetes API Integration and provided in the Data Explorer. 

## Deploy Nginx Server with Prometheus Exporter
In this demo we will create a nginx server in the namespace 'nginx' which will expose its website on Port 80 and NodePort 30007 and the prometheus metrics on port 9113 and NodePort 30008.

Before deploying the NGINX server we will need to provide a configuration to expose the metrics from the server. The configuration needs to be stored at '/etc/nginx/conf.d/nginx.conf' within the pod.

'''
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
'''

### Create NGINX Config 
To provide the configuration we will create ConfigMap name "nginx-conf"

'''
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
'''

### Create NGINX Pod Config
In the following steps we will create the deployment file step by step for the NGINX Server deployment:

The following configuration 
'''
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
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
'''