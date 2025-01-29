# RamPuddle

This is an experiment to test the following thesis.

1. Network latencies are comparable to SSD / NVMe latencies. (Order of 10s of microseconds)
2. Network bandwidths are comparable to SSD / NBMe bandwidths.
Ex: at the HPC instances on [EC2](https://docs.aws.amazon.com/ec2/latest/instancetypes/hpc.html), 
network is at ~ 100Gbps which is about order 10GBps. Instance local store is comparable.

Hence, if carefully implemented, distributed memory (and with care, 
distributed instance storage) can be an effective medium as a storage caching 
layer.


## Architecture
Fundamentally, RamPuddle provides a *trusted* (unauthenticated, or minimally 
authenticated), AP, distributed key-value store for *large* (up to 1GB) 
fixed length values. There is no fault tolerance, no replication, no sharding,
etc. Any replication or load balancing management is left to the
client implementation.

Each server process is single threaded, manages a single fixed size pool 
with optional disk spillover, and is completely stand-alone and is unaware of
any other server proceses.

Clients are simply provided a list of all servers. We can achieve this either
with a single config file, or a controller server, or a service discovery
mechanism like ZooKeeper, Consul, etc.

## Protocol
The protocol is TCP based, but designed with RDMA primitives in mind.

### Scatter
```
Scatter([(key1, offset1, value1), (key2, offset2, value2), ...])
```
Writes the each value to the offset within a given key. Values are truncated
if the key length is insufficient.

### Gather
```
Gather([(key1, offset1, length1), (key2, offset2, length1), ...]) -> [bytes, (retlength1, retlength2, ...)]
```
Reads a number of bytes at the offset within a given key, returning the
result as a single concatenation. If the length exceeds, the result is truncated.
retlength provides the actual return length of each read.
    
### Create
```
Create([(key1, length, value1), (key2, length, value2), ...])
```
Creates a set of keys each with a given length and initial value.
Value does not need to be of the full length, 0 padding will be used.
There is a minimum length of 64KB.
### Delete
```
Delete([key1, key2, ...])
```
Deletes a set of keys.

## Internal Design
The goal is latency miminization; have as few cycles as necessary between 
receiving a message, and returning a response.


## Uses
Anything that needs large files to be read simultaneously by a cluster.
