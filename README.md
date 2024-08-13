## README

### produce STRING key JSON value - no schema, no headers

```
POST /clusters/{cluster_id}/topics/{topic_name}/records HTTP/1.1
Host: example.com
Content-Type: application/json
```
### caveat

The schema IDs and versions will probably be different from the ones used in this example. Please adjust your requests accordingly   

### V3

https://docs.confluent.io/platform/current/kafka-rest/api.html#records-v3

`http POST :8082/v3/clusters/4L6g3nShT-eMCtK--X86sw/topics/test-topic/records < avro_with_schema_v3.json`

you will first need to find out the cluster ID by sending a GET to `v3/clusters`  

#### request

```json
{
  "partition_id": 0,
  "key": {
      "type": "STRING",
      "data": "someKey"
  },
  "value": {
      "type": "AVRO",
      "subject": "test-topic-value",
      "schema": "{\"type\":\"record\",\"name\":\"SimpleRecord\",\"namespace\":\"com.example.avro\",\"fields\":[{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"}]}",
      "data": {"name":"John Doe","age":30}
  },
  "timestamp": "2024-08-12T19:14:42Z"
}
```


#### response - OK

```json
{
    "cluster_id": "4L6g3nShT-eMCtK--X86sw",
    "error_code": 200,
    "key": {
        "size": 7,
        "type": "STRING"
    },
    "offset": 0,
    "partition_id": 0,
    "timestamp": "2024-08-12T19:14:42Z",
    "topic_name": "test-topic",
    "value": {
        "schema_id": 2,
        "schema_version": 1,
        "size": 15,
        "subject": "test-topic-value",
        "type": "AVRO"
    }
}
```

#### request without a schema - use latest version

Just drop the schema line from the request. Also notice to drop `"type": "AVRO"`

request: 

response 

```json
{
    "cluster_id": "4L6g3nShT-eMCtK--X86sw",
    "error_code": 200,
    "key": {
        "size": 7,
        "type": "STRING"
    },
    "offset": 1,
    "partition_id": 0,
    "timestamp": "2024-08-12T19:14:42Z",
    "topic_name": "test-topic",
    "value": {
        "schema_id": 2,
        "schema_version": 1,
        "size": 15,
        "subject": "test-topic-value",
        "type": "AVRO"
    }
}
```

#### request without a schema - use explicit schema version

request - notice we dropped `"type": "AVRO"`

```json
{
  "partition_id": 0,
  "key": {
      "type": "STRING",
      "data": "someKey"
  },
  "value": {
      "subject": "test-topic-value",
      "schema_version": 1,
      "data": {"name":"John Doe","age":30}
  },
  "timestamp": "2024-08-12T19:14:42Z"
}
```

response

```json
{
    "cluster_id": "4L6g3nShT-eMCtK--X86sw",
    "error_code": 200,
    "key": {
        "size": 7,
        "type": "STRING"
    },
    "offset": 2,
    "partition_id": 0,
    "timestamp": "2024-08-12T19:14:42Z",
    "topic_name": "test-topic",
    "value": {
        "schema_id": 2,
        "schema_version": 1,
        "size": 15,
        "subject": "test-topic-value",
        "type": "AVRO"
    }
}
```

#### request without a schema AND subject - use explicit schema ID - breaks

same type of error. Adding or removing  `"type": "AVRO"`has no effect.

```json
{
  "partition_id": 0,
  "headers": [
      {
          "name": "Header-1",
          "value": "SGVhZGVyLTE="
      },
      {
          "name": "Header-2",
          "value": "SGVhZGVyLTI="
      }
  ],
  "key": {
      "type": "STRING",
      "data": "someKey"
  },
  "value": {
      "subject": "test-topic-value",
      "type": "AVRO",
      "data": {"name":"John Doe","age":30}
  },
  "timestamp": "2024-08-12T19:14:42Z"
}
```

