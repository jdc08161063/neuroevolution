# Neuroevolution

Replication of [Uber AI Labs Neuroevolution paper](https://arxiv.org/pdf/1712.06567.pdf).

> Graph of training curve goes here

## ToDo

- Monitor logs to .txt file
- Run full scale Venture-v4 run
- full environment list in cloudformation
- Another run (maybe one not done by Uber?)
- better ECS image map in cloudformation
    - also note that you need to open spot fleet in console to create role
- AVX2 compiled tensorflow in docker image (c5s support this)


## Approach

I use a master-worker architecture, with:

- Golang master
    - gRPC server that controls the genetic algorithm internals
    - Sends sequences of seeds (individuals) to workers
- Python workers
    - Each has a copy of the policy network and environment
    - Swaps out full network weights on each evaluation (without rebuilding network)
    - Sends seeds and their scores back to master

This results in a simpler system than Uber / OpenAIs approach, which uses redis as an intermediary,
and requires workers to synchronise.


## Deployment

### Building Images

Both master and worker are packaged as docker containers. Either pull the containers from docker-hub,
or build them yourself:
```
# Download
docker pull cshenton/neuro:worker
docker pull cshenton/neuro:master

# Build
docker build -t cshenton/neuro:worker -f worker/Dockerfile .
docker build -t cshenton/neuro:master -f master/Dockerfile .
```

### Launching Cluster

Cloudformation scripts deploy the experiment. The following information is required:
- Availability Zone
- VPC
- Gym Environment Name
- Number of workers

Then the cloudformation scripts create:
- Master
    - Security group (Open on 8080)
    - On-demand instance (`c5.large`)
    - ECS Task
    - Single container ECS Service
    - Log group
- Workers
    - Security group (no ingress)
    - Spot fleet of desired size (`c5.18xlarge`)
    - ECS Task (1 vCPU per container)
    - ECS Service with `numWorkers` tasks
    - Log group

Unfortunately, AWS limits spot fleets to 360 vCPUs per account by default, which is 180 discrete
cpu cores. Therefore, we run 180 workers by default, and larger attempts will fail to fulfill
the bigger spot request.

Running this 180 worker fleet for an hour costs at most `0.04*180 + 0.1 = $7.3`, with more likely
prices hovering around `$4.80`.

For comparison, a p3 instance in the same region (`ap-southeast-2`) costs `$12.24` per hour for a
four NVIDIA Tesla V100 GPU instance.


## Comments



## Protobufs

`gRPC` is used to enable simple communication between workers and master. Server, client stubs
are generated as follows.

```
# python
python -m grpc_tools.protoc -I . proto/neuroevolution.proto --python_out=. --grpc_python_out=.

# golang
protoc -I . proto/neuroevolution.proto --go_out=plugins=grpc:.
```
