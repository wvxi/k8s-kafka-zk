https://medium.com/@martin.hodges/deploying-kafka-on-a-kind-kubernetes-cluster-for-development-and-testing-purposes-ed7adefe03cb

Deploying Kafka on a Kind Kubernetes cluster for development and testing purposes
Many event-based solutions are based on Kafka. It is highly scalable, reliable and performant. I previously wrote about how you can use Kind to create a Kubernetes cluster for development and testing. This article adds a Kafka cluster to a Kind cluster.
Martin Hodges
Martin Hodges

·
Follow

7 min read
·
Apr 2, 2024
112


4




Kafka on Kind cluster
Kafka
Kafka is described using a lot of different terms. I like to call it an event queue, whereby each event is described by way of a message.

The message is placed on the queue by one or more producers and read from the queue by one or more consumers. The messages can be read, sequentially, from the queue at any time in the future.

There are a few caveats.

Messages may appear out of expected sequence.
An application may read a message more than once.
Messages are available up until the point they are purged (a configurable time, defaulting to 1 week)
You can read more about Kafka in my article describing topics, partitions and brokers.

What we will create
In this article we will deploy a Kafka cluster on a Kind Kubernetes cluster. For development and test purposes, you can use Kind to deploy a multi-node Kubernetes cluster locally on your development machine.

The diagram above shows what we will create:

A 4 node Kind cluster (1 master, 3 workers)
A 3 broker Kafka cluster deployed to the Kind cluster
A NodePort service to expose the Kafka cluster to the host
Once created, we will then use command line tools to try out our new Kafka cluster.

I use a Mac for my development and so the instructions you see in my articles are for macOS.

Creating the Kind cluster
I am assuming you have Kind deployed on your machine. If not, you can find installation instructions for most operating systems here.

To create a multi-node Kubernetes cluster, you will require a Kind configuration file. Create the following:

kind-config.yml

apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30092
    hostPort: 30092
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
- role: worker
- role: worker
- role: worker
This will create one control-plane node (master) and 3 worker nodes.

I have selected 3 worker nodes so that each of our Kafka brokers can sit on a different node.

You can also see that we expose the network port 30092 on our host machine. This is because Kind implements its Kubernetes nodes as Docker containers and we need to expose any nodePort services to our local machine. In this case, it is port 30092.

We can now start up our Kubernetes cluster with:

kind create cluster --config kind-config --name my-cluster
The name is optional. If you are only using one cluster, it is easier to leave it off. In this article, I will assume you have not used a name.

It takes a minute or two to create the cluster. Once up and running you can confirm the 4 nodes are up and Kubernetes is running with:

kubectl get nodes
kubectl get pods -A
Creating the Kafka cluster
Now we have a Kubernetes cluster, we can now install Kafka.

We will install a Kafka KRaft installation which does not need to deploy ZooKeeper and, instead, uses a Kafka version of the Raft protocol (KRaft).

Although there are Helm charts available, I am going to show how to install using manifest files so you can wee what it takes below the surface.

Namespace
First we will create the Kafka namespace:

kubectl create namespace kafka
Deployment
Now we will create the deployment file:

kafka-deployment.yml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
  labels:
    app: kafka-app
spec:
  serviceName: kafka-svc
  replicas: 3
  selector:
    matchLabels:
      app: kafka-app
  template:
    metadata:
      labels:
        app: kafka-app
    spec:
      containers:
        - name: kafka-container
          image: doughgle/kafka-kraft
          ports:
            - containerPort: 9092
            - containerPort: 9093
          env:
            - name: REPLICAS
              value: '3'
            - name: SERVICE
              value: kafka-svc
            - name: NAMESPACE
              value: kafka
            - name: SHARE_DIR
              value: /mnt/kafka
            - name: CLUSTER_ID
              value: bXktY2x1c3Rlci0xMjM0NQ==
            - name: DEFAULT_REPLICATION_FACTOR
              value: '3'
            - name: DEFAULT_MIN_INSYNC_REPLICAS
              value: '2'
          volumeMounts:
            - name: data
              mountPath: /mnt/kafka
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: "1Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-svc
  namespace: kafka
  labels:
    app: kafka-app
spec:
  type: NodePort
  ports:
    - name: '9092'
      port: 9092
      protocol: TCP
      targetPort: 9092
      nodePort: 30092
  selector:
    app: kafka-app
There are a few things to note here:

