# Configuring a Redis cache using a ConfigMap object

This topic provides a tutorial for configuring a Redis cache using data stored in a ConfigMap object. This tutorial builds upon the procedures described in [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).


## Prerequisites

Before you begin this tutorial, you must have the following:

* A Kubernetes cluster. We recommend running this tutorial on a cluster with at least two nodes that are not acting as control plane hosts. If you do not already have a cluster, you can do one of the following:
  * Create a cluster with minikube. To create a cluster with minikube, see the [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/?) documentation.
  * Use one of these Kubernetes playgrounds:
    * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
    * [Play with Kubernetes](https://labs.play-with-k8s.com/)

* The `kubectl` command-line tool, configured to communicate with your cluster. The example shown on this page works with `kubectl` v1.14 and above. To check which version of `kubectl` you have installed, run `kubectl version` in a terminal window.

* A working understanding of the procedures described in [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).


## Tutorial: Configure a Redis cache using a ConfigMap object

The tutorial in this topic helps you achieve the following objectives:

* Create a ConfigMap object that can configure a Redis cache. For more information, see [Create a ConfigMap object](#create-a-configmap-object) below.

* Create a Redis pod that mounts and uses the ConfigMap object you create. For more information, see [Create and configure a Redis pod](#create-and-configure-a-redis-pod) below.

* Confirm that your ConfigMap object and Redis pod configurations have updated correctly. For more information, see [Verify your ConfigMap and Redis configurations](#verify-your-configmap-and-redis-configurations) below.

### Create a ConfigMap object

To create a ConfigMap object that can configure a Redis cache:

1. In a terminal window, create a ConfigMap object manifest named `example-redis-config` with an empty `redis-config` section by running:

    ```
    cat <<EOF >./example-redis-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: ""
    EOF
    ```

1. In a terminal window, apply the ConfigMap object manifest you created in the previous step by running:

    ```
    kubectl apply -f example-redis-config.yaml
    ```

### Create and configure a Redis pod

To create and configure a Redis pod:

1. Create a Redis pod by running:

    ```
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

1. In a text editor, open the Redis pod manifest to view the configuration parameters it contains:

    ```yaml
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
    ```
    
    In the `spec` section of the Redis pod manifest, the following properties configure the Redis pod to mount and use the ConfigMap object you created in [Create a ConfigMap object](#create-a-configmap-object) above:
      * The `volumes` section creates a volume named `config`:
  
          ```
	        volumes:
          [...]
          - name: config
            configMap:
              name: example-redis-config
              items:
              - key: redis-config
                path: redis.conf
          ```
          Where:
        
          * `name` is the name of your ConfigMap object, `example-redis-config`.
          * `key` is the name of the section in your ConfigMap object that contains your Redis configuration parameters, `redis-config`.
          * `path` creates a file named `redis.conf` that contains the configuration parameters found in `redis-config`.
        
      * The `containers` section mounts the volumes configured in the `volumes` section:

          ```
          containers:
          - name: redis
            [...]
            volumeMounts:
            [...]
            - mountPath: /redis-master
              name: config
          ```
          Where:
        
          * `name` is the name of the volume to be mounted, `config`.
          * `mountPath`: is the path at which the volume is to be mounted, `/redis-master`.
   
     When these properties are configured correctly, they reference the Redis configuration parameters in your ConfigMap object and expose them at `/redis-master/redis.conf` inside the Redis pod.

1. View the ConfigMap object and Redis pod you created by running:

    ```
    kubectl get pod/redis configmap/example-redis-config 
    ```

    You should see the following terminal output:
    
    ```bash
    NAME        READY   STATUS    RESTARTS   AGE
    pod/redis   1/1     Running   0          8s
    
    NAME                             DATA   AGE
    configmap/example-redis-config   1      14s
    ```

1. View the details of your ConfigMap object by running:

    ```
    kubectl describe configmap/example-redis-config
    ```
    
    Because you left the `redis-config` section empty when you created your ConfigMap object, the terminal output should include an empty `redis-config` field:
    
    ```bash
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    
    Data
    ====
    redis-config:
    ```

1. View the current configuration of your Redis pod:
    1. Enter your Redis pod and start the Redis CLI by running:

        ```
        kubectl exec -it redis -- redis-cli
        ```
    1. View the value of the configured `maxmemory` limit by running:

        ```
        127.0.0.1:6379> CONFIG GET maxmemory
        ```
        
        The terminal output should show the default `maxmemory` value of `0`:
        
        ```bash
        1) "maxmemory"
        2) "0"
        ```
    1. View the value of the configured `maxmemory-policy` limit by running:

        ```
        127.0.0.1:6379> CONFIG GET maxmemory-policy
        ```
    
        The terminal output should show the default `maxmemory-policy` value of `noeviction`:
        
        ```bash
        1) "maxmemory-policy"
        2) "noeviction"
        ```

1. In a text editor, open your ConfigMap object manifest.

1. In the `data` section of your ConfigMap object manifest, configure `maxmemory` and `maxmemory-policy` limits for your Redis cache by editing the `redis-config` section to include the following parameters:
    
    ```yaml
    data:
      redis-config: |
        maxmemory 2mb
        maxmemory-policy allkeys-lru
    ```

1. Save your changes to your ConfigMap object manifest.

1. Apply the updated ConfigMap object manifest by running:

    ```
    kubectl apply -f example-redis-config.yaml
    ```

1. Before your Redis pod can use the updated values in your ConfigMap object, you must restart it. To restart your Redis pod:
    1. Delete your Redis pod by running:
         ```
         kubectl delete pod redis
         ```
    1. Recreate your Redis pod by running:
         ```
         kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
         ```

### Verify your ConfigMap and Redis configurations

To confirm that your ConfigMap object and Redis pod configurations have updated correctly:

1. Confirm that your ConfigMap object has updated correctly by running:

    ```
    kubectl describe configmap/example-redis-config
    ```
    
    The terminal output should show the `maxmemory` and `maxmemory-policy` limits you configured in [Create and configure a Redis pod](#create-and-configure-a-redis-pod) above:
    
    ```bash
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
    ```

1. Confirm that your updated ConfigMap object has correctly re-configured your Redis pod:
     1. Enter your Redis pod and start the Redis CLI by running:
        
          ```
          kubectl exec -it redis -- redis-cli
          ```
     1. View the value of the configured `maxmemory` limit by running:
    
          ```
          127.0.0.1:6379> CONFIG GET maxmemory
          ```

          The terminal output should show a `maxmemory` value of `2097152`, or 2 MB:
    
          ```bash
          1) "maxmemory"
          2) "2097152"
          ```
     1. View the value of the configured `maxmemory-policy` limit by running:

          ```
          127.0.0.1:6379> CONFIG GET maxmemory-policy
          ```

          The terminal output should show a `maxmemory-policy` value of `allkeys-lru`:
          
          ```bash
          1) "maxmemory-policy"
          2) "allkeys-lru"
          ```

1. Clean your work up by deleting your ConfigMap object and Redis pod. Run:

    ```
    kubectl delete pod/redis configmap/example-redis-config
    ```


## Next steps

* To learn more about ConfigMap objects, see [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* To learn more about updating other configurations within a pod, see [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