```json
{
    "error_code": 400,
    "message": "Cannot construct instance of `io.confluent.kafkarest.entities.v3.ProduceRequest$ProduceRequestData`, problem: 'schema_version=latest' cannot be used with 'serializer'."
}
```

#### !!! CAVEAT - DO NOT SPECIFY `"type": "AVRO"` without an explicit schema - using the latest or an explicit schema ID

The request will fail with an extremely unhelpful message: 

```json
{
    "error_code": 400,
    "message": "Cannot construct instance of `io.confluent.kafkarest.entities.v3.ProduceRequest$ProduceRequestData`, problem: 'schema_version=1' cannot be used with 'serializer'."
```

### wrong schema ID

When data does not match the schema:

```json
{
    "error_code": 400,
    "message": "Bad Request: Expected field name not found: name"
}
```

### trying to produce with incompatible schema

```json
{
    "error_code": 422,
    "message": "Error: 42207 : Error serializing message. Error when registering schema. format = AVRO, subject = test-topic-value, schema = {\"type\":\"record\",\"name\":\"SimpleRecord\",\"namespace\":\"com.example.avro\",\"fields\":[{\"name\":\"blip\",\"type\":\"string\"},{\"name\":\"blop\",\"type\":\"int\"}]}"
}
```

### producing with incompatible schema - using a specific subject

You can use any subject the RP principal has access to. While still producing to your topic of interest.

```json
  "value": {
      "type": "AVRO",
      "subject": "some-other-value",
      "schema": "{\"type\":\"record\",\"name\":\"SimpleRecord\",\"namespace\":\"com.example.avro\",\"fields\":[{\"name\":\"blip\",\"type\":\"string\"},{\"name\":\"blop\",\"type\":\"int\"}]}",
      "data": {"blip":"John Doe","blop":30}
  },
```

The schema will be registered and you can avoid the incompatibility issues. 

### `auto.register.schemas`does not prevent compatible schemas from being registered

tried it. I do not get a warning, as suggested in [[2023.04.06 Rest Proxy produce v3 ignores `auto.regster.schemas` and `use.latest. version`]]

But a compatible schema does register

`KAFKA_REST_AUTO_REGISTER_SCHEMAS: false`

and you can see that the setting is there in the client

```
[2024-08-13 06:06:48,469] INFO SchemaRegistryConfig values:
	auto.register.schemas = false
...

[2024-08-13 06:22:25,457] INFO KafkaAvroSerializerConfig values:
	auto.register.schemas = false
```

### V2

`POST` to `/topics` or `/partitions`: https://docs.confluent.io/platform/current/kafka-rest/api.html#topics

#### example request - with schema inlined

`http POST :8082/topics/rp_avro Content-Type:application/vnd.kafka.avro.v2+json < avro_with_schema_v2.json`

```json
{
  "value_schema": "{\"name\":\"int\",\"type\": \"int\"}",
  "records": [
    {
      "value": 12
    },
    {
      "value": 24,
      "partition": 1
    }
  ]
}
```

#### response - with new schema ID

```json
{
    "key_schema_id": null,
    "offsets": [
        {
            "error": null,
            "error_code": null,
            "offset": 0,
            "partition": 0
        },
        {
            "error": null,
            "error_code": null,
            "offset": 1,
            "partition": 0
        }
    ],
    "value_schema_id": 1
}
```

#### request with schema ID

`http POST :8082/topics/rp_avro Content-Type:application/vnd.kafka.avro.v2+json < avro_with_schema.id_v2.json`

response is the same as above, just with changed offset.

```json
{
    "value_schema_id": 1,
    "records": [
      {
        "value": 12
      },
      {
        "value": 24,
        "partition": 0
      }
    ]
  }
```

#### request with a non-existent schema ID - leads to error

`http POST :8082/topics/rp_avro Content-Type:application/vnd.kafka.avro.v2+json < avro_with_nonexisting_schema.id_v2.json`

```json
{
    "error_code": 42207,
    "message": "Error serializing message. Error when fetching schema by id. schemaId = 999\nSchema 999 not found; error code: 40403"
}