The use of a StatefulSet to fix each replica’s network identity and to use a consistent Persistent Volume Claim (PVC) so if a broker fails, it can restart transparently
We are using a docker image from doughgle/kafka-kraft
The use of the kafka namespace
A replicas value of 3, representing 3 Kafka brokers
The exposing of two ports, 9092 (for clients) and 9093 (for the control plane)
A 16 character cluster ID (eg: my-cluster-12345), base 64 encoded (use your own value)
A default replication factor of 3, so each partition appear in each broker
A minimum in sync replica value of 2, so producers only have to wait for two copies of a message to be written rather than all three (known as In Sync Replicas or ISRs)
The request for a volume that will hold the commit log for the broker
On Kind, the volume Persistent Volume Claim (PVC) and Persistent Volume (PV) are automatically created.

Service
We need a service to expose the Kafka cluster to clients in the Kubernetes cluster and also to clients outside the cluster.

To do this, we use a NodePort service and add its definition as a separate document within the deployment manifest. This provides a service within the cluster and also exposes it on port 30092 of every node.

By running Kafka on Kubernetes, this service gives us an important feature. As the service automatically load balances across the Kafka brokers, we only need to use its DNS name to bootstrap our clients, ie: kavka-svc (when accessing in the same namespace) or, if a fully qualified domain name is required: kafka-svc.kafka.svc.cluster.local.

Deploying the cluster
We can now deploy our cluster and the service with:

kubectl apply -f kafka-deployment.yml
It may take a little while to start but you can check it has started with:

kubectl get pods -n kafka
You should see all 3 brokers up and running:

NAME      READY   STATUS    RESTARTS   AGE
kafka-0   1/1     Running   0          13s
kafka-1   1/1     Running   0          8s
kafka-2   1/1     Running   0          4s
I have noted that the Kafka cluster does not always come up cleanly. To check this, first check the logs from the 3 brokers:

kubectl logs kafka-0 -n kafka | grep STARTED
kubectl logs kafka-1 -n kafka | grep STARTED
kubectl logs kafka-2 -n kafka | grep STARTED
If any of them show no logs, you may need to restart them with:

kubectl delete pod <pod name> -n kafka
Check the logs again after it restarts.

You should see a line in the offending broker saying:

The broker has been unfenced. Transitioning from RECOVERY to RUNNING.
Congratulations, you now have a Kafka cluster up and running in Kubernetes.

Trying it out
To try it out, we will get a command line in one of the brokers with:

kubectl exec -it kafka-0 -n kafka -- bash
This will now give you access to Kaka command line tools that have been loaded with the Kafka image we used.

First let’s create a topic, called my-topic. We will then list topics to check it is there:

kafka-topics.sh --create --topic my-topic --bootstrap-server kafka-svc:9092
You can check it was created with:

kafka-topics.sh --list --topic my-topic --bootstrap-server kafka-svc:9092
Now start a new command line terminal on your development machine and get a command line on one of the brokers:

kubectl exec -it kafka-1 -n kafka -- bash
On one of the brokers you have opened, start a console producer with:

kafka-console-producer.sh  --bootstrap-server kafka-svc:9092 --topic my-topic
On the other broker, start a console consumer with:

kafka-console-consumer.sh --bootstrap-server kafka-svc:9092 --topic my-topic
With these set up, you can start typing at the producer’s > prompt. When you press return, your message should sent and should appear at the consumer. You can send as many messages as you like.

Stop the consumer with control C. Send some more messages. Start the consumer again but this time add --from-beginning. The consumer will then display all the message from the start.

To clean up, you can use:

kafka-topics.sh --delete --topic my-topic --bootstrap-server kafka-svc:9092
Note that if you delete your Kafka cluster with kubectl delete -f kafka-deployment.yml and then recreate it, you will find your topics and messages still exist as the PV is held and reused by Kind due to the use of a StatefulSet.

Summary
This article has taken us through the installation of Kafka on a Kind Kubernetes cluster.

Following a quick recap of Kafka we:

Created a 4 node Kind Kubernetes cluster
Installed a 3 broker Kafka cluster on the Kubernetes cluster
Deployed a Kubernetes service for Kafka
Used the command line tools to test our cluster
We saw that we can produce messages that can then be read by a consumer using Kafka command line instructions. The cluster is now ready to use with your application.

If you found this article of interest, please give me a clap as that helps me identify what people find useful and what future articles I should write. If you have any suggestions, please add them in the comments section.
