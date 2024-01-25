
# Week 2 — Distributed Tracing

## To view the frontend 

```
localhost:3000
```

## To view the backend

```
localhost:4567/api/activities/home
```

### 1. Setting up Honeycomb

Create a free tier Honeycomb account using here [Honeycomb.io](https://ui.honeycomb.io/) 

To set our Honeycomb API key as an environment variable 

```bash
# export the key to the terminal
export HONEYCOMB_API_KEY="..."

# confirm if it was exported 
env | grep HONEY

```

### 2. Instrument backend flask to use OpenTelemetry (OTEL) with Honeycomb.io as the provider 

Honeycomb typically refers to a cloud-based observability platform. Honeycomb is an all-in-one solution for distributed tracing, metrics, and logging that provides deep visibility into the performance and behavior of complex systems and applications. It uses `OTEL` libraries which is simply the `OpenTelemetry` libraries.

Honey comb shows you your traces and let's you aggregate over them.

 Read more [here](https://www.honeycomb.io/)

Next we will add the honeycomb environment variables for the `backend-flask` in the `docker-compose.yml`

```yaml
OTEL_SERVICE_NAME: 'backend-flask'
OTEL_EXPORTER_OTLP_ENDPOINT: "https://api.honeycomb.io"
OTEL_EXPORTER_OTLP_HEADERS: "x-honeycomb-team=${HONEYCOMB_API_KEY}"
```

Next we will add the required `OTEL` libraries to our `requirements.txt` file which is located in the `backend-flask/` directory

```txt
opentelemetry-api 
opentelemetry-sdk 
opentelemetry-exporter-otlp-proto-http 
opentelemetry-instrumentation-flask 
opentelemetry-instrumentation-requests # this should instrument outgoing HTTP calls
```

Install the dependencies using the command 

```bash
pip install -r requirements.txt
```

Next we will Initialise honeycomb by adding the following lines of code to `app.py` file

```python
# Honeycomb
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Initialize tracing and an exporter that can send data to Honeycomb
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

# Initialize automatic instrumentation with Flask
app = Flask(__name__) # if this link already exists, DON'T call it again
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```


Then we will build our application using `docker compose up`

The Dataset in honeycomb is automatically created when it arrives with the given service name. In our case the service name is `backend-flask`


Then you have to check honeycomb to see if the data has been imported. If not you can follow the steps below to troubleshoot.


- Add the following lines of code in app.py when initialising the tracing

```python
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor


simple_processor = SimpleSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(simple_processor)
```

Here we are telling the provider to send the spans to the through the simple processor which send the spans over the console compared to the OLTP exporter that sends spans over the internet.

This way the logs of the service will also receive the spans in stdout.  

Next you can view the docker logs by right clicking on the container in vscode and clicking ov `view logs`.

- Check again if the API KEY is set using `env | grep HONEY` on your terminal.

- You can also exec into the docker container and check `env` to see if the container picked up the environment variable. This is a probable reason why you might be getting a `401` error because the docker container didnt pick up the API KEY environment variable.

![Single span](<assets/Screenshot 2024-01-24 at 11.12.56 PM.png>)
in the picture above you can see that we have just one trace with a single span which is the root span(a span is a single piece of instrumentation from a single location in your code, multiple spans make up a single trace).

Read more about datasets, Traces and spans [here](https://www.honeycomb.io/blog/datasets-traces-spans)

### Hardcoding a Span

We are going to harcode a span. This will use just the opentelemetry api just to add instrumentation that specific piece of code.

Go to services/home_activities.py and add in the following

```python
from opentelemetry import trace 
tracer = trace.get_tracer("home_activities")

# inside def run()
with tracer.start_as_current_span("home_activites-mock-data"):
# indent remaining line of code under 
```

`docker compose up -d` again then reload `localhost:4567/api/activities/home`

On the Honeycomb dashboard, go to Home -> Recent traces -> click on the icon of the most recent one.

![mock-data-span](<assets/Screenshot 2024-01-25 at 12.29.49 AM.png>)

### Adding attributes to spans

This is really useful because with this the spans can become replacements for logs.

```python
span = trace.get_current_span()
span.set_attribute("app.now", now.isoformat())

#before the return statement
span.set_attribute("app.result_length", len(results))

```

Now we have two attributes that are meaningful to us.
On Honeycomb go to the recent trace, in the field section search for `app.` and you'll see the two attributes. 

## OBSERVABILITY IN AWS

USING XRAY

![XRAY Architecture](<assets/Screenshot 2024-01-25 at 12.58.45 PM.png>)

The X-Ray daemon is a container that runs alongside your application, it collects the data, batches it and sends it over to the X-ray api so that you can visualise your data. 

AWS X-Ray supports using OpenTelemetry Python and the AWS Distro for OpenTelemetry(ADOT) Collector to instrument the application and send trace data to X-Ray

Step 1: Install the AWS SDK 

Add `aws-xray-sdk` to the requirements.txt file, cd into the backend-flask directory and `pip install -r requirements.txt`


The importance of middleware for tracing is so that for any single request that goes through, we can extract






















# Week 2 — Distributed Tracing

## X-Ray

### Instrument AWS X-Ray for Flask


```sh
export AWS_REGION="ca-central-1"
gp env AWS_REGION="ca-central-1"
```

Add to the `requirements.txt`

```py
aws-xray-sdk
```

Install pythonpendencies

```sh
pip install -r requirements.txt
```

Add to `app.py`

```py
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

xray_url = os.getenv("AWS_XRAY_URL")
xray_recorder.configure(service='Cruddur', dynamic_naming=xray_url)
XRayMiddleware(app, xray_recorder)
```

### Setup AWS X-Ray Resources

Add `aws/json/xray.json`

```json
{
  "SamplingRule": {
      "RuleName": "Cruddur",
      "ResourceARN": "*",
      "Priority": 9000,
      "FixedRate": 0.1,
      "ReservoirSize": 5,
      "ServiceName": "Cruddur",
      "ServiceType": "*",
      "Host": "*",
      "HTTPMethod": "*",
      "URLPath": "*",
      "Version": 1
  }
}
```

```sh
FLASK_ADDRESS="https://4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}"
aws xray create-group \
   --group-name "Cruddur" \
   --filter-expression "service(\"$FLASK_ADDRESS\") {fault OR error}"
```

```sh
aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json
```

 [Install X-ray Daemon](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon.html)

[Github aws-xray-daemon](https://github.com/aws/aws-xray-daemon)
[X-Ray Docker Compose example](https://github.com/marjamis/xray/blob/master/docker-compose.yml)


```sh
 wget https://s3.us-east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-3.x.deb
 sudo dpkg -i **.deb
 ```

### Add Deamon Service to Docker Compose

```yml
  xray-daemon:
    image: "amazon/aws-xray-daemon"
    environment:
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
      AWS_REGION: "us-east-1"
    command:
      - "xray -o -b xray-daemon:2000"
    ports:
      - 2000:2000/udp
```

We need to add these two env vars to our backend-flask in our `docker-compose.yml` file

```yml
      AWS_XRAY_URL: "*4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}*"
      AWS_XRAY_DAEMON_ADDRESS: "xray-daemon:2000"
```

### Check service data for last 10 minutes

```sh
EPOCH=$(date +%s)
aws xray get-service-graph --start-time $(($EPOCH-600)) --end-time $EPOCH
```

## HoneyComb

When creating a new dataset in Honeycomb it will provide all these installation insturctions



We'll add the following files to our `requirements.txt`

```
opentelemetry-api 
opentelemetry-sdk 
opentelemetry-exporter-otlp-proto-http 
opentelemetry-instrumentation-flask 
opentelemetry-instrumentation-requests
```

We'll install these dependencies:

```sh
pip install -r requirements.txt
```

Add to the `app.py`

```py
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
```


```py
# Initialize tracing and an exporter that can send data to Honeycomb
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)
```

```py
# Initialize automatic instrumentation with Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```

Add teh following Env Vars to `backend-flask` in docker compose

```yml
OTEL_EXPORTER_OTLP_ENDPOINT: "https://api.honeycomb.io"
OTEL_EXPORTER_OTLP_HEADERS: "x-honeycomb-team=${HONEYCOMB_API_KEY}"
OTEL_SERVICE_NAME: "${HONEYCOMB_SERVICE_NAME}"
```

You'll need to grab the API key from your honeycomb account:

```sh
export HONEYCOMB_API_KEY=""
export HONEYCOMB_SERVICE_NAME="Cruddur"
gp env HONEYCOMB_API_KEY=""
gp env HONEYCOMB_SERVICE_NAME="Cruddur"
```

## CloudWatch Logs


Add to the `requirements.txt`

```
watchtower
```

```sh
pip install -r requirements.txt
```


In `app.py`

```
import watchtower
import logging
from time import strftime
```

```py
# Configuring Logger to Use CloudWatch
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur')
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
LOGGER.info("some message")
```

```py
@app.after_request
def after_request(response):
    timestamp = strftime('[%Y-%b-%d %H:%M]')
    LOGGER.error('%s %s %s %s %s %s', timestamp, request.remote_addr, request.method, request.scheme, request.full_path, response.status)
    return response
```

We'll log something in an API endpoint
```py
LOGGER.info('Hello Cloudwatch! from  /api/activities/home')
```

Set the env var in your backend-flask for `docker-compose.yml`

```yml
      AWS_DEFAULT_REGION: "${AWS_DEFAULT_REGION}"
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
```

> passing AWS_REGION doesn't seems to get picked up by boto3 so pass default region instead


## Rollbar

https://rollbar.com/

Create a new project in Rollbar called `Cruddur`

Add to `requirements.txt`


```
blinker
rollbar
```

Install deps

```sh
pip install -r requirements.txt
```

We need to set our access token

```sh
export ROLLBAR_ACCESS_TOKEN=""
gp env ROLLBAR_ACCESS_TOKEN=""
```

Add to backend-flask for `docker-compose.yml`

```yml
ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"
```

Import for Rollbar

```py
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception
```

```py
rollbar_access_token = os.getenv('ROLLBAR_ACCESS_TOKEN')
@app.before_first_request
def init_rollbar():
    """init rollbar module"""
    rollbar.init(
        # access token
        rollbar_access_token,
        # environment name
        'production',
        # server root directory, makes tracebacks prettier
        root=os.path.dirname(os.path.realpath(__file__)),
        # flask already sets up logging
        allow_logging_basic_config=False)

    # send exceptions from `app` to rollbar, using flask's signal system.
    got_request_exception.connect(rollbar.contrib.flask.report_exception, app)
```

We'll add an endpoint just for testing rollbar to `app.py`

```py
@app.route('/rollbar/test')
def rollbar_test():
    rollbar.report_message('Hello World!', 'warning')
    return "Hello World!"
```

[Rollbar Flask Example](https://github.com/rollbar/rollbar-flask-example/blob/master/hello.py)


## [Note] Changes to Rollbar

During the original bootcamp cohort, there was a newer version of flask.
This resulted in rollback implementation breaking due to a change is the flask api.

If you notice rollbar is not tracking, utilize the code from this file:

https://github.com/omenking/aws-bootcamp-cruddur-2023/blob/week-x/backend-flask/lib/rollbar.py