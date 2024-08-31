# openobserve
OpenTelemetry logging and visualization using OpenObserve as the backend, with Otel and Fluent Bit collectors. Built to easily expand for traces and metrics with Prometheus in the future.



**Additional Details:**

- Log observability is implemented as a sidecar using Docker.
- Easily attachable to any application, emitting logs to disk or tail.
- Utilizes disk-based storage for logs.
- Supports environment variables for managing secrets and configuration variables securely.
- Allows custom log stream names.
- Provides examples for both Otel and Fluent Bit collectors handling multiple log streams.
- Includes basic username and password authentication for the OpenObserve API. 


**How to customize to your requirements?**

1. Clone the repository and cd into `openobserve`

2. Create `.env` file in the root and adjust the variables to your requirements.

**.env file**

```.env
OPENOBSERVE_HOST=openobserve
OPENOBSERVE_PORT=5080
OPENOBSERVE_USERNAME=root_user@example.com
OPENOBSERVE_PASSWORD=root_password
```

3. Edit `config.yml` and `fluent-bit.conf` located inside your root `config` folder with the path of the log files. You can use bothor disable/remove any one of the collectors.

**Fluent-bit Configuration**

```fluent-bit.conf
[INPUT]
    Name         tail
    Path         /var/log/*.log
    #Tag          fluentbit_logs_collector # This will be the log stream name
    Path_Key     filename
    Skip_Empty_Lines  On
    Read_from_Head    true

######## You can add as many [INPUTS] ########
#[INPUT]
#    Name         tail
#    Path         /var/log/*.out
#    Path_Key     filename


[OUTPUT]
    Name http
    Match *
    URI  /api/default/fluentbit/_json
    Host ${OPENOBSERVE_HOST}
    Port ${OPENOBSERVE_PORT}
    tls  Off
    tls.verify Off
    Format json
    Json_date_key    _timestamp
    Json_date_format iso8601
    HTTP_User ${OPENOBSERVE_USERNAME}
    HTTP_Passwd ${OPENOBSERVE_PASSWORD}
```

**Otel Configuration**

```config.yml
receivers:
  filelog:
    include: ['/etc/logs/*.txt']
    start_at: beginning
    include_file_name: true  # Ensure the file name is included in the logs
    include_file_path: true  # Ensure the file path is included in the logs
               
exporters:
  logging:
    verbosity: detailed

  otlphttp/openobserve:
    endpoint: http://${OPENOBSERVE_HOST}:${OPENOBSERVE_PORT}/api/default
    headers:
    #  Authorization: "Basic cm9vdEBleGFtcGxlLmNvbTpSVW5hSlFkMUpuTDVqV29v" 
      stream-name: otel_logs_collector # This will be the log stream name
    auth:
      authenticator: basicauth/client

service:
  extensions: [basicauth/client]
  pipelines:
    logs:
      receivers: [filelog]
      processors: []
      exporters: [logging,otlphttp/openobserve]

extensions:
  basicauth/client:
    client_auth: 
      username: ${OPENOBSERVE_USERNAME}
      password: ${OPENOBSERVE_PASSWORD}
```

**Run by docker-compose up**

```
docker-compose up -d
```
Access the openobserve UI @ `http://localhost:5080` with login and password as `root_user@example.com` |  `root_password` 

**Shut down the application**

```
docker-compose down
```

**Shut down the application and remove volume**

```
docker-compose down -v
```

