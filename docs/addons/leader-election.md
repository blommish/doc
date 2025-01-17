# Leader Election

With leader election it is possible to have one responsible pod. This can be used to control that only one pod runs a batch-job or similar tasks. This is done by asking the `elector` pod which pod is the current leader, and comparing that to the pod's hostname. (onprem it is at least one)

The leader election configuration does not control which pod the external service requests will be routed to.

## Enable leader election

Enabling leader election in a pod is done by adding the line `leaderElection: true` to your `nais.yaml`-file. With that setting enabled, NAIS will sidecar an elector container into your pod.

When you have the `elector` container running in your pod, you can run a HTTP GET on the URL set in environment variable `$ELECTOR_PATH` to see which pod is the leader. This will return a JSON object with the name of the leader, which you can now compare with your hostname.

## Examples

### Code example

```java
// Implementation of getJSONFromUrl is left as an exercise for the reader
public boolean isLeader() {
    String electorPath = System.getenv("ELECTOR_PATH");
    JSONObject leaderJson = getJSONFromUrl(electorPath);
    String leader = leaderJson.getString("name");
    String hostname = InetAddress.getLocalHost().getHostname();

    return hostname.equals(leader);
}
```

### cURL example

```bash
$ kubectl exec -it elector-sidecar-755b7c5795-7k2qn -c debug bash
root@elector-sidecar-755b7c5795-7k2qn:/# curl $ELECTOR_PATH
{"name":"elector-sidecar-755b7c5795-2kcm7"}
```

## Issues

* This is not really an issue with NAIS but _Kubernetes' `leader-election`-image does not support fencing, which means it does not guarantee that there is only one leader_.

* Redeployment of the deployment/pods will in some cases make the non-leaders believe that the old leader still exists and is the leader.
  The current leader is not affected, and will be aware that the pod itself is the leader.
  This is not really an issue as long as you do not need to know exactly which pod is the leader.
  This can be resolved by deleting pods with erroneous leader election configurations.
