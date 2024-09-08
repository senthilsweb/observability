# Observability as Side-car

I have provided a ready-to-use, zero-coding solution to implement observability for existing applications, independent of technology or platform. This Dockerized sidecar solution leverages OpenObserve and OpenTelemetry to seamlessly integrate telemetry and observability into your applications without any code changes.

![App with Observability](app-with-observability.drawio.svg "App with Observability")


**Implementation Highlights:**

- [x] Log and host metrics telemetry and observability implemented as a Dockerized sidecar.
- [x] Easily attachable to any application emitting logs to disk or via tail, utilizing disk-based storage.
- [x] Supports environment variables for secure management of secrets and configuration.
- [x] Customizable log stream names for flexibility.
- [x] Sample log file included from DBT for demonstration purposes.
- [x] Log data enrichment examples using a RegEx parser.
- [x] Otel and Fluent Bit collector examples for handling multiple log streams.
- [x] Basic username and password authentication for the OpenObserve API.
- [x] Examples for both logs and metrics to guide implementation.

## Obsewrvability Implementaiton Architecture

![Observability as side car](observability-side-car.drawio.svg "Observability as side car")

## Comparing Observability with Human Physiology

In the analogy between observability and human physiology, think of telemetry as the sensors and devices that collect vital data, similar to how doctors and modern gadgets gather health information.

For example, **doctors** use tools like:
- **ECG monitors** to track heart rhythms,
- **Blood pressure cuffs** to measure blood pressure,
- **Blood glucose monitors** to assess sugar levels,
- **Thermometers** to check body temperature.

These tools provide the raw data needed to assess a patient’s health.

Similarly, **modern lifestyle gadgets** like:
- **Fitbit** and **Apple Watch** continuously monitor metrics like heart rate, physical activity, sleep patterns, and even blood oxygen levels. 

These devices provide users with real-time data that can alert them to potential health issues or help them optimize their wellness routines.

In the context of observability, telemetry functions like these medical tools and gadgets, gathering logs, metrics, and traces from a software system. Observability then acts like the doctor or health app, interpreting this data to diagnose issues, monitor system health, and ensure optimal performance—keeping the system robust and resilient, much like a well-functioning body.

## Getting Started

1. Clone the repository and cd into `observability`

2. Create `.env` file in the root and adjust the variables to your requirements.

**.env file**

```env
# .env
OPENOBSERVE_HOST=openobserve
OPENOBSERVE_PORT=5080
OPENOBSERVE_USERNAME=root_user@example.com
OPENOBSERVE_PASSWORD=root_password
```

3. Edit `config.yml` (and optional `fluent-bit.conf`) located inside the root `config` folder with the path of the log files. 

**Otel Configuration**

> **ℹ️ Info**
> 
> For advanced OpenTelemetry `filelog` receiver configurations, please refer to the `config-advanced.yml` file. This configuration includes file routing, data enrichment using regex parsers, and additional attribute handling.


```yaml
# config.yml
receivers:
  filelog:
    include: 
      - /etc/logs/**/*.log
    multiline:
     line_start_pattern: '^\x1b\['
    start_at: beginning
    include_file_name: true  # Ensure the file name is included in the logs
    include_file_path: true  # Ensure the file path is included in the logs
    operators:
      - field: attributes.log_type
        type: add
        value: file
      - type: regex_parser
        regex: '(?P<timestamp>\d{2}:\d{2}:\d{2}\.\d{6})\s+\[(?P<log_level>\w+)\s*\]\s+\[(?P<thread>\w+)\]:\s+(?P<message>.*)'
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%d %H:%M:%S'
        message:
          parse_from: attributes.body
      - type: regex_parser
        regex: '\*/(?P<sql>[\s\S]*)'
        sql:
          parse_from: attributes.body
  hostmetrics:
    collection_interval: 30s
    root_path: /hostfs
    scrapers:
      cpu: {}
      load: {}
      memory: {}
      disk: {}
      filesystem: {}
      network: {}

processors:
    attributes/app_metadata:
      actions:
        - key: client_id
          value: 1123
          action: insert
        - key: app_name
          value: dbt
          action: insert
               
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
  prometheusremotewrite:
    endpoint: http://${OPENOBSERVE_HOST}:${OPENOBSERVE_PORT}/api/default/prometheus/api/v1/write
    auth:
      authenticator: basicauth/client

service:
  extensions: [basicauth/client]
  pipelines:
    logs:
      receivers: [filelog]
      processors: [attributes/app_metadata]
      exporters: [logging,otlphttp/openobserve]
    metrics:
      receivers: [hostmetrics]
      processors: []
      exporters: [prometheusremotewrite]

extensions:
  basicauth/client:
    client_auth: 
      username: ${OPENOBSERVE_USERNAME}
      password: ${OPENOBSERVE_PASSWORD}
```

