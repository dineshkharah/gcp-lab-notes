# GSP095, Pub/Sub: Qwik Start Command Line

Four tasks, **two** checkpoints, ten minutes stated and **no cost**. About **two minutes** for the scored part. **Pure Cloud Shell**, and no lab values are needed at all: nothing in it is project, region or name specific.

Manual and last tested both **March 2025**, second oldest test date in these notes. Nothing broke because of it.

The command line twin of `gsp096-pubsub-qwik-start.md`, which is the console version and smaller still.

## No ordering trap, which is worth saying out loud

The two checkpoints are `myTopic` and `mySubscription`. The lab deletes things in both task 1 and task 2, but **only ever `Test1` and `Test2`**, never the two resources being scored.

Recorded because the run of storage and Cloud Run labs around this one all had a destructive step that ate an earlier checkpoint, and it would be easy to start splitting blocks out of habit. Here tasks 1 and 2 go in a single paste.

## Block 1, tasks 1 and 2, claims both checkpoints

```
gcloud pubsub topics create myTopic
gcloud pubsub topics create Test1
gcloud pubsub topics create Test2
gcloud pubsub topics list --format="value(name)"
gcloud pubsub topics delete Test1
gcloud pubsub topics delete Test2
gcloud pubsub topics list --format="value(name)"
gcloud pubsub subscriptions create --topic myTopic mySubscription
gcloud pubsub subscriptions create --topic myTopic Test1
gcloud pubsub subscriptions create --topic myTopic Test2
gcloud pubsub topics list-subscriptions myTopic
gcloud pubsub subscriptions delete Test1
gcloud pubsub subscriptions delete Test2
gcloud pubsub topics list-subscriptions myTopic
```

Ran clean, both checkpoints green. Three topics down to one, three subscriptions down to one.

`--format="value(name)"` in place of the lab's bare `list`, which prints a `messageStoragePolicy` block per topic and makes three topics fill a screen.

**`Test1` and `Test2` are used as both a topic name and a subscription name** at different points in the lab. That is legal, they are different resource types in different namespaces, and it is the only mildly confusing thing here. Note also the capital `T`; these names are case sensitive.

## Tasks 3 and 4 are unscored, and they are the actual lesson

Written out and not run on this pass, since nothing after checkpoint 2 is scored. Kept because the semantics are the useful part of the lab.

```
gcloud pubsub topics publish myTopic --message "Hello"
gcloud pubsub topics publish myTopic --message "Publisher's name is NAME"
gcloud pubsub topics publish myTopic --message "Publisher likes to eat FOOD"
gcloud pubsub topics publish myTopic --message "Publisher thinks Pub/Sub is awesome"
sleep 10
gcloud pubsub subscriptions pull mySubscription --auto-ack
```

Run that last line five times. **Four messages, and the first four pulls each return exactly one, with the fifth returning `Listed 0 items.`** Two behaviours combine to produce that:

- a bare `pull` returns **one** message regardless of how many are waiting
- `--auto-ack` acknowledges it, so it is never delivered again

The lab's placeholders are `<YOUR NAME>` and `<FOOD>`. They sit inside double quotes, so the angle brackets are literal rather than redirections and the apostrophe in `Publisher's` is safe, but substitute them anyway.

The `sleep 10` is not in the lab. Publishing and pulling back to back can return nothing because the message has not landed yet, and that empty result is indistinguishable from the deliberate empty result at the end of the sequence. The lab half acknowledges this with a *"Wait a minute to let the topics get created"* line placed oddly in task 4.

```
gcloud pubsub topics publish myTopic --message "Publisher is starting to get the hang of Pub/Sub"
gcloud pubsub topics publish myTopic --message "Publisher wonders if all messages will be pulled"
gcloud pubsub topics publish myTopic --message "Publisher will have to test to find out"
sleep 10
gcloud pubsub subscriptions pull mySubscription --limit=3
gcloud pubsub subscriptions pull mySubscription --limit=3 --auto-ack
```

All three arrive in one table from the first pull.

**The lab ends on a command with no `--auto-ack` and does not mention it.** Its final `pull --limit=3` reads the three messages without acknowledging them, so Pub/Sub redelivers them once the ack deadline passes. The second line above is the one that actually clears them. This is the mechanism behind a subscription that appears to keep handing back the same messages, and the lab leaves the subscription in exactly that state.

Quiz answer: **True**.

## Related files

- `gsp096-pubsub-qwik-start.md`, the console version of the same four operations.
- `gsp080-cloud-run-functions-command-line.md` for a topic as a function trigger, including a publish that succeeds against the wrong topic name and never triggers anything.
- `stream-processing-pubsub-to-dataflow.md` and `streaming-analytics-into-bigquery.md` for Pub/Sub as a pipeline source.
