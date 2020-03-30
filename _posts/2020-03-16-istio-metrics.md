---
title: istio1.5 metrics
author: Noodles
layout: post
comments: true
permalink: /2020/03/16/istio-metrics
photos:
- http://q61qnv6m4.bkt.clouddn.com/2020/0316/istio.png
---

本文对istio1.5版本下envoy prometheus指标进行梳理

<!--more-->

 ---------------------------------------------------

  istio 默认使用prometheus来存储遥测数据。

### prometheus 配置

``` yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: istio-system
  labels:
    app: prometheus
    release: istio
data:
  prometheus.yml: |-
    global:
      scrape_interval: 15s
    scrape_configs:

    # Mixer scrapping. Defaults to Prometheus and mixer on same namespace.
    #
    - job_name: 'istio-mesh'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-telemetry;prometheus

    # Scrape config for envoy stats
    - job_name: 'envoy-stats'
      metrics_path: /stats/prometheus
      kubernetes_sd_configs:
      - role: pod

      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        action: keep
        regex: '.*-envoy-prom'
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:15090
        target_label: __address__
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod_name

    - job_name: 'istio-policy'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system


      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-policy;http-policy-monitoring

    - job_name: 'istio-telemetry'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system

      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-telemetry;http-monitoring

    - job_name: 'pilot'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system

      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-pilot;http-monitoring

    - job_name: 'galley'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system

      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-galley;http-monitoring

    - job_name: 'citadel'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system

      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-citadel;http-monitoring

    - job_name: 'sidecar-injector'

      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system

      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-sidecar-injector;http-monitoring

    # scrape config for API servers
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - default
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kubernetes;https

    # scrape config for nodes (kubelet)
    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    # Scrape config for Kubelet cAdvisor.
    #
    # This is required for Kubernetes 1.7.3 and later, where cAdvisor metrics
    # (those whose names begin with 'container_') have been removed from the
    # Kubelet metrics endpoint.  This job scrapes the cAdvisor endpoint to
    # retrieve those metrics.
    #
    # In Kubernetes 1.7.0-1.7.2, these metrics are only exposed on the cAdvisor
    # HTTP endpoint; use "replacement: /api/v1/nodes/${1}:4194/proxy/metrics"
    # in that case (and ensure cAdvisor's HTTP server hasn't been disabled with
    # the --cadvisor-port=0 Kubelet flag).
    #
    # This job is not necessary and should be removed in Kubernetes 1.6 and
    # earlier versions, or it will cause the metrics to be scraped twice.
    - job_name: 'kubernetes-cadvisor'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    # scrape config for service endpoints.
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:  # If first two labels are present, pod should be scraped  by the istio-secure job.
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_sidecar_istio_io_status]
        action: drop
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_annotation_istio_mtls]
        action: drop
        regex: (true)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod_name
```

  在配置中，除了`k8s`的一些指标外，在`envoy-stats`job的配置中，可以看到istio也通过envoy暴露出的`/stats/prometheus`接口将envoy内部数据采集上来。


#### envoy-proxy bootstrap yaml
  
  istio-init镜像生成的envoy是一个json文件。位于`istio-proxy`container中的`/etc/istio/proxy`目录下，文件名为: `envoy-rev0.json`。这里为了方便阅读，将json文件转成了yaml。注意，此处仅仅是为了阅读分析使用。如果直接使用json转yaml的在线工具生成，此yaml在个别配置项可能并不符合envoy的配置规则。

```yaml

---
node:
  id: sidecar~10.244.2.14~front-6fcd8fd47-bg4jt.default~default.svc.cluster.local
  cluster: front.default
  locality: {}
  metadata:
    CLUSTER_ID: Kubernetes
    CONFIG_NAMESPACE: default
    EXCHANGE_KEYS: NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,CANONICAL_TELEMETRY_SERVICE,MESH_ID,SERVICE_ACCOUNT
    INSTANCE_IPS: 10.244.2.14
    INTERCEPTION_MODE: REDIRECT
    ISTIO_PROXY_SHA: istio-proxy:73f240a29bece92a8882a36893ccce07b4a54664
    ISTIO_VERSION: 1.5.0
    LABELS:
      app: front
      pod-template-hash: 6fcd8fd47
      security.istio.io/tlsMode: istio
      service.istio.io/canonical-name: front
      service.istio.io/canonical-revision: latest
    MESH_ID: cluster.local
    NAME: front-6fcd8fd47-bg4jt
    NAMESPACE: default
    OWNER: kubernetes://apis/apps/v1/namespaces/default/deployments/front
    POD_NAME: front-6fcd8fd47-bg4jt
    POD_PORTS: '[{"containerPort":8080,"protocol":"TCP"}]'
    SDS: 'true'
    SERVICE_ACCOUNT: default
    TRUSTJWT: 'true'
    WORKLOAD_NAME: front
stats_config:
  use_all_default_tags: false
  stats_tags:
  - tag_name: cluster_name
    regex: "^cluster\\.((.+?(\\..+?\\.svc\\.cluster\\.local)?)\\.)"
  - tag_name: tcp_prefix
    regex: "^tcp\\.((.*?)\\.)\\w+?$"
  - regex: "(response_code=\\.=(.+?);\\.;)|_rq(_(\\.d{3}))$"
    tag_name: response_code
  - tag_name: response_code_class
    regex: _rq(_(\dxx))$
  - tag_name: http_conn_manager_listener_prefix
    regex: "^listener(?=\\.).*?\\.http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
  - tag_name: http_conn_manager_prefix
    regex: "^http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
  - tag_name: listener_address
    regex: "^listener\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
  - tag_name: mongo_prefix
    regex: "^mongo\\.(.+?)\\.(collection|cmd|cx_|op_|delays_|decoding_)(.*?)$"
  - regex: "(reporter=\\.=(.+?);\\.;)"
    tag_name: reporter
  - regex: "(source_namespace=\\.=(.+?);\\.;)"
    tag_name: source_namespace
  - regex: "(source_workload=\\.=(.+?);\\.;)"
    tag_name: source_workload
  - regex: "(source_workload_namespace=\\.=(.+?);\\.;)"
    tag_name: source_workload_namespace
  - regex: "(source_principal=\\.=(.+?);\\.;)"
    tag_name: source_principal
  - regex: "(source_app=\\.=(.+?);\\.;)"
    tag_name: source_app
  - regex: "(source_version=\\.=(.+?);\\.;)"
    tag_name: source_version
  - regex: "(destination_namespace=\\.=(.+?);\\.;)"
    tag_name: destination_namespace
  - regex: "(destination_workload=\\.=(.+?);\\.;)"
    tag_name: destination_workload
  - regex: "(destination_workload_namespace=\\.=(.+?);\\.;)"
    tag_name: destination_workload_namespace
  - regex: "(destination_principal=\\.=(.+?);\\.;)"
    tag_name: destination_principal
  - regex: "(destination_app=\\.=(.+?);\\.;)"
    tag_name: destination_app
  - regex: "(destination_version=\\.=(.+?);\\.;)"
    tag_name: destination_version
  - regex: "(destination_service=\\.=(.+?);\\.;)"
    tag_name: destination_service
  - regex: "(destination_service_name=\\.=(.+?);\\.;)"
    tag_name: destination_service_name
  - regex: "(destination_service_namespace=\\.=(.+?);\\.;)"
    tag_name: destination_service_namespace
  - regex: "(request_protocol=\\.=(.+?);\\.;)"
    tag_name: request_protocol
  - regex: "(response_flags=\\.=(.+?);\\.;)"
    tag_name: response_flags
  - regex: "(grpc_response_status=\\.=(.*?);\\.;)"
    tag_name: grpc_response_status
  - regex: "(connection_security_policy=\\.=(.+?);\\.;)"
    tag_name: connection_security_policy
  - regex: "(permissive_response_code=\\.=(.+?);\\.;)"
    tag_name: permissive_response_code
  - regex: "(permissive_response_policyid=\\.=(.+?);\\.;)"
    tag_name: permissive_response_policyid
  - regex: "(cache\\.(.+?)\\.)"
    tag_name: cache
  - regex: "(component\\.(.+?)\\.)"
    tag_name: component
  - regex: "(tag\\.(.+?)\\.)"
    tag_name: tag
  - regex: "(source_canonical_service=\\.=(.+?);\\.;)"
    tag_name: source_canonical_service
  - regex: "(destination_canonical_service=\\.=(.+?);\\.;)"
    tag_name: destination_canonical_service
  - regex: "(source_canonical_revision=\\.=(.+?);\\.;)"
    tag_name: source_canonical_revision
  - regex: "(destination_canonical_revision=\\.=(.+?);\\.;)"
    tag_name: destination_canonical_revision
  stats_matcher:
    inclusion_list:
      patterns:
      - prefix: reporter=
      - prefix: component
      - prefix: cluster_manager
      - prefix: listener_manager
      - prefix: http_mixer_filter
      - prefix: tcp_mixer_filter
      - prefix: server
      - prefix: cluster.xds-grpc
      - suffix: ssl_context_update_by_sds
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 15000
dynamic_resources:
  lds_config:
    ads: {}
  cds_config:
    ads: {}
  ads_config:
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: xds-grpc
static_resources:
  clusters:
  - name: prometheus_stats
    type: STATIC
    connect_timeout: 0.250s
    lb_policy: ROUND_ROBIN
    hosts:
    - socket_address:
        protocol: TCP
        address: 127.0.0.1
        port_value: 15000
  - name: sds-grpc
    type: STATIC
    http2_protocol_options: {}
    connect_timeout: 10s
    lb_policy: ROUND_ROBIN
    hosts:
    - pipe:
        path: "/etc/istio/proxy/SDS"
  - name: xds-grpc
    type: STRICT_DNS
    dns_refresh_rate: 300s
    dns_lookup_family: V4_ONLY
    connect_timeout: 10s
    lb_policy: ROUND_ROBIN
    tls_context:
      common_tls_context:
        alpn_protocols:
        - h2
        tls_certificate_sds_secret_configs:
        - name: default
          sds_config:
            api_config_source:
              api_type: GRPC
              grpc_services:
              - envoy_grpc:
                  cluster_name: sds-grpc
        validation_context:
          trusted_ca:
            filename: "./var/run/secrets/istio/root-cert.pem"
          verify_subject_alt_name:
          - istiod.istio-system.svc
    hosts:
    - socket_address:
        address: istiod.istio-system.svc
        port_value: 15012
    circuit_breakers:
      thresholds:
      - priority: DEFAULT
        max_connections: 100000
        max_pending_requests: 100000
        max_requests: 100000
      - priority: HIGH
        max_connections: 100000
        max_pending_requests: 100000
        max_requests: 100000
    upstream_connection_options:
      tcp_keepalive:
        keepalive_time: 300
    max_requests_per_connection: 1
    http2_protocol_options: {}
  - name: zipkin
    type: STRICT_DNS
    dns_refresh_rate: 300s
    dns_lookup_family: V4_ONLY
    connect_timeout: 1s
    lb_policy: ROUND_ROBIN
    hosts:
    - socket_address:
        address: zipkin.istio-system
        port_value: 9411
  listeners:
  - address:
      socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 15090
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: AUTO
          stat_prefix: stats
          route_config:
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/stats/prometheus"
                route:
                  cluster: prometheus_stats
          http_filters:
            name: envoy.router
tracing:
  http:
    name: envoy.zipkin
    config:
      collector_cluster: zipkin
      collector_endpoint: "/api/v2/spans"
      collector_endpoint_version: HTTP_JSON
      trace_id_128bit: 'true'
      shared_span_context: 'false'

```

  istio只对envoy进行了最小化的统计配置，收集的项主要包括:

  - cluster_manager
  - listener_manager
  - http_mixer_filter
  - tcp_mixer_filter
  - server
  - cluster.xds-grpc


