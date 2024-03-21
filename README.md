## Setup the workshop environment

Create the working directory and the Docker Compose file

```bash
mkdir ~/workshop-monitoring && cd $_
touch docker-compose.yaml
cat <<EOF >> docker-compose.yaml
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
EOF
```

Create the Docker file with the python image that installs the required pip packages

```bash
cat <<EOF >> Dockerfile
FROM python:3
WORKDIR /var/app
COPY requirements.txt /var/app
RUN pip install -r requirements.txt
EXPOSE 8080
ENTRYPOINT [ "python" ]
EOF
```

Create the requirements file to install the required pip packages

```bash
cat <<EOF >> requirements.txt
Flask==3.0.2
Requests==2.31.0
EOF
```

Create the python code to run a flask server

```bash
cat <<EOF >> server.py
from flask import Flask, Response
from random import random, seed, randint

seed(1)
app = Flask(__name__)


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

EOF
```

Create the python script that performs http requests to the flask server.

```bash
cat <<EOF >> client.py
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
    sleep(round(y,2))
EOF
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

Open http://localhost:8080/ in the browser to check it yourself.

## Introduce an unintended side effect in the application

In the `server.py` file replace the condition in the purchase function with:

```diff
def purchase():
    rand_option = randint(0, 10)
-   if rand_option > 2:
+   if rand_option < 1:
```

Restart the server to use the new source code... I mean, let's perform a deployment.

```bash
docker-compose restart my_app
```

Open http://localhost:8080/ in the browser to check it yourself.

Looks like the application is working ok, no?

It turns out clients start to complain. They are not able to purchase products!

## Time to instrument our application for monitoring

Add the Prometheus Client library to the `requirements.txt` file

```diff
Flask==3.0.2
Requests==2.31.0
+prometheus-client==0.20.0
```

Rebuild the container image to install the new package

```bash
docker-compose build
```

Add the import statement to use the prometheus client library in the `server.py` file

```diff
from flask import Flask, Response
from random import random, seed, randint
+from prometheus_client import Counter, generate_latest
```

Then, define the necessary Prometheus metrics

```diff
seed(1)
app = Flask(__name__)

+successful_purchases_counter = Counter('successful_purchases_total', 'Total number of successful purchases')
+failed_purchases_counter = Counter('failed_purchases_total', 'Total number of failed purchases')
```

Modify the `purchase()` function to increase the counter when a purchase is successful and failed

```diff
def purchase():
    rand_option = randint(1, 10)
    if rand_option < 1:
+       successful_purchases_counter.inc()
        response_body = {
            'purchased_items': [
                {'id': 'f5702b0f-37ce-410c-9ffd-77ace989f955'}
            ]
        }
        response_code = 200
    else:
+       failed_purchases_counter.inc()
        response_body = {'message': 'Your credit card was not accepted'}
        response_code = 402
    return response_body, response_code
```

Finally, add the metrics endpoint to the script

```diff
@app.route('/', methods=['GET'])
def home():
    return('I am running!')


+@app.route('/metrics')
+def metrics():
+    # Expose Prometheus metrics
+    return Response(generate_latest(), mimetype='text/plain')
```

Restart the application to make use of new version of the code

```bash
docker-compose up my_app -d
```

Open the URL http://localhost:8080/metrics to check the metrics. You can reload it every now and then to check how the metrics are updated.

## Run the Prometheus server

Add the following services to the `docker-compose.yaml` file

```yaml
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
```

Create the prometheus configuration file

```bash
cat <<EOF >> prometheus.yaml
# Global configuration
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093


# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "my_app"
    static_configs:
      - targets:
          - "my_app:8080"
EOF
```

Run the Prometheus and Grafana servers

```bash
docker-compose up prometheus -d
```

Open the prometheus dashboard http://localhost:9090/

You can run some simple queries to counters:

```promql
# Get the latest sample
successful_purchases_total

# Get the samples collected in the last 5 minutes. The result is a range vector
successful_purchases_total[5m]

# The following example expression returns the per-second rate of successful purchases as measured over the last 5 minutes, per time series in the range vector
rate(successful_purchases_total[5m])

# Calculate the percentage of failed purchases
rate(failed_purchases_total[5m]) / (rate(successful_purchases_total[5m]) + rate(failed_purchases_total[5m])) * 100
```

Let's check how summary metrics look.

We already scrape metrics from the prometheus server itself.

```promql
# Summary metrics

prometheus_target_interval_length_seconds # summary
prometheus_target_interval_length_seconds_count # count
prometheus_target_interval_length_seconds_sum # count

# Average latency over the last minute
rate(prometheus_target_interval_length_seconds_sum[1m]) / rate(prometheus_target_interval_length_seconds_count[1m])
```

## Same metrics from multiple instances

Update the Docker Compose file to run two replicas of the python application:

```diff
  my_app:
    ports:
-     - 8080:8080
+     - 8080
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./server.py:/var/app/server.py
    command:
    - server.py
+   deploy:
+     replicas: 2
```

Then update the Prometheus configuration file to scrape the metrics

```diff
  - job_name: "my_app"
    static_configs:
      - targets:
# We remove the hostname my_app because it will balance the traffic
# We need to get the metrics directly from each instance
-         - "my_app:8080"
+         - "workshop-monitoring-my_app-1:8080"
+         - "workshop-monitoring-my_app-2:8080"
```

Restart the prometheus server and recreate the application containers

```bash
docker-compose up my_app -d
docker-compose restart prometheus
```

Open again the prometheus dashboard http://localhost:9090/

Let's try some aggregations:

```promql
# The metric failed_purchases_total now has two time-series, one for each instance
failed_purchases_total

# The collected samples in the last five minutes, the result is a range-vector
failed_purchases_total[5m]

# Remember that counters almost always should come with the rate() function
rate(failed_purchases_total[5m])

# Now we can aggregate them.
# We can use the sum() function. It will produce a new metric. All labels will be gone
rate(failed_purchases_total[5m])

# We can keep the job label
sum by (job) (rate(failed_purchases_total[5m]))
```

## Add a new metric to measure latency

Update the following import statement

```diff
-from flask import Flask, Response
+from flask import Flask, Response, request
from random import random, seed, randint
-from prometheus_client import Counter, generate_latest
+from prometheus_client import Counter, Histogram, generate_latest
+from time import time, sleep
```

Add the new metric definition and a new function to add the obsevation

```diff
successful_purchases_counter = Counter('successful_purchases_total', 'Total number of successful purchases')
failed_purchases_counter = Counter('failed_purchases_total', 'Total number of failed purchases')
+request_time = Histogram('purchase_processing_seconds', 'Time spent processing request', ['method', 'endpoint'])

+def record_request_data(method, endpoint, start_time):
+    y = random()
+    sleep(round(y,2))
+    duration = time() - start_time
+    request_time.labels(method, endpoint).observe(duration)
```

Update the purchase function

```diff
def purchase():
+   start_time = time()
    rand_option = randint(1, 10)
    if rand_option > 1:
        successful_purchases_counter.inc()
        response_body = {
            'purchased_items': [
                {'id': 'f5702b0f-37ce-410c-9ffd-77ace989f955'}
            ]
        }
        response_code = 200
    else:
        failed_purchases_counter.inc()
        response_body = {'message': 'Your credit card was not accepted'}
        response_code = 402
    
+   method = request.method
+   endpoint = request.path
+   record_request_data(method, endpoint, start_time)
    return response_body, response_code
```

Restart the application instances to produce the new metric

```bash
docker-compose restart my_app
```

Open again the Prometheus dashboard and try queries with the new metric

```promql
# All quantiles for purchase_processing_seconds_bucket
purchase_processing_seconds_bucket

# Aggregate them by job and le labels
sum by (job, le) (rate(purchase_processing_seconds_bucket[5m]))

# Show the quantiles
histogram_quantile(0.95, sum by (job, le) (rate(purchase_processing_seconds_bucket[5m])))
```


## Challenge

1. Run the node exporter container and modify the prometheus configuration to add a new job with the new target. You can make use of the Docker Compose definition explained here https://github.com/prometheus/node_exporter?tab=readme-ov-file#docker Note: remember that the name of the service will be the name of the hostname.
1. Execute a query to get the cpu usage.