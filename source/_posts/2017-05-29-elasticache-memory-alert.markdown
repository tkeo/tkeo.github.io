---
layout: post
title: "ElastiCacheの空きメモリのアラートを設定した"
date: 2017-05-29 21:44:01 +0900
comments: true
categories: prometheus
---
メモリの空きが20%切ったらwarning投げるとか、アラートのしきい値を割合で設定したい。
CloudWatchで取れるのは使用しているサイズなので、なんとかして最大値を取得して計算する必要がある。

最大値自体は[describe-cache-parameters](http://docs.aws.amazon.com/cli/latest/reference/elasticache/describe-cache-parameters.html)で取れるのでファイルに書き出しておいて、node_exporterのtextfile collectorで読み込ませることにした。

ファイルは下のような感じのスクリプトで生成した。
あらかじめ取得した値を列挙したけど、めったにノード追加しないので都度取得でもよかったかもしれない。

```
require "aws-sdk-core"

# elasticache.describe_cache_parameters(cache_parameter_group_name: "default.memcached1.4").
#   cache_node_type_specific_parameters.detect {|param| param.parameter_name == "max_cache_memory" }.
#   cache_node_type_specific_values.map {|value| [value.cache_node_type, value.value] }.to_h

memcached_max_memories = {
  "cache.c1.xlarge"=>"6600",
  "cache.m1.large"=>"7100",
  "cache.m1.medium"=>"3350",
  "cache.m1.small"=>"1300",
  "cache.m1.xlarge"=>"14600",
  "cache.m2.2xlarge"=>"33800",
  "cache.m2.4xlarge"=>"68000",
  "cache.m2.xlarge"=>"16700",
  "cache.m3.2xlarge"=>"28600",
  "cache.m3.large"=>"6200",
  "cache.m3.medium"=>"2850",
  "cache.m3.xlarge"=>"13600",
  "cache.m4.10xlarge"=>"158355",
  "cache.m4.2xlarge"=>"30412",
  "cache.m4.4xlarge"=>"62234",
  "cache.m4.large"=>"6573",
  "cache.m4.xlarge"=>"14618",
  "cache.r3.2xlarge"=>"59600",
  "cache.r3.4xlarge"=>"120600",
  "cache.r3.8xlarge"=>"242600",
  "cache.r3.large"=>"13800",
  "cache.r3.xlarge"=>"29100",
  "cache.t1.micro"=>"213",
  "cache.t2.medium"=>"3301",
  "cache.t2.micro"=>"555",
  "cache.t2.small"=>"1588",
  "dax.r3.2xlarge"=>"59600",
  "dax.r3.4xlarge"=>"120600",
  "dax.r3.8xlarge"=>"242600",
  "dax.r3.large"=>"13800",
  "dax.r3.xlarge"=>"29100"
}

# elasticache.describe_cache_parameters(cache_parameter_group_name: "default.redis2.8").
#   cache_node_type_specific_parameters.detect {|param| param.parameter_name == "maxmemory" }.
#   cache_node_type_specific_values.map {|value| [value.cache_node_type, value.value] }.to_h

redis_max_memories = {
  "cache.c1.xlarge"=>"6501171200",
  "cache.m1.large"=>"7025459200",
  "cache.m1.medium"=>"3093299200",
  "cache.m1.small"=>"943718400",
  "cache.m1.xlarge"=>"14889779200",
  "cache.m2.2xlarge"=>"35022438400",
  "cache.m2.4xlarge"=>"70883737600",
  "cache.m2.xlarge"=>"17091788800",
  "cache.m3.2xlarge"=>"29989273600",
  "cache.m3.large"=>"6501171200",
  "cache.m3.medium"=>"2988441600",
  "cache.m3.xlarge"=>"14260633600",
  "cache.m4.10xlarge"=>"166047614239",
  "cache.m4.2xlarge"=>"31889126359",
  "cache.m4.4xlarge"=>"65257290629",
  "cache.m4.large"=>"6892593152",
  "cache.m4.xlarge"=>"15328501760",
  "cache.r3.2xlarge"=>"62495129600",
  "cache.r3.4xlarge"=>"126458265600",
  "cache.r3.8xlarge"=>"254384537600",
  "cache.r3.large"=>"14470348800",
  "cache.r3.xlarge"=>"30513561600",
  "cache.t1.micro"=>"142606336",
  "cache.t2.medium"=>"3461349376",
  "cache.t2.micro"=>"581959680",
  "cache.t2.small"=>"1665138688",
  "dax.r3.2xlarge"=>"62495129600",
  "dax.r3.4xlarge"=>"126458265600",
  "dax.r3.8xlarge"=>"254384537600",
  "dax.r3.large"=>"14470348800",
  "dax.r3.xlarge"=>"30513561600"
}

elasticache = Aws::ElastiCache::Client.new(region: "ap-northeast-1")
cache_clusters = elasticache.describe_cache_clusters(show_cache_node_info: true).cache_clusters
cache_clusters.each do |cluster|
  if cluster.engine == "memcached"
    max_memory = memcached_max_memories[cluster.cache_node_type].to_i * 1024**2
  else
    max_memory = redis_max_memories[cluster.cache_node_type].to_i
  end
  cluster.cache_nodes.each do |node|
    puts %(aws_elasticache_max_memory_bytes{cache_cluster_id="#{cluster.cache_cluster_id}",cache_node_id="#{node.cache_node_id}"} #{max_memory})
  end
end
```

出力はこんな感じ。

```
aws_elasticache_max_memory_bytes{cache_cluster_id="memcache-example",cache_node_id="0001"} 581959680
```

これをnode_exporter経由で取得した値と、cloudwatch_exporterで取得した値ではラベルが違うので、`metric_relabel_configs`を使って同じになるように調整する必要がある。
exporter側でjobとinstanceを設定してもprometheusで読み込むときにprefixを付けてリネームしてしまうのでこうせざるを得なかった。

```
# prometheus.yml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9100']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: aws.*
        target_label: job
        replacement: cloudwatch
      - source_labels: [cache_cluster_id, cache_node_id]
        regex: (.+);(.+)
        target_label: instance
        replacement: cache.${1}.${2}
  - job_name: 'cloudwatch'
    static_configs:
      - targets: ['localhost:9106']
    metric_relabel_configs:
      - source_labels: [cache_cluster_id, cache_node_id]
        regex: (.+);(.+)
        target_label: instance
        replacement: cache.${1}.${2}
      - action: labeldrop
        regex: exported_job
```

あとはアラートのruleを設定するだけ。

```
ALERT elasticache_memcached_used_memory
  IF aws_elasticache_bytes_used_for_cache_items_average / aws_elasticache_max_memory_bytes > 0.8
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary = "Used memory is large",
    description = "{{ $labels.cache_cluster_id }}({{ $labels.cache_node_id }}): {{ humanize $value }}B"
  }

ALERT elasticache_redis_used_memory
  IF aws_elasticache_bytes_used_for_cache_average / aws_elasticache_max_memory_bytes > 0.5
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary = "Used memory is large",
    description = "{{ $labels.cache_cluster_id }}({{ $labels.cache_node_id }}): {{ humanize $value }}B"
  }
```

はじめはredis,memcached共通で使える`aws_elasticache_freeable_memory_average`を使って設定したら1.4とかふざけた値になったので、使用量を元に計算するように変更した。
これって自分の環境だけなのかな…
