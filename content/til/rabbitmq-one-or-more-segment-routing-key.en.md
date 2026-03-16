+++
date = '2026-03-15T20:54:37+01:00'
draft = true
title = 'How to create a binding with a routing key that has at least one segment in RabbitMQ'
+++

Recently, I discovered a bug in a system I'm developing that uses RabbitMQ for message exchange.

We have a `topic`-type exchange named `distribution` and a queue named `foo.test.message`. There is also a binding connecting the exchange and the queue with the following configuration:

```yaml
source: distribution
destinationType: queue
destination: foo.test.message
routingKey: foo.test.*.message
```

*Note: This is an excerpt from the configuration of a `Binding` CRD from `rabbitmq.com/v1beta1`.*

Several applications in the system publish messages to a `topic`-type `distribution` exchange with a routing key dependent on a business value within the message. The routing key is constructed on the fly when the message is posted.

A change was made to the format of the business value, adding a segment to it. During integration testing, we noticed that messages with the new format were no longer routed correctly.

I hadn’t anticipated any impact from the change in the business value’s format because, in my understanding of routing keys at the binding level, `*` corresponded to a full wildcard, allowing any characters.

That’s actually not the case at all. In RabbitMQ, routing keys are built from segments, consisting of an alphanumeric string separated with a `.`. For example, the routing key `foo.test.hello.message` is composed of four segments. There are also two special symbols that can be used:

- `*` which means “exactly one segment”
- `#` which means “zero or any number of segments”

In my case, to make the new format work, I needed to allow “one or more segments” instead of “exactly one segment.” But this is apparently not possible with the capabilities offered by the routing key format in RabbitMQ.

After a good 10 minutes of cursing, I finally found the solution, which had been right in front of me the whole time.

You can combine the two special symbols. Thus, “one or more segments” can be written as `*.#`, which in my case gives `foo.test.*.#.message`.

The method can be extended to the general case of “n segments or more” with n a natural number, which gives `*.(<*. n-2 times>).*.#`, although I honestly find this notation very hard to read (and pretty messy).
