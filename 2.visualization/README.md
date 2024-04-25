## Setup the workshop environment

Create the working directory and the Docker Compose file

```bash
mkdir ~/workshop-monitoring && cd $_
touch docker-compose.yaml \
      Dockerfile \
      server.py \
      client.py \
      requirements.txt \
      prometheus.yaml
```

Add content to the `docker-compose.yaml` file:

```yaml
version: '3.8'

services:

  my_app:
    ports:
      - 8080:8080
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./server.py:/var/app/server.py
    command:
    - server.py

  client:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./client.py:/var/app/client.py
    command:
    - client.py

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    command:
      - '--path.rootfs=/host'
    pid: host
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'
  
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
```

Add the following directives to the `Dockerfile` file:

```Dockerfile
FROM python:3
WORKDIR /var/app
COPY requirements.txt /var/app
RUN pip install -r requirements.txt
EXPOSE 8080
ENTRYPOINT [ "python" ]
```

Add the following content to the `requirements.txt` file:

```txt
Flask==3.0.2
Requests==2.31.0
prometheus-flask-exporter==0.23.0
```

Add the following code to the `server.py` file:

```py
from flask import Flask, Response, request
from random import random, seed, randint
from time import time, sleep
from prometheus_flask_exporter import PrometheusMetrics


app = Flask(__name__)
metrics = PrometheusMetrics(app)

metrics.info('app_info', 'Application info', version='1.0.3')


@app.route('/login', methods=['POST'])
def login():
    response_body = {
        'user_id': 'e6b9540f-8866-4173-8700-b55fcec1f6cb',
        'token': 'xxxxxx'
    }
    return(response_body)

@app.route('/purchase', methods=['POST'])
def purchase():
    rand_option = randint(0, 10)
    if rand_option > 2:
        response_body = {
            'purchased_items': [
                {'id': 'f5702b0f-37ce-410c-9ffd-77ace989f955'}
            ]
        }
        response_code = 200
    else:
        response_body = {'message': 'Your credit card was not accepted'}
        response_code = 402
    
    return response_body, response_code

@app.route('/user/profile', methods=['GET'])
def user_profile():
    response_body = {
        'name': 'John',
        'age': 40,
    }
    return(response_body)

@app.route('/', methods=['GET'])
def home():
    return('I am running!')


if __name__ == '__main__':
    app.run(host='0.0.0.0', port='8080')
```

Add the following code to the `client.py` file:

```py
import requests
from time import sleep
from random import random, seed, randint
import json

seed(1)
url = 'http://my_app:8080'

def pythonrequests():
    try:
        rand_option = randint(0, 2)
        if rand_option == 0:
            r = requests.post('{}/login'.format(url))
        elif rand_option == 1:
            r = requests.post('{}/purchase'.format(url))
        elif rand_option == 2:
            r = requests.get('{}/user/profile'.format(url))
        log_dict = {
            'URL': str(r.url),
            'status': str(r.status_code),
            'content': str(r.content),
        }
        print(json.dumps(log_dict, indent=2, separators=(',', ':')))
    except requests.exceptions.RequestException as err:
        log_dict = {
            'error': str(err),   
        }
        print(json.dumps(log_dict, indent=2, separators=(',', ':')))

while True:
    pythonrequests()
    y = random()
    sleep(round(y,1))
```

Finally, add the content of the prometheus configuration file `prometheus.yaml`

```yaml
# Global configuration
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "node"
    static_configs:
      - targets: ["node_exporter:9100"]
  - job_name: "my_app"
    static_configs:
      - targets:
          - "my_app:8080"
```

Build the container image and run the containers. Then you can check the logs of each container, the server and the client

```bash
# Build the docker image
docker-compose build

# Start up the containers in detached mode
docker-compose up -d

# Verify that the containers are up and running
docker-compose ps 

# Check the logs of the application
docker-compose logs my_app

# Check the logs of the client
docker-compose logs client
```

## Include the visualization tool (Grafana)

Modify the `docker-compose.yaml` file and add the grafana service

```
  grafana:
    image: grafana/grafana
    ports:
      - 3000:3000
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=helloworld123
```

Then run the `docker-compose` command to start up the service.

```bash
docker-compose up grafana -d
```

Open http://localhost:3000

Login with the following credentials:

- username: `admin`
- password: `helloworld123`

## Add a Data Source manually

Let's use Prometheus as our Data Source

Open Connections > Data Sources > + Add new data source > Choose Prometheus

Fill in:
- Name: prometheusds
- Prometheus server URL: http://prometheus:9090


## Create a new dashboard

Use the following promql queries to create a new dashboard

### Duration

Duration per path

```promql
rate(flask_http_request_duration_seconds_sum{status="200"}[30s]) / rate(flask_http_request_duration_seconds_count{status="200"}[30s])
```

### Requests per second

Average throughput in the last 30 seconds (requests per second)

```promql
sum by(status) (rate(flask_http_request_total[30s]))
```

### Memory usage

Memory used by the application

```promql
process_resident_memory_bytes{job="my_app"}
```

Total memory available in the node

```promql
node_memory_MemTotal_bytes
```

### Application version

Application information

```promql
app_info
```

## Import an existing dashboard

There is an existing dashboard to visualize the most common metrics exposed by the node exporter https://grafana.com/grafana/dashboards/1860-node-exporter-full/

Dashboards > New > Import

Fill in the ID of the dashboard `1860`, then press Load.

Select the Prometheus data source and click on Import.

## Use Grafana configuration files

Create the following directories and files

```
.
├── provisioning
│   ├── dashboards
│   │   ├── dashboards.yaml
│   │   ├── my_app.json
│   │   └── node_exporter.json
│   └── datasources
│       └── datasources.yaml
```

Click on Dashboards > Application my_app > Dashboard Settings > JSON Model

Copy the JSON Model and save it as `provisioning/dashboards/my_app.json`
Save the node exporter dashboard as well as `provisioning/dashboards/node_exporter.json`

Place the following content in `provisioning/datasources/datasources.yaml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
```

Place the following content in `provisioning/dashboards/dashboards.yaml`

```yaml
apiVersion: 1
providers:
  - name: "Dashboard provider"
    orgId: 1
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

Update the grafana service in `docker-compose.yaml` by specifying the volumes

```diff
grafana:
    image: grafana/grafana
    ports:
      - 3000:3000
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=helloworld123
+   volumes:
+     - ./provisioning:/etc/grafana/provisioning
```

Recreate the grafana container

```bash
docker-compose down grafana
docker-compose up grafana -d
```

Open the grafana dashboard and provide the login credentials again

http://localhost:3000

Verify that the Prometheus datasource has been specified.
Verify that the Application my_app is working again.
Unfortunatelly it is not :(

## Fix the dashboard datasource

Navigate to:

Dashboards > Application my_app > Dashboard settings > Variables > + New Variable

Following the fields:

**General**:

- Select variable type: Data Source
- Name: promdb

**Data source options**

- Type: Prometheus

Click on Apply and then Save dashboard

Copy the resultant json and replace the file `provisioning/dashboards/my_app.json`.
Replace the prometheus hardcoded source ID with the new variable `$promdb` in all places where the datasource is specified:

```json
"datasource": {
  "type": "prometheus",
  "uid": "${promdb}"
},
```

To validate if the dashboard has been fixed, navigate to:

Dashboards > Application my_app

You should be able to select a Data Source and the visualizations should work again.

## Challenge

As you can see, to show the current version of the application, the table is not the best visualization for that particular information.
Browse the page https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/ and choose the best visualization to show the version of the application.
Then, create a new Panel with the visualization you chose.