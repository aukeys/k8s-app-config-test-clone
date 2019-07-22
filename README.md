头条技术号：实战架构

# 工程目的

提供一个低成本容器化所有组件的最佳实践，方便大家学习/研究。

|              Directory               |                             Description                             |                       备注                       |
| ------------------------------------ | ------------------------------------------------------------------- | --------------------------------------------------- |
| helm                        | 提供helm方式的容器化，步骤及注意事项。                                    |
| helm/min-cluster-allinone   | 用最小资源使用helm方式将大部分基础组件容器化。                            | 最小资源要求是8核32G，需要留足够的内存给os和demo。     |
| other                       | 备份性质目录，没有实际意义。                                |                                              |
| wayne                       | 使用cicd进行最小资源条件下的容器化，目前暂停，意义不大。                  |                                              |
| yaml                        | 提供yaml方式的容器化，步骤及注意事项。  |                                              |
| yaml/min-cluster-allinone   | 用最小资源使用helm方式将大部分基础组件容器化，适当修改可直接用于生产环境。| 最小资源要求是8核32G，需要留足够的内存给os和demo。     |

# QA

## QA1.storage资源不足发生pod驱逐
原来ECS的磁盘小，扩容到100G后解决。
```
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  6m14s (x6 over 7m37s)  default-scheduler  0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
  Normal   Scheduled         6m14s                  default-scheduler  Successfully assigned es-min-kibana/es-min-kibana-kibana-6c685b8d68-cbstb to future
  Normal   Pulling           6m13s                  kubelet, future    pulling image "docker.elastic.co/kibana/kibana:6.4.3"
  Normal   Pulled            5m13s                  kubelet, future    Successfully pulled image "docker.elastic.co/kibana/kibana:6.4.3"
  Normal   Created           5m13s                  kubelet, future    Created container
  Normal   Started           5m12s                  kubelet, future    Started container
  Warning  Unhealthy         4m6s (x4 over 4m55s)   kubelet, future    Readiness probe failed:
  Warning  Evicted           4m6s                   kubelet, future    The node was low on resource: ephemeral-storage. Container kibana was using 66732Ki, which exceeds its request of 0.
  Normal   Killing           4m6s                   kubelet, future    Killing container with id docker://kibana:Need to kill Pod
```

pod被驱逐(Evicted);节点加了污点导致pod被驱逐;ephemeral-storage超过限制被驱逐

EmptyDir 的使用量超过了他的 SizeLimit，那么这个 pod 将会被驱逐;

Container 的使用量（log，如果没有 overlay 分区，则包括 imagefs）超过了他的 limit，则这个 pod 会被驱逐;

Pod 对本地临时存储总的使用量（所有 emptydir 和 container）超过了 pod 中所有container 的 limit 之和，则 pod 被驱逐
ephemeral-storage是一个pod用的临时存储.

```
resources:
       requests: 
           ephemeral-storage: "2Gi"
       limits:
           ephemeral-storage: "3Gi"
节点被驱逐后通过get po还是能看到,用describe命令,可以看到被驱逐的历史原因

Message: The node was low on resource: ephemeral-storage. Container codis-proxy was using 10619440Ki, which exceeds its request of 0.
```

```
来源于:
https://github.com/hepyu/k8s-app-config/tree/master/yaml/min-cluster-allinone/skywalking-min
https://cloud.tencent.com/developer/article/1456427
```
