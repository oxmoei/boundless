# Step-by-Step Guide to Modify the Number of GPUs
The provided [compose.yml](https://github.com/0xmoei/boundless/blob/main/compose.yml) file is configured to use 4 GPUs, each assigned to a `gpu_prove_agent` service (`gpu_prove_agent0` to `gpu_prove_agent3`).

To adjust the number of GPUs—either by adding or removing them. You need to modify the `gpu_prove_agent` service definitions and update the `depends_on` list in the `broker` service.

This guide outlines the specific parts of the file to change.

## Identify the Target Number of GPUs
* Decide how many GPUs you want to use (e.g., 3 instead of 4, or 5 instead of 4).
* Confirm the GPU device IDs available on your host with `nvidia-smi -L`

## Modify the compose.yml File
* Open the `compose.yml` file in a text editor.
* Locate the `gpu_prove_agentX` services (lines defining `gpu_prove_agent0` to `gpu_prove_agent3`) and the broker service’s `depends_on` list.

## Option 1: To Add a GPU
### 1- Duplicate a `gpu_prove_agent` service definition:
* Copy an existing block, such as the one for `gpu_prove_agent3`:
```
gpu_prove_agent3:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G
  cpus: 4
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['3']
            capabilities: [gpu]
```
* Rename it to the next sequential number (e.g., `gpu_prove_agent4` for a fifth GPU).
* Update the `device_ids` field to the new GPU ID (e.g., change `'3'` to `'4'`).
* Example for `gpu_prove_agent4`:
```
gpu_prove_agent4:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G
  cpus: 4
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['4']
            capabilities: [gpu]
```

### 2- Update the `broker` service’s `depends_on` list:
Find the `broker` service:
```yaml
broker:
  restart: always
  depends_on:
    - rest_api
    - gpu_prove_agent0
    - gpu_prove_agent1
    - gpu_prove_agent2
    - gpu_prove_agent3
    - exec_agent0
    - exec_agent1
    - aux_agent
    - snark_agent
    - redis
    - postgres
```

* Add the new service name (e.g., `gpu_prove_agent4`) to the depends_on list:
```yaml
depends_on:
  - rest_api
  - gpu_prove_agent0
  - gpu_prove_agent1
  - gpu_prove_agent2
  - gpu_prove_agent3
  - gpu_prove_agent4
  - exec_agent0
  - exec_agent1
  - aux_agent
  - snark_agent
  - redis
  - postgres
```


## Option 2: To Remove a GPU
### 1- Delete a `gpu_prove_agent` service definition:
* Remove the entire block for the service you no longer need (e.g., `gpu_prove_agent3`):
```yaml
gpu_prove_agent3:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G
  cpus: 4
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['3']
            capabilities: [gpu]
```

### 2- Update the broker service’s depends_on list:
* Remove the corresponding service name (e.g., `gpu_prove_agent3`) from the `depends_on` list in the `broker` service:
```yaml
gpu_prove_agent3:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G
  cpus: 4
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['3']
            capabilities: [gpu]
```


