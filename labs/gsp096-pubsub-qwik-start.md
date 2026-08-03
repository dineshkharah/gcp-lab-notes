# GSP096, Pub/Sub qwik start, console

The smallest of these. Create a topic, add a pull subscription, publish a message, pull it back. Eight minutes, or about one minute from the shell.

## The whole lab from the shell

```
gcloud pubsub topics create MyTopic
gcloud pubsub subscriptions create MySub --topic=MyTopic
gcloud pubsub topics publish MyTopic --message="Hello World"
gcloud pubsub subscriptions pull --auto-ack MySub
```

`subscriptions create` defaults to pull delivery, which is what the lab requires.

One difference worth knowing. The console Create topic dialog also auto creates `MyTopic-sub` because that checkbox is ticked by default. Doing it from the shell you get only `MySub`, which is cleaner: it means the pull is guaranteed to receive the message rather than racing a second subscription for it.

If a pull returns nothing, run it again. A pull only returns what is buffered at that moment.

## Quiz

- A publisher sends messages to a ___, subscribers create a ___ to receive them: **topic, subscription**
- Pub/Sub is an asynchronous messaging service designed to be highly reliable and scalable: **True**
