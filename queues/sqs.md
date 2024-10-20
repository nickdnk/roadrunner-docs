# SQS Driver

[Amazon Simple Queue Service (SQS)](https://aws.amazon.com/sqs/) is an alternative
queue server developed by Amazon, provided as part of the AWS Cloud infrastructure. You can find the documentation
[here](https://docs.aws.amazon.com/sqs/).

While this service is usually deployed on AWS, you can also self-host any compatible version, such as the
community-developed [ElasticMQ](https://github.com/softwaremill/elasticmq).

## Configuration

Specify the following connection settings in the `sqs` configuration settings.

{% code title=".rr.yaml" %}

```yaml
sqs:
  # AWS AccessKey ID.
  # Default: empty
  key: access-key

  # AWS Access Key Secret.
  # Default: empty
  secret: api-secret

  # AWS region.
  # Default: empty
  region: us-west-1

  # AWS session token.
  # Default: empty (optional; local testing only)
  session_token: test

  # AWS SQS endpoint.
  # This parameter *is only required* if your SQS endpoint is *not* hosted on the AWS infrastructure.
  # Always leave this empty if your queue is on AWS SQS.
  # For ElasticMQ, the default is http://127.0.0.1:9324
  # Default: empty
  endpoint: ""
```
{% endcode %}

### Examples

Because the options can be mixed in ways that don't make sense, here are a few examples of how to use the driver
in various scenarios. In all examples, the **queue name** is provided in the `jobs.pipelines.pipeline_name.config.queue`
property below.

#### AWS SQS with dynamic IAM in same region

If you access your AWS SQS queue from inside AWS via a service that supports dynamic IAM roles (such as EC2), and the
queue is in the same region.

{% code title=".rr.yaml" %}
```yaml
sqs:
  # No parameters are required. Normally, if your IAM role has the required permissions and your service endpoint
  # is in the same region, the environment variables associated with your instance/task will resolve all the 
  # configuration for you. You must provide the parent sqs key to enable SQS, even if you want to use the defaults.
```
{% endcode %}

#### AWS SQS in different region with dynamic IAM

If you access your AWS SQS queue from inside AWS via a service that supports dynamic IAM roles (such as EC2), and the
queue is in a different region.

{% code title=".rr.yaml" %}
```yaml
sqs:
  # Only region is required
  region: eu-central-1
```
{% endcode %}

#### AWS SQS without dynamic IAM

If you access your AWS SQS queue from outside AWS, or from a service that does not support IAM, you must provide
an access key and secret. If you access the service from outside AWS, you must also provide `region`.
You may also use this option if your service *does* support dynamic IAM, but you want to use explicit access keys.

{% code title=".rr.yaml" %}
```yaml
sqs:
  key: ASIAIOSFODNN7EXAMPLE
  secret: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  session_token: AQoDYXdzEJr... # optional; usually not required
  # If in a different region *or* if outside AWS, provide the correct region.
  region: eu-west-1 
```
{% endcode %}

#### Self-hosted (ElasticMQ)

If you self-host your SQS-compatible queue, only the endpoint is required.

{% code title=".rr.yaml" %}
```yaml
sqs:
  endpoint: http://127.0.0.1:9324
```
{% endcode %}

After you have configured the driver, you should configure the queue that will use this connection.

{% code title=".rr.yaml" %}

```yaml
version: "3"

sqs:
# SQS connection configuration. See above example.

jobs:
  pipelines:
    test-sqs-pipeline:
      # Required section.
      # Should be "sqs" for the Amazon SQS driver.
      driver: sqs

      config:
        # Optional section.
        # Default: 10
        prefetch: 10

        # Consume any payload type (not only Jobs structured)
        # Default: false
        consume_all: false

        # Get queue URL only
        # Default: false
        skip_queue_declaration: false

        # Optional section.
        # Default: 0
        visibility_timeout: 0

        # Optional section.
        # Default: 0
        wait_time_seconds: 0

        # Optional section.
        # Default: default
        queue: default

        # Message group ID: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html#SQS-SendMessage-request-MessageGroupId
        # Default: empty, should be set if FIFO queue is used
        message_group_id: "test"

        # Optional section.
        # Default: empty
        attributes:
          DelaySeconds: 42
          # etc... see https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html

        # Optional section.
        # Default: empty
        tags:
          test: "tag"
```

{% endcode %}

## Configuration Options

### Prefetch

`prefetch` - Number of jobs to prefetch from the queue until ACK/NACK.

Default: `10`

### Visibility Timeout

`visibility_timeout` - The duration (in seconds) that the received messages are hidden from subsequent retrieve requests
after being retrieved by a `ReceiveMessage` request. Max value is `43200` seconds (12 hours).

Default: `0`

### Wait Time

`wait_time_seconds` - The duration (in seconds) for which the call waits for a message to arrive in the queue before
returning. If a message is available, the call returns sooner than `wait_time_seconds`. If no messages are available and
the wait time expires, the call returns successfully with an empty list of messages.

Default: `5`

### Queue

`queue` - The name of the queue. May only contain alphanumeric characters, hyphens (`-`), and underscores (`_`). On AWS,
this will form the last part of the queue URL, i.e. `https://sqs.<region>.amazonaws.com/<account id>/<queue>`.

Default: `default`

### Message Group ID

`message_group_id` - Message group ID is required for FIFO queues. Messages that belong to the same message group are
processed in a FIFO manner.
More info: [link](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html#SQS-SendMessage-request-MessageGroupId)

### Skip Queue Declaration

`skip_queue_declaration` - By default, RR tries to create the queue (using the `queue` name) if it does not exist. Set
this option to `true` if the queue already exists.

Default: `false`

{% hint style="info" %}
For queue creation to work, the credentials or the IAM role used must have the
[required permissions](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-api-permissions-reference.html),
such as `sqs:CreateQueue` and `sqs:SetQueueAttributes`.
{% endhint %}

### Consume All

`consume_all` - By default, RR consumes only messages in the queue with the `Jobs` structure. Set this option to `true`
if you want to consume all messages in the queue.

Default: `false`

### Attributes

`attributes` - A list of
[AWS SQS attributes](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html)
to configure for the queue. Attributes are only set if RR creates the queue. Existing queues are not modified.

{% code title=".rr.yaml" %}

```yaml
attributes:
  DelaySeconds: 0
  MaximumMessageSize: 262144
  MessageRetentionPeriod: 345600
  ReceiveMessageWaitTimeSeconds: 0
  VisibilityTimeout: 30
```

{% endcode %}

### Tags

`tags` - Tags don't have any semantic meaning. Amazon SQS interprets tags as character. Tags are only set if RR creates
the queue. Existing queues are not modified.

{% hint style="info" %}
This functionality is rarely used and slows down queues. Please see
[this link](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-tags.html).
{% endhint %}
