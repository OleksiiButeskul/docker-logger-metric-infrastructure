# Docker infrastructure for R&D

Docker base infrastructure consists from next services:
* [Fluentd](https://docs.fluentd.org/) log aggregator - will process logs sent by Docker Fluentd logging driver
* [OpenSearch](https://opensearch.org/docs/latest) and OpenSearch Dashboard - will store logs and visualize them using UI application (AWS ElasticSearch and Kibana alternatives)
* [cAdviser](https://github.com/google/cadvisor) - metric collector of the Docker containers
* [Prometheus](https://prometheus.io/docs/introduction/overview/) - will store metrics collected by cAdviser
* [Grafana](https://grafana.com/docs/grafana/latest/introduction/) - will visualize collected metrics using cAdvisor Dashboard

Before starting `docker-compose` file from `infrastructure` folder, we need to create external network to link them. 
We will use docker [Fluentd logging driver](https://docs.fluentd.org/container-deployment/docker-logging-driver) for our services 
to stream logs from Docker containers to Fluentd log aggregator. To run container with `Fluentd logging driver` enabled 
we need to launch Fluentd aggregator first, otherwise container launch will be failed.

### 1) Docker external network creation
`Fluentd logging driver` is required static IP address to work (Docker won't handle ip address based on host name of the Fluentd service). 
We need to create custom network with providing `subnet` address pool available on server machine using next command:

```shell
docker network create fluentd-network --subnet=172.65.0.0/24
```

In case `subnet` address pool will be changed, we need to update both `docker-compose` files for Fluentd service 
static IP assign and use Fluentd service static IP address to update Fluentd logging driver setting in service and metric `docker-compose` file.

Fluentd docker-compose file:
```yaml
  fluentd:
    build: .
    container_name: fluentd
    image: fluentd-elasticsearch:0.14.3
    restart: always
    networks:
      default:
        ipv4_address: 172.65.0.15 # static IP address related to subnet of the network
```

Service and metrics docker-compose file:
```yaml
  web:
    image: httpd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: 172.65.0.15:24224 # static IP address of the Fluentd service
        tag: httpd.access
```

### 2) Base infrastructure for logs aggregation and metric collection
After we created custom external network, we need to start `docker-compose` file with base Docker infrastructure from `infrastructure` folder first. 
It will prepare base infrastructure for collecting and monitoring data of our services we want to launch in the future.

In case we have some changes in the files for Fluentd service, we need to rebuild image first using next command:
```shell
docker-compose build
```
After images docker were built, we can launch our base infrastructure using next command:
```shell
docker-compose up
```


### 3) Preparing Service infrastructure Docker compose
After we launched base Docker infrastructure, we can launch our service's infrastructure in separate docker-compose file. 
Service infrastructure docker-compose file should have next configuration to work with base Docker infrastructure:

1) We need to replace default bridge network with our external network created in the first step:
```yaml
networks:
  default:
    name: fluentd-network
    external: true
```
2) Replace default `json` Docker logger with `Fluentd logger driver` for our Java services and provide tag `java` in the driver configuration:
```yaml
logging:
      driver: "fluentd"
      options:
        fluentd-address: 172.65.0.15:24224 # static IP address of the Fluentd service
        tag: java
```

In case base Docker infrastructure was launched and configuration in Services docker-compose file was done correctly, 
user should be able to see logs in OpenSearch Dashboard and containers metrics in Grafana Dashboard.

### 4) OpenSearch Dashboard and Grafana configurations
After infrastructure was launched we need to create index pattern in OpenSearch Dashboard to check logs from containers 
using `containers*` index pattern in `Stack Management` > `Index Patterns` configuration section. 
For time field we need to select `@timestamp` field from log. From Discover dashboard we can check logs of our services.
After index pattern was configured, we need to configure index retention policy to delete old indexes in 
`Index Management` > `Index policies` OpenSearch plugin configuration. 

In Grafana during first login it's better to change password to more secure password after deployment to AWS EC2 instance. 
For local development environment we can use credentials from environment variable provided for Grafana service. 
After we logged in with admin account we can use existing [cAdvisor exporter](https://grafana.com/grafana/dashboards/14282) dashboard from Grafana. 
To add existing dashboard we need to copy ID of the dashboard and using `Import` button on `Dashboards` page import it 
by providing dashboard ID and selecting our Prometheus datasource from list. After dashboard was imported, we will see 
our metrics collected by cAdvisor for our containers.