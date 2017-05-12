---
layout: post
title: "prometheus, blackbox_exporter, komachi_heartbeat"
date: 2017-05-12 22:00:33 +0900
comments: true
categories: prometheus
---
最近[Prometheus](https://prometheus.io/)の導入に向けて検証中。
今日は外形監視の設定を試した。

外形監視には[blackbox_exporter](https://github.com/prometheus/blackbox_exporter)を使う。
[komachi_heartbeat](https://github.com/mitaku/komachi_heartbeat)を導入しているので、レスポンスとして`heartbeat:ok`が返ってくるかチェックできればよい。

```
# prometheus.yml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_komachi_heartbeat]
      path: ['/path/to/heartbeat']
    static_configs:
      - targets:
        - example.com
    relabel_configs:
      - source_labels: [__address__, __param_path]
        regex: (.*);(.*)
        target_label: __param_target
        replacement: http://${1}${2}
      - source_labels: []
        regex: .*
        target_label: __address__
        replacement: 127.0.0.1:9115  # blackbox_exporterが動いているIPアドレスにする
```

```
# blackbox.yml
modules:
  http_komachi_heartbeat:
    prober: http
    timeout: 5s
    http:
      fail_if_not_matches_regexp:
      - "heartbeat:ok"
```

複数あるwebサーバーそれぞれに対してリクエストを投げるのは以下のような感じ。


```
# prometheus.yml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_komachi_heartbeat]
      path: ['/path/to/heartbeat']
    ec2_sd_configs:
      - region: ap-northeast-1
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_tag_role]
        regex: web
        action: keep
      - source_labels: [__meta_ec2_tag_Name]
        regex: (.*)
        target_label: name
        replacement: ${1}
      - source_labels: [__param_target]
        regex: (.*)
        target_label: instance
        replacement: ${1}
      - source_labels: [__meta_ec2_public_ip, __param_path]
        regex: (.*);(.*)
        target_label: __param_target
        replacement: http://${1}${2}
      - source_labels: []
        regex: .*
        target_label: __address__
        replacement: 127.0.0.1:9115  # blackbox_exporterが動いているIPアドレスにする
```


```
# blackbox.yml
modules:
  http_komachi_heartbeat:
    prober: http
    timeout: 5s
    http:
      headers:
        Host: example.com
      fail_if_not_matches_regexp:
      - "heartbeat:ok"
```

監視したいドメインが複数ある場合、ドメインごとにblackbox.ymlの設定を増やしていかないといけないのが微妙かもしれない。
