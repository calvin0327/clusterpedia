## 检索多集群资源：

```shell
kubectl --cluster clusterpedia api-resources

kubectl --cluster clusterpedia get deployments -l "search.clusterpedia.io/clusters in (cluster-1,cluster-2)”

kubectl --cluster clusterpedia get deployments -l "search.clusterpedia.io/clusters=cluster-1"

kubectl --cluster cluster-1 get deployments
```

## 检索单集群资源：

```shell
kubectl --cluster cluster-1 get deployments -n kube-system

kubectl --cluster cluster-1 get pods -l \
    "search.clusterpedia.io/owner-name=deploy-1,\
     search.clusterpedia.io/owner-seniority=1”
```

## 检索聚合资源：

```shell
kubectl get --raw="/apis/clusterpedia.io/v1beta1/collectionresources/workloads?limit=1" | jq
```