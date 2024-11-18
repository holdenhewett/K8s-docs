---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

Kubernetes lets you manage dynamic environments that you can change without altering your
application's code or image by using Redis. To do this, you must configure Redis using your {{<
glossary_tooltip text="ConfigMap" term_id="configmap">}}. For example, this enables you to
centralize the configuration for multiple environments or manage dynamic updates to Redis
configurations.

This tutorial teaches you how to configure Redis by creating and applying a ConfigMap that holds
Redis configuration values. You'll create a Redis {{< glossary_tooltip text="Pod" term_id="pod"
>}}, verify its configuration, update the ConfigMap with new settings, and restart the Pod to apply
the changes.

This page provides a real world example of how to configure Redis using a ConfigMap and builds upon
the [Configure a Pod to Use a
ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task. 



## What you'll learn

By following this guide, you'll learn to do the following:

* Create a ConfigMap with Redis configuration values.
* Create a Redis Pod that mounts and uses the created ConfigMap.
* Verify that the configuration was correctly applied.



## Requirements


You need to have the following:

* A Kubernetes cluster.
* The `kubectl` (version 1.14+) command-line tool configured to communicate with your cluster.
* You followed [Configure a Pod to Use a
  ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) to completion.

{{< note title="Don't have a Kubernetes cluster yet?" >}}
We recommend that you run this tutorial on a cluster with at least two nodes that are not
acting as control plane hosts. If you do not already have a cluster, you can create one by using
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) or you can use one of these
Kubernetes playgrounds:

* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [Play with Kubernetes](https://labs.play-with-k8s.com/)
{{< /note >}}


## Step 1: Create a ConfigMap

The following steps show you how to set up an empty ConfigMap to hold Redis configuration values.

1. Create a new ConfigMap, using the filename `example-redis-config.yaml`.
1. Add the following code to the file:

   {{< highlight shell "hl_lines=6" >}}
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: example-redis-config
   data:
     redis-config: ""
   {{< /highlight >}}

   Notice how the `redis-config` block is empty. You'll do more with this later in the tutorial.


## Step 2: Deploy the Redis Pod

To prepare and deploy the necessary resources for Redis to run in your Kubernetes cluster, you must
first deploy the ConfigMap to ensure the Redis Pod has access to its configuration at runtime. Then,
you can apply a Pod manifest to deploy the Redis Pod. You'll use the
[`redis-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml)
Pod manifest file for this tutorial.

1. Use the `kubectl` CLI command to apply the ConfigMap you just created
   (`example-redis-config.yaml`) to your Kubernetes cluster:

   ```shell
   kubectl apply -f example-redis-config.yaml
   ```
1. Use `kubectl` to apply the Pod manifest (`redis-pod.yaml`):

   ```shell
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
   ```
1. After the Redis Pod has deployed, verify the objects were created:

   ```shell
   kubectl get pod/redis configmap/example-redis-config 
   ```

   `kubectl` will output similar to the following:
   
   ```
   NAME        READY   STATUS    RESTARTS   AGE
   pod/redis   1/1     Running   0          8s
   
   NAME                             DATA   AGE
   configmap/example-redis-config   1      14s
   ```

   If you see `Running` in the `STATUS` column and the Pod is using the
   `configmap/example-redis-config` ConfigMap, you successfully created and deployed the Redis Pod!


## Step 3: Inspect the initial configuration

Examine the contents of the Redis pod manifest and note the following:

* A volume named `config` is defined in `spec.volumes` and populated with data from the
  `example-redis-config` ConfigMap.
  * Specifically, the `redis-config` key in the ConfigMap is exposed as a file named `redis.conf`
    within this volume.
* The config volume is mounted at `/redis-master` inside the container (via
  `spec.containers[0].volumeMounts`).
* This setup allows the file `/redis-master/redis.conf` inside the container to reflect the value of
  `data.redis-config` in the ConfigMap, enabling Redis to use this configuration.

{{< highlight yaml "hl_lines=23 28 32 33" >}}
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:5.0.4
    command:
      - redis-server
      - "/redis-master/redis.conf"
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf
{{< /highlight >}}

Recall that you left `redis-config` key in the `example-redis-config` ConfigMap empty. Follow these
steps to verify what this empty configuration looks like:

1. Check the details of the `exmaple-redis-config` ConfigMap:

   ```shell
   kubectl describe configmap/example-redis-config
   ```

   You should see an empty `redis-config` key:
   
   {{< highlight yaml "hl_lines=8" >}}
   Name:         example-redis-config
   Namespace:    default
   Labels:       <none>
   Annotations:  <none>
   
   Data
   ====
   redis-config:
   {{< /highlight >}}

   This confirms that the `redis-config` is indeed empty.

1. Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

   ```shell
   kubectl exec -it redis -- redis-cli
   ```

1. Check `maxmemory`:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory
   ```

   It should show the default value of 0:

   ```shell
   1) "maxmemory"
   2) "0"
   ```

1. Similarly, check `maxmemory-policy`:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory-policy
   ```

   Which should also yield its default value of `noeviction`:
   
   ```shell
   1) "maxmemory-policy"
   2) "noeviction"
   ```


## Step 4: Update the ConfigMap

To illustrate how updating a ConfigMap doesn't alter your application's code or image, update the
`example-redis-config` ConfigMap to the following:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

What these new configuration values mean:

* `maxmemory 2mb`: Limits Redis memory usage to 2 MB. This is useful in situations where memory is
  limited like in shared Kubernetes clusters or low-memory environments.
* `maxmemory-policy allkeys-lru`: Configures Redis to evict the least recently used keys when memory
  is full. This ensures that frequently accessed keys remain in memory.


## Step 5: Apply the updated configuration

Restart the Redis Pod to apply changes and verify the new settings:

1. Apply the updated ConfigMap:

   ```shell
   kubectl apply -f example-redis-config.yaml
   ```

1. Confirm that the ConfigMap was updated:

   ```shell
   kubectl describe configmap/example-redis-config
   ```

   You should see the configuration values you just added:
   
   {{< highlight yaml "hl_lines=10 11" >}}
   Name:         example-redis-config
   Namespace:    default
   Labels:       <none>
   Annotations:  <none>
   
   Data
   ====
   redis-config:
   ----
   maxmemory 2mb
   maxmemory-policy allkeys-lru
   {{< /highlight >}}


1. Check the Redis Pod again using `redis-cli` via `kubectl exec` to see if the configuration was
   applied:

   ```shell
   kubectl exec -it redis -- redis-cli
   ```

1. Check `maxmemory`:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory
   ```
   
   It remains at the default value of `0`:
   
   ```shell
   1) "maxmemory"
   2) "0"
   ```