```yaml
# config-advanced.yml
receivers:
  filelog:
    include: 
      - /etc/logs/dbt-data-pipeline/*.log
    multiline:
     line_start_pattern: '^\x1b\[' # Example for dbt log pattern
    start_at: beginning
    include_file_name: true 
    include_file_path: true 
    operators:
      - field: attributes.log_type
        type: add
        value: file
      - type: router
        id: route_by_filename
        routes:
          - output: extract_metadata_from_filepath
            expr: 'body matches "^\\x1b\\["' # Match only dbt logs
            #expr: 'strings.contains(attributes["log.file.name"], "dbt")'
          - output: default_route 
            expr: 'true' # catchall routes
      - type: regex_parser
        id: extract_metadata_from_filepath
        regex: '(?P<timestamp>\d{2}:\d{2}:\d{2}\.\d{6})\s+\[(?P<log_level>\w+)\s*\]\s+\[(?P<thread>\w+)\]:\s+(?P<message>.*)' # Example for dbt log pattern
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%d %H:%M:%S'
        message:
          parse_from: attributes.body
        output: attach_service
      - field: attributes.service
        id: attach_service
        type: add
        value: "DBT Data Pipeline"  
      - type: regex_parser # Example for dbt log pattern to extract sql statement
        regex: '\*/(?P<sql>[\s\S]*)'
        sql:
          parse_from: attributes.body
      - type: noop # For Catchall route to retain the body as-is
        id: default_route

  filelog/tomcat:
    include: 
      - /etc/logs/**/catalina.out
    multiline:
     line_start_pattern: '^\d{2}-[A-Za-z]{3}-\d{4}\s\d{2}:\d{2}:\d{2}\.\d{3}' # Example for tomcat log pattern
    start_at: beginning
    include_file_name: true  
    include_file_path: true  
    operators:
      - field: attributes.log_type
        type: add
        value: file
      - field: attributes.service
        type: add
        value: "Tomcat Web App"
      - type: regex_parser
        regex: '(?P<date>\d{2}-[A-Za-z]{3}-\d{4})\s(?P<time>\d{2}:\d{2}:\d{2}\.\d{3})\s(?P<log_level>[A-Z]+)\s\[(?P<thread>[^\]]+)\]\s(?P<class>[^\s]+)\s(?P<message>.*)'
        timestamp:
          parse_from: attributes.date, attributes.time
          layout: '%d-%b-%Y %H:%M:%S.%f'
        message:
          parse_from: attributes.body

  filelog/apache-common:
    include: 
      - /etc/logs/apache-common/apache-common.log
    start_at: beginning
    include_file_name: true  
    include_file_path: true  
    operators:
      - field: attributes.log_type
        type: add
        value: file
      - field: attributes.service
        type: add
        value: "Apache Common log"
processors:
    attributes/app_metadata:
      actions:
        - key: app_version
          value: 7.2.0.1
          action: insert
        - key: app_name
          value: TemplrJS
          action: insert
               
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
  telemetry:
    logs:
      level: debug
  extensions: [basicauth/client]
  pipelines:
    logs:
      receivers: [filelog,filelog/tomcat,filelog/apache-common]
      processors: [attributes/app_metadata]
      exporters: [logging,otlphttp/openobserve]

extensions:
  basicauth/client:
    client_auth: 
      username: ${OPENOBSERVE_USERNAME}
      password: ${OPENOBSERVE_PASSWORD}
    

```

4. **Mount application log folder path to docker volume**


```yaml
# docker-compose.yml
services:
  otel:
    image: otel/opentelemetry-collector-contrib:0.106.1
    restart: unless-stopped
    command: [ "--config=/etc/config.yml" ]
    volumes:
      - ./config/otel/config.yml:/etc/config.yml
      #- ./config/otel/config-advanced.yml:/etc/config.yml
      - ./demo-logs:/etc/logs
      - /:/hostfs:ro
    depends_on:
      - openobserve
    environment:
      OPENOBSERVE_HOST: ${OPENOBSERVE_HOST}
      OPENOBSERVE_PORT: ${OPENOBSERVE_PORT}
      OPENOBSERVE_USERNAME: ${OPENOBSERVE_USERNAME}
      OPENOBSERVE_PASSWORD: ${OPENOBSERVE_PASSWORD}
    networks:
      - nw_openobserve

  openobserve:
    image: openobserve/openobserve:v0.10.8
    container_name: openobserve
    restart: always
    ports:
      - 5080:5080
    volumes:
      - vol_openobserve:/data
    environment:
      ZO_ROOT_USER_PASSWORD: ${OPENOBSERVE_PASSWORD}
      ZO_ROOT_USER_EMAIL: ${OPENOBSERVE_USERNAME}
      ZO_DATA_DIR: "/data"
      ZO_COMPACT_DATA_RETENTION_DAYS: "5"
      ZO_TELEMETRY: "false"
    networks:
      - nw_openobserve
  log_fluentbit:
    container_name: log_fluentbit
    image: fluent/fluent-bit:2.2.3
    volumes:
      - ./config/fluentbit:/fluent-bit/etc/
      - ./demo-logs/dbt-data-pipeline:/var/log
    restart: always
    environment:
      OPENOBSERVE_HOST: ${OPENOBSERVE_HOST:-openobserve}
      OPENOBSERVE_PORT: ${OPENOBSERVE_PORT:-5080}
      OPENOBSERVE_USERNAME: ${OPENOBSERVE_USERNAME}
      OPENOBSERVE_PASSWORD: ${OPENOBSERVE_PASSWORD}
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    depends_on:
      - openobserve
    networks:
      - nw_openobserve

volumes:
  vol_openobserve: {}

networks:
  nw_openobserve:
```

5. **Run by docker-compose up**

```
docker-compose up -d
```

6. Access the openobserve UI @ `http://localhost:5080` with login and password as `root_user@example.com` |  `root_password` 

7. **Shut down the application**

```
docker-compose down
```

8. **Shut down the application and remove volume**

```
docker-compose down -v
```
