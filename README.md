# amq-online-pdb-example

Demonstrates use of `maxUnavailable` to preserve availabilty of a partition queue during an OpenShift upgrade.

This example configures AMQ Online with `maxUnavailable` `1` for both brokers and routes.  When used with partition queues, this has the benefit of achieving
queue availability during an OpenShift Upgrade.  It does not achieve message availability.  Messages will be stranded on a broker whilst that node is upgrade,
as soon the broker pod comes back, the messages will be available again.

AMQ Online *prefers* that brokers are run on separate nodes (anti-affinity).  If there are more brokers than nodes, the kubernetes scheduler will have no choice
than to co-locate brokers.  This could result in partitions from the same queue being scheduled to the same node which may lead to a queue unavailability during upgrade.


## Prereqiistes

1.  https://github.com/rh-messaging/cli-rhea

## Preparation

1. Install AMQ Online 1.7
2. Clone this repo
```
git clone git@github.com:k-wall/amq-online-pdb-example.git
```
3. Apply the YAML
```
oc apply -f amq-online-pdb-example
```

## Demonstration

Let's demonstrate the the queue remains available during an OpenShift Upgrade.  To simulate an OpenShift Upgrade, we'll use `oc adm cordon/drain`.

1. After application of the YAML, the broker pods will look like this:

```
oc get pods --output wide
NAME                                        READY   STATUS    RESTARTS   AGE     IP             NODE                           NOMINATED NODE   READINESS GATES
address-space-controller-7b56f97f6f-ljcpl   1/1     Running   0          4h38m   10.128.8.8     ip-10-0-167-36.ec2.internal    <none>           <none>
admin.4676ff5-555f5d4f56-xqgvq              2/2     Running   0          4h24m   10.130.10.9    ip-10-0-189-154.ec2.internal   <none>           <none>
broker-4676ff5-6sj9-0                       1/1     Running   0          14m     10.129.10.10   ip-10-0-213-246.ec2.internal   <none>           <none>
broker-4676ff5-cthm-0                       1/1     Running   0          4h16m   10.131.10.9    ip-10-0-146-113.ec2.internal   <none>           <none>
console-5c58b5d698-rfs62                    2/2     Running   0          4h38m   10.129.8.9     ip-10-0-153-11.ec2.internal    <none>           <none>
enmasse-operator-7b76f56d88-tkqmp           1/1     Running   0          4h39m   10.130.2.10    ip-10-0-223-50.ec2.internal    <none>           <none>
example-authservice-679769df9f-v9sfs        1/1     Running   0          4h36m   10.128.4.11    ip-10-0-207-179.ec2.internal   <none>           <none>
qdrouterd-4676ff5-0                         1/1     Running   0          4h24m   10.131.4.8     ip-10-0-195-174.ec2.internal   <none>           <none>
qdrouterd-4676ff5-1                         1/1     Running   0          4h22m   10.130.6.10    ip-10-0-186-21.ec2.internal    <none>           <none>
```

2. Send some messages to the queue.

```
cli-rhea-sender --count 30 --msg-durable true --msg-content pepa --log-lib TRANSPORT_FRM --conn-ssl true --log-msgs json --broker guest:guest@$(oc get addressspace example-space -o 'jsonpath={.status.endpointStatuses[?(@.name=="messaging")].externalHost}'):443 --address example-partitioned-queue
```

3. Verify that all shards contain some messages.

```
oc exec broker-4676ff5-cthm-0 -- /opt/amq/bin/artemis  queue stat   --user $(oc get secret broker-support-4676ff5  --template='{{.data.username}}' | base64 --decode) --password $(oc get secret broker-support-4676ff5  --template='{{.data.password}}' | base64 --decode) | grep example
...
|example-partitioned-queue|example-partitioned-queue|1              |14            |58             |0                |44             |0               |ANYCAST      
```
+
Repeat for the other broker ensuring that the total message count across the two equals 30.

4. Uncordon and drain a node containing a broker
```
oc adm cordon ip-10-0-213-246.ec2.internal && oc adm drain ip-10-0-213-246.ec2.internal --delete-local-data --ignore-daemonsets
```

5. *Immediately* start a pod watch in one terminal and start a receiver another
```
oc get pods -w
```

```
cli-rhea-receiver --count 30  --log-lib TRANSPORT_FRM --conn-ssl true --log-msgs json --broker guest:guest@$(oc get addressspace example-space -o 'jsonpath={.status.endpointStatuses[?(@.name=="messaging")].externalHost}'):443 --address example-partitioned-queue
```

6. Notice that the receiver will receive the messages from the broker running on the uncordoned node but the rest won't appear until the other broker pod returns to service.

```
{"durable":true,"priority":4,"ttl":null,"first-acquirer":null,"delivery-count":0,"id":null,"user-id":null,"address":"example-partitioned-queue","subject":null,"reply-to":null,"correlation-id":null,"content-type":"string","content-encoding":null,"absolute-expiry-time":null,"creation-time":null,"group-id":null,"group-sequence":null,"reply-to-group-id":null,"properties":{},"content":"pepa"}
  rhea:frames [connection-1]:0 -> disposition#15 {"role":true,"last":13,"settled":true,"state":[]}  +1ms
  
     # ^^ messages from broker running on the uncordoned node received.
  
  rhea:frames [connection-1]:0 -> empty +4s
  rhea:frames [connection-1]:0 -> empty +4s
  rhea:frames [connection-1]:0 -> empty +4s
  rhea:frames [connection-1]:0 -> empty +4s
  
   # <--  Broker returns to service here (on alternative node)
 
  rhea:frames [connection-1]:0 <- transfer#14 {"delivery_id":14,"delivery_tag":{"type":"Buffer","data":[8,130,0,0,0,0,0,0]}} <Buffer 00 53 70 c0 07 05 41 40 40 40 52 01 00 53 73 d0 00 00 00 32 00 00 00 0d 40 40 a1 19 65 78 61 6d 70 6c 65 2d 70 61 72 74 69 74 69 6f 6e 65 64 2d 71 75 ... 41 more bytes> +192ms
{"durable":true,"priority":4,"ttl":null,"first-acquirer":null,"delivery-count":1,"id":null,"user-id":null,"address":"example-partitioned-queue","subject":null,"reply-to":null,"correlation-id":null,"content-type":"string","content-encoding":null,"absolute-expiry-time":null,"creation-time":null,"group-id":null,"group-sequence":null,"reply-to-group-id":null,"properties":{},"content":"pepa"}

   # ^^ messages from second broker now received
  
```

7. Uncordon the node
```
oc adm uncordon ip-10-0-213-246.ec2.internal 
```
