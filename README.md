# Knative Eventing - Kafka Demo

## Clone this repo

```shell
git clone https://github.com/kameshsampath/kn-kafka-demo
cd kn-kafka-demo
export PROJECT_HOME=`pwd`
```

## Setup Kafka

```shell
cat strimzi/strimzi-cluster-operator-0.14.0.yaml \
  | sed 's/namespace: .*/namespace: kafka/' \
  | kubectl apply -f - -n kafka
# using ephemeral kafka
kubectl apply -f strimzi/kafka-persistent-single.yml -n kafka
```

## Setup Knative Serving and Istio

Please follow bit.ly/knative-tutorial to have your minikube and Knative serving/eventing

## Setup Knative Eventing and  Eventing Sources

```shell
kubectl apply -f https://github.com/knative/eventing-contrib/releases/download/v0.8.2/kafka-importer.yaml

curl -L https://github.com/knative/eventing-contrib/releases/download/v0.8.2/kafka-channel.yaml \
  | sed 's/ bootstrapServers: REPLACE_WITH_CLUSTER_URL/  bootstrapServers: my-cluster-kafka-external-bootstrap.kafka:9094/' \
  | kubectl apply -f -

```

Watch the status of the eventing resources `kubectl get -n knative-eventing pods -w`


Set the default channel to be  `KafkaChannel`

```shell
cat <<-EOF | kubectl apply -f -
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: default-ch-webhook
  namespace: knative-eventing
data:
  default-ch-config: |
    clusterDefault:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: InMemoryChannel
    namespaceDefaults:
      knativetutorial:
        apiVersion: messaging.knative.dev/v1alpha1
        kind: KafkaChannel
        spec:
          numPartitions: 1
          replicationFactor: 1
EOF
```

```shell
kubectl label namespace knativetutorial knative-eventing-injection=enabled
```

## Services

Make sure you have your docker env setup

```shell
eval(minikube docker-env)
```
### Stream Greeter

A microservice to send messages to Kafka topic called `greetings`

```shell
git clone https://github.com/kameshsampath/stream-greeter &&\
cd stream-greeter &&\
./buildAndDeploy.sh
```

### Lingua Greeter

The Knative Eventing sink that will subscribe to `greetings-channel` and performs translation of the incoming message

```shell
git clone https://github.com/kameshsampath/kn-lingua-greeter &&\
cd kn-lingua-greeter &&\
kubectl create secret generic -n knativetutorial google-cloud-creds --from-file=google-cloud-credentials.json=<YOUR GOOGLE API CREDENTIALS> &&\
./buildAndDeploy.sh
```

NOTE: It will take few minutes for the Knative service to be available

## Knative

- Create channels

```shell
kubectl create -f $PROJECT_HOME/knative/01-channels.yaml
```

A successful channel creation should show an output like:

```shell
NAME                   READY   REASON   URL                                                                        AGE
greetings-channel      True             http://greetings-channel-kn-channel.knativetutorial.svc.cluster.local      10s
translated-greetings   True             http://translated-greetings-kn-channel.knativetutorial.svc.cluster.local   10s
```

- Create source

```shell
kubectl create -f $PROJECT_HOME/knative/02-greetings-source.yaml
```

Checking the Sink URI of the source using the command below should match to the `greetings-channel` URL (check the output in previous section)

```shell
kubectl get kafkasources.sources.eventing.knative.dev kafka-source -o yaml | yq r - status.sinkUri
```

- Create Subscription

```shell
kubectl create -f $PROJECT_HOME/knative/03-subscription.yaml
```

Check the status of the subscription using the command:

```shell
kubectl get subscriptions.eventing.knative.dev lingua-greeter-sub
```

A successfully created subscription should show an output like:

```shell
NAME                 READY   REASON   AGE
lingua-greeter-sub   True             10s
```

## Send and Receceive messages

To send messages fire the following command:

```shell
cd $PROJECT_HOME
./throttleUp.sh
```

To stop send messages fire the following command:

```shell
cd $PROJECT_HOME
./throttleDown.sh
```

## Useful command

Get messages from a topic

```shell
kubectl -n mykafka run kafka-greetings-consumer -ti --image=strimzi/kafka:0.14.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-external-bootstrap:9094 --topic knative-messaging-kafka.knativetutorial.translated-greetings --from-beginning
```

knative-messaging-kafka.translated-greetings

Get list of topics

```shell
kubectl -n knativetutorial run kafka-topics-list -ti --image=strimzi/kafka:0.14.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-topics.sh --list --bootstrap-server my-cluster-kafka-external-bootstrap.kafka:9094
```
