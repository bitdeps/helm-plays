# Elasticsearch + Logstash

This play focus is on logstash configuration. ILM serves for task such as indexes rolling over, merging, shard shrinking and retention:

  * Using [ILM configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)

    ILM hot-worm-cold allows to allocate indexes on different nodes [see here](https://www.elastic.co/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management).

    **Note**: The freeze index API was removed in 8.0. Frozen indices are no longer useful due to recent improvements in heap memory usage.

  * The newer Elasticsearch 8 also provides data stream lifecycle:

    - [Data streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html)
    - [Data streams lifecycle](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-stream-lifecycle.html)

## Refs

* [Logstash elasticsearch output](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html)

## Logstash >= 8.8 with ILM

Note it's better to specifically disable `data_stream`.

Main takeaway points:
* When manually managing index `template_name` set a high priority, if logstash manages the template it sets high priority.
  Otherwise the built-in default `logs` index template for data streams takes precedence it matches index patterns `logs-*-*`

  **Note:** Use a priority higher than 200 to avoid collisions with built-in templates. See Avoid index pattern collisions!!!
* `ilm_rollover_alias` is used instead of `index` when ilm is enabled.
* `ilm_policy` **custom-ilm** should be created before applying the configuration

```json
    output {
      elasticsearch {
        data_stream => false
        ilm_enabled => "true"
        ilm_pattern => "{now/d}-000001"
        ilm_rollover_alias => "logs-test"

        ilm_policy => "custom-ilm"
        template_name => "logs-test"
        action => "create"
      }
    }
```

```bash
helmfile -l name=logstash -l ilm=true apply
```

## Logstash >= 8.8 with data stream

Currently data streams lifecycle is a technology preview feature it [provides settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-stream-lifecycle-settings.html) for data retention, rollover and tail merging. It doesn't seem there's shard shrinking option for data stream lifecycle, however data streams can also use [ILM policies](https://www.elastic.co/guide/en/elasticsearch/reference/current/set-up-a-data-stream.html#create-index-lifecycle-policy).

If we don't plan a full HOT-WORM-COLD ILM process with shrinking as well, we should be fine just with the current data stream lifecycle settings.

Before moving on:
  * We should [create and index template for a data stream](https://www.elastic.co/guide/en/elasticsearch/reference/current/tutorial-manage-new-data-stream.html#create-index-template-with-lifecycle). Rollover and retention can be controlled which you can look up in the data stream lifecycle settings provided in the link above. Rollover has already have optimal defaults, so you can set just the retention.

**Note:** Use a priority higher than 200 to avoid collisions with built-in templates. See Avoid index pattern collisions!!!

```yaml
# PUT _index_template/logs-servers-audit
{
  "index_patterns": ["logs-servers-audit-*"],
  "data_stream": { },
  "priority": 200,
  "template": {
    "settings": {
      "index": { "mapping": { "total_fields": { "limit": "1000" } } }
    },
    "lifecycle": {
      "data_retention": "62d"
    }
  },
  "_meta": {
    "description": "Index template for servers audit logs"
  }
}
```

Now we can point logstash to the desired data stream:

```json
    output {
      elasticsearch {
        data_stream => true
        data_stream_type => "logs"
        data_stream_dataset => "servers"
        data_stream_namespace => "audit"
        action => "create"
      }
    }
```

```bash
helmfile -l name=logstash -l dsl=true apply
```
