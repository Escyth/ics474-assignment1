# Assignment 1
**Prepared by:**
- Usama Bakkar (202263960)
- Naif Alenazi (202163490)
- Anas Alhanaya (201967330)

**Tool:** [Apache Flink](https://flink.apache.org/)

**Demo**: https://youtu.be/IPROpHU06tE

## Installation:
The team chose a local cluster approach with containers using [Docker](https://www.docker.com/) because it's 1. easier to manage, 2. sufficient to simulate the environment.

Following the official [setup guide](https://nightlies.apache.org/flink/flink-docs-release-2.1/docs/deployment/resource-providers/standalone/docker/) for Flink v2.1.0 on Docker:

Why Session Mode over Application Mode?
Session Mode is basically a cluster that runs long-term. Each job can be submitted to the cluster after it has been deployed for execution. On the other hand, Application Mode is a dedicated cluster which runs a single job that it has been compiled with, which is ideal for production use, rather than experimental use (our assignment).

1. Ensure Docker Desktop is installed
2. Get the appropriate `docker-compose.yml` for Session Mode (Appendix A)
3. Open a terminal in the same directory
4. Run `docker compose up` to launch a cluster with 1 JobManager and 1 TaskManager
5. (Optional) Run `docker compose up -d --scale taskmanager=<N>` to scale the cluster up/down to N TaskManagers
6. Access the Flink Web UI at http://localhost:8081/ (Appendix B)
7. Run `docker compose down` or `Ctrl+C` to kill the cluster

## Experimentation:
The workload will be a word count operation, which is a computational one. In order to evaluate performance across different metrics, other variables will be fixed except for the one we are testing. Please note that GPU % is not applicable because the selected workload is CPU-bound.

- **Data volume:** small, medium, and large
- **Compute nodes (TM):** 1, 2, and 3

### Data volume (nodes=1 TM / 2 task slots)
Command:
```sh
docker exec -it assignment1-jobmanager-1 /opt/flink/bin/flink run "-Dexecution.runtime-mode=BATCH" -p 2 /opt/flink/examples/streaming/WordCount.jar --input file:///home/flink/data/<file_name> --output /tmp/out
```

small.txt (~2 MB)
|  Metric |       Value       |
|:-------:|:-----------------:|
|  CPU %  |       33.03%      |
|  MEM %  |        7.5%       |
| NET I/O | 6.73 MB / 5.23 MB |
|   Time  |       568 ms      |

medium.txt (~29 MB)
|  Metric  |      Value    |
|:-------:|:-------------:|
|  CPU %  |      133%     |
|  MEM %  |      7.8%     |
| NET I/O | 6.29 MB / 4.95 MB |
|   Time  |    6707 ms    |

large.txt (~81 MB)
|  Metric  |      Value    |
|:-------:|:-------------:|
|  CPU %  |      136%     |
|  MEM %  |       8%      |
| NET I/O | 6.49 MB / 5.05 MB |
|   Time  |    18646 ms   |

### Compute nodes (data size=medium)
Commands:
```sh
docker compose up -d --scale taskmanager=<N>
docker exec -it assignment1-jobmanager-1 /opt/flink/bin/flink run "-Dexecution.runtime-mode=BATCH" -p <2*N> /opt/flink/examples/streaming/WordCount.jar --input file:///home/flink/data/medium.txt --output /tmp/out
```

1 TM (2 task slots)
|  Metric  |      Value    |
|:-------:|:-------------:|
|  CPU %  |      133%     |
|  MEM %  |      7.8%     |
| NET I/O | 6.29 MB / 4.95 MB |
|   Time  |    6707 ms    |

2 TM (4 task slots)
|  Metric  |      Value (TM1, TM2)    |
|:-------:|:-------------:|
|  CPU %  |      98%, 38%     |
|  MEM %  |      7.2%, 7.3%     |
| NET I/O | 38.9 MB / 300 kB, 405 kB, 38.8 MB |
|   Time  |    5806 ms    |

3 TM (6 task slots)
|  Metric  |      Value (TM1, TM2, TM3)    |
|:-------:|:-------------:|
|  CPU %  |      66%, 28%, 64%     |
|  MEM %  |      7.74%, 7%, 6.9%     |
| NET I/O | 567 kB / 69.7 MB, 17.1 MB / 205 kB, 52.9 MB / 371 kB |
|   Time  |    5324 ms    |

## Findings:
The following summarizes observed trends for each variable.

Based on the data volume experiment measurements:
- CPU usage increases from low percentages to higher percentages demonstrating full utilization of available slots once data is sufficient to saturate both cores
- Stable memory usage, around 7-8%, indicating efficient state management
- Network I/O is constant because shuffling occurs once regardless of input size
- In execution time, we can almost notice exact linearity, confirming predictable throughput aligning with what we theoretically studied (0.5s -- *13.8 size --> 6.7s -- *2.8 size --> 18.6s)

Based on the compute nodes experiment measurements:
- CPU usage per TaskManager decreases as more nodes are added, showing distribution of workload. Additionally, overall usage remains around ~140% confirming parallel execution
- Memory usage is consistent, demonstrating that scaling increases available slots but doesn't inflate per-node memory overhead
- Execution time slightly improves; however, its scaling efficiency is limited by network and coordination costs, but still shows measurable gains

## Conclusion
The experiments demonstrate that Apache Flink efficiently scales both vertically (with data volume) and horizontally (with additional TaskManagers). While CPU utilization and throughput increase predictably with workload size, network overhead limits linear speedup in multi-node configurations — a realistic behavior for distributed systems.


## Appendices:

**A**
```yml
version: "2.2"
services:
  jobmanager:
    image: flink:2.1.0-scala_2.12
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager        

  taskmanager:
    image: flink:2.1.0-scala_2.12
    depends_on:
      - jobmanager
    command: taskmanager
    scale: 1
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2
```

> **numberOfTaskSlots** can be thought of as number of CPU threads per TaskManager.

**B** (using taskmanagers=2)

![Flink Web UI](https://i.imgur.com/8Zk0tDk.png)