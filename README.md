This is a high level description for the work I've done for the MapReduce project in CS6210 Advanced Operatings Systems at Georgia Tech. No source code is inlcuded for the purpose of academic integerity.

## Introduction

This project implements a distributed MapReduce framework similar to Google's MapReduce paper. The framework allows for distributed processing of large datasets across multiple worker nodes, handling the complexities of parallelization, fault tolerance, and inter-machine communication.

The system consists of two main components:

1. **Master**: Coordinates the overall MapReduce job, distributes tasks, and handles worker failures
2. **Workers**: Execute map and reduce tasks as assigned by the master

The communication between the master and workers is implemented using gRPC. Specifically, three RPCs are implemented that are called by the master on the workers:

1. **Wait**: Instructs the worker to wait a while and come back later
2. **Map**: Sends a map input (file shard) to the worker to start a map job, receives a list of filenames as the intermediate output
3. **Reduce**: Sends a list of filenames as the reduce input to the worker, and receives a filename

The workers are designed as stateless servers that handle each incoming request synchronously. The logic is fairly standard.

The master uses asynchronous gRPC calls to communicate with workers. This allows the master to handle multiple workers simultaneously without any explicit multithreading. The master is organized into two components:

- **Job Manager**: Maintains input data and states of each individual job (TODO/PROCESSING/DONE), and a global state (phase) of the whole task (MAP/REDUCE/ALL DONE)
- **Async Client**: Manages the connection to each worker. At the beginning of the process, and every time an RPC is completed (which is when a worker is idle and ready to take a task), it queries the job manager to get an appropriate job if one is available, or instructs the worker to wait and come back later

## Error Handling and Fault Tolerance

The following mechanisms are implemented to ensure the robustness of the system:

1. **Deadline Handling**:
   - All worker requests have configurable deadlines
   - If a worker doesn't respond within the deadline, the task is considered failed and available to be assigned to other workers
   - Exponential backoff mechanism increases timeouts for slow workers

2. **Task Recovery**:
   - Failed tasks are automatically returned to the TODO state
   - The system keeps track of how many workers are processing each task
   - Tasks in process are only returned to TODO when all assigned workers have failed

3. **Atomic File Operations**:
   - Workers write output to temporary files first
   - Files are renamed atomically only upon successful completion
   - This prevents corrupt output in case of worker failures

## Slow Worker Handling

We handle potentially slow workers by speculatively assigning parallel jobs to an idle worker when it reports to the master. The master tracks task start times using priority queues. The oldest executing tasks are identified for potential speculative execution.

The system keeps track of how many workers are working on each task. For a single job, only the first worker that successfully returns with a reply is handled, while other responses are ignored.
