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
--restart=always
-it \
-p 4000:4000 \
-p 4001:4001 \
tensorflow/serving:latest-devel-gpu
```

    `--runtime nvidia` : This argument enable the docker container to communicate with GPUs.

    `--name` : This argument gives name to the docker container

    `--restart=always` : 

4. Once you enter the docker container, 

4. Create a file called `config.conf`
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
base_path: "/tensorflow-serving/models/human_classifiemodelnumber1r",
model_platform: "tensorflow",
model_version_policy: {
       latest: {
               num_versions: 2
               }
       }
}
}
```

