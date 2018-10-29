# Steps for TF Serving Deployment using Docker

1. Install Docker from the following [link](https://www.tensorflow.org/install/docker#gpu_support)

2. Pull the latest tensorflow docker image 
````
docker pull tensorflow/serving:latest-devel-gpu
````

3. Initialize and enter the docker container
```
docker run --runtime nvidia \
--name nameofdockercontainer \
--restart=always \
-it \
-p 4000:4000 \
-p 4001:4001 \
tensorflow/serving:latest-devel-gpu
```

`--runtime nvidia` : This argument enables the docker container to communicate with GPUs.

`--name` : This argument gives a name to the docker container.

`--restart=always` : Always restart the container if it stops.

`-it` : This argument ensures that container starts with interactive mode. That means we’ll be inside the container once it starts. 

`-p 3000:3000` : This command publishes the container᾿s port or a range of ports to the host. Format: hostPort:containerPort.


4. Once you enter the docker container, to detach from the container, press the key combination `ctrl + pq`

5. Create a file called `config.conf`
```
model_config_list: {
config: {
name: "modelnumber1",
base_path: "/tensorflow-serving/models/modelnumber1",
model_platform: "tensorflow",
model_version_policy: {
       specific: {
               versions: 1
               }
       }
},
config: {
name: "modelnumber2",
base_path: "/tensorflow-serving/models/modelnumber2",
model_platform: "tensorflow",
model_version_policy: {
       latest: {
               num_versions: 2
               }
       }
}
}
```
This configuration file has a list of configurations for each models we want to host.
- name: Model name to specify which model we are calling for predictions during inference.
- base_path: Base path for models. This basepath will contain all the versions of each model.
- model_version_policy: This argument decides how we want to manage versioning of each model.

Tensrflow will then take care of serving each of the listed models, and automatically manage the versioning. Insertion of a new version will be automatically picked up by Tensorflow while the injection of an entirely new model will require a restart of the serving service.


6. Copy `config.conf` to the docker container

```
docker cp config.conf nameofdockercontainer:/tensorflow-serving/
```

7. Create a directory called `models` that is structured in the following way

```
models/
├── modelnumber1
│   └── 1
│       ├── saved_model.pb
│       └── variables
│           ├── variables.data-00000-of-00001
│           └── variables.index
├── modelnumber2
│   └── 1
│       ├── saved_model.pb
│       └── variables
│           ├── variables.data-00000-of-00001
│           └── variables.index

```

8. Copy the `models` directory to the docker container

```
docker cp models nameofdockercontainer:/tensorflow-serving/
```

9. To enable batching, create a file called `batching.txt`

```
max_batch_size { value: 1024 }
batch_timeout_micros { value: 0 }
max_enqueued_batches { value: 1000000 }
num_batch_threads { value: 8 }
```

The arguments are explained below:

- `max_batch_size`: The maximum size of any batch. This parameter governs the throughput/latency tradeoff, and also avoids having batches that are so large they exceed some resource constraint (e.g. GPU memory to hold a batch's data).
- `batch_timeout_micros`: The maximum amount of time to wait before executing a batch (even if it hasn't reached max_batch_size). Used to rein in tail latency.
- `num_batch_threads`: The degree of parallelism, i.e. the maximum number of batches processed concurrently.
max_enqueued_batches: The number of batches worth of tasks that can be enqueued to the scheduler. Used to bound queueing delay, by turning away requests that would take a long time to get to, rather than building up a large backlog.

10. Copy `batching.txt` to the docker container

```
docker cp bactching.txt nameofdockercontainer:/tensorflow-serving/
```

9. Get inside the docker container.

```
docker attach nameofdockercontainer
```

10. Start the server

```
tensorflow_model_server \
--port=4000 \
--rest_api_port=4001 \
--model_config_file=config.conf &> logs &
```

If we are using batching then run

```
tensorflow_model_server \
--port=4000 \
--rest_api_port=4001 \
--enable_batching=true
--batching_parameters_file=batching.txt
--model_config_file=config.conf &> logs &
```


We can give the following arguments to `tensorflow_model_server`:
- `port`: Port to listen on for gRPC API
- `rest_api_port`: Port to listen on for HTTP/REST API. This port must be different than the one specified in --port.
- `rest_api_num_threads`: Number of threads for HTTP/REST API processing. If not set, will be auto set based on number of CPUs.
- `rest_api_timeout_in_ms`: Timeout for HTTP/REST API calls.
- `enable_batching`: enable batching (true or false).
- `batching_parameters_file`: Parameters for batching are specified by this file.
- `model_config_file`: Reads the given file, and serve the models in that file. This config file can be used to specify multiple models to serve and other advanced parameters including non-default version policy.
- `per_process_gpu_memory_fraction`: Fraction that each process occupies of the GPU memory space the value is between 0.0 and 1.0 (with 0.0 as the default) If 1.0, the server will allocate all the memory when the server starts, If 0.0, Tensorflow will automatically select a value.
- `grpc_channel_arguments`: A comma separated list of arguments to be passed to the grpc server. (e.g. grpc.max_connection_age_ms=2000)
- `enable_model_warmup`: Enables model warmup, which triggers lazy initializations (such as TF optimizations) at load time, to reduce first request latency.

11. The server will be running at 
- `localhost:4001/v1/models/modelnumber1/versions/1:predict`
- `localhost:4001/v1/models/modelnumber2/versions/1:predict`

If hosted on a VM instance `localhost` can be replaced with the IP address of the the instance.

## Some Handy Commands

1. To check if server is running, enter the docker container and run

```
ps aux
```

2. To check what containers are running

```
docker ps 

docker ps -a # to view stopped containers as well
```

3. To start a container

```
docker start nameofdockercontainer
```