1. Similarly, `maxmemory-policy` remains at the `noeviction` default setting:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory-policy
   ```
   
   Returns:
   
   ```shell
   1) "maxmemory-policy"
   2) "noeviction"
   ```

1. The configuration values have not changed because the Pod needs to be restarted to grab updated
   values from associated ConfigMaps. Let's delete and recreate the Pod:

   ```shell
   kubectl delete pod redis
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
   ```

1. Now re-check the configuration values one last time:

   ```shell
   kubectl exec -it redis -- redis-cli
   ```

1. Check `maxmemory`:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory
   ```

   It should now return the updated value of `2097152`:

   ```shell
   1) "maxmemory"
   2) "2097152"
   ```

1. Similarly, `maxmemory-policy` has also been updated:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory-policy
   ```
   
   It now reflects the desired value of `allkeys-lru`:
   
   ```shell
   1) "maxmemory-policy"
   2) "allkeys-lru"
   ```


## Step 6: Clean up resources

After completing the tutorial, it's important to remove the created resources to free up cluster
resources and maintain a clean working environment. This step helps prevent unnecessary resource
usage and ensures that your Kubernetes environment remains manageable.

1. Clean up your work by deleting the Redis Pod and ConfigMap:

   ```shell
   kubectl delete pod/redis configmap/example-redis-config
   ```
1. Verify that the resources deleted successfully, run:

   ```shell
   kubectl get pod/redis configmap/example-redis-config
   ```

   You should see no output, indicating that the objects no longer exist.

{{< note title="Why the clean up?" >}}
Leaving unused resources running in a cluster can:

* Waste memory, CPU, or other resources, even if the objects are idle.
* Cause confusion or issues in shared or production environments if other users try to interact with
  leftover resources. {{< /note >}}


## {{% heading "whatsnext" %}}

* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a
  ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
