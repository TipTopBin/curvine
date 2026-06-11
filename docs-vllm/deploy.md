# Amazon SageMaker HyperPod EKS + Curvine L2 KV Cache 部署

HyperPod 的 **Managed Tiered KV Cache** 是一种两级缓存架构，用于在 LLM 推理过程中缓存注意力机制的 Key-Value 向量，避免对已处理 token 的重复计算，可实现 **3-10 倍**延迟优化。

基于 [LMCache](https://github.com/LMCache/LMCache) 实现，与 vLLM 深度集成，将 KV Cache 分布存储在 GPU 显存、CPU 内存、NVMe 和 S3 上。

创新链路：SageMaker HyperPod + Curvine

```text
ALB / Router -> vLLM Worker -> LMCache -> Curvine CSI/FUSE PVC
```

架构示意：

```text
                    ┌─────────────────────────────────┐
                    │       Intelligent Router          │
                    │  (prefixaware/kvaware/session/rr) │
                    └──────────┬──────────────────┬────┘
                               │                  │
                    ┌──────────▼──────┐  ┌────────▼────────┐
                    │   vLLM Replica 1 │  │  vLLM Replica 2  │
                    │  ┌────────────┐  │  │  ┌────────────┐  │
                    │  │  GPU Memory │  │  │  │  GPU Memory │  │
                    │  └──────┬─────┘  │  │  └──────┬─────┘  │
                    │  ┌──────▼─────┐  │  │  ┌──────▼─────┐  │
                    │  │ L1 Cache   │  │  │  │ L1 Cache   │  │
                    │  │ (CPU Mem)  │  │  │  │ (CPU Mem)  │  │
                    │  └──────┬─────┘  │  │  └──────┬─────┘  │
                    └─────────┼────────┘  └─────────┼────────┘
                              │                     │
                    ┌─────────▼─────────────────────▼────────┐
                    │            L2 Cache (Shared)            │
                    │  Redis / SageMaker Tiered Storage /     │
                    │  Curvine (via fs:// FUSE mount)         │
                    └─────────────────────────────────────────┘
```

## 0. 变量

按实际集群和模型修改：

```bash
export REGION="us-west-2"
export HYPERPOD_CLUSTER_NAME="<hyperpod-cluster-name>"
export EKS_CLUSTER_NAME="<eks-cluster-name>"

export ENDPOINT_NAME="curvine-qwen2-7b-instruct"
export NAMESPACE="default"
export MODEL_NAME="Qwen2-7B-Instruct"
export INSTANCE_TYPE="ml.g6.xlarge"
export REPLICAS="1"

export MODEL_BUCKET="<model-bucket>"
export MODEL_LOCATION="models/qwen2-7b/"
export MODEL_IMAGE="public.ecr.aws/deep-learning-containers/vllm:0.11.1-gpu-py312-cu129-ubuntu22.04-ec2-v1.0"
export CERT_S3_URI="s3://<tls-cert-output-bucket-or-prefix>"

export ROUTING_STRATEGY="prefixaware"
export MAX_MODEL_LEN="20000"
export TENSOR_PARALLEL_SIZE="1"
export GPU_COUNT="1"
export CPU_REQUEST="1"
export MEMORY_REQUEST="2Gi"

export CURVINE_NAMESPACE="curvine"
export CURVINE_MASTER_ADDR="curvine-master.curvine.svc.cluster.local:8995"
export CURVINE_FS_PATH="/l2cache"
export CURVINE_PVC="curvine-pvc"
export CURVINE_MOUNT_PATH="/mnt/curvine/l2cache"
```

如果不知道 EKS 集群名：

```bash
export EKS_CLUSTER_NAME=$(aws sagemaker describe-cluster \
  --cluster-name "${HYPERPOD_CLUSTER_NAME}" \
  --query 'Orchestrator.Eks.ClusterArn' \
  --output text | cut -d'/' -f2)
```

## 1. 启用 HyperPod Tiered Storage

Inference Operator 的 CRD 只允许 `tieredstorage` / `redis`，使用 Curvine 时仍先启用
HyperPod tiered storage，通过 CRD 校验后再 patch 到 Curvine。

```bash
aws sagemaker describe-cluster \
  --cluster-name "${HYPERPOD_CLUSTER_NAME}" \
  --query '{Status:ClusterStatus,TieredStorage:TieredStorageConfig,NodeRecovery:NodeRecovery}'
```

启用：

```bash
aws sagemaker update-cluster \
  --cluster-name "${HYPERPOD_CLUSTER_NAME}" \
  --tiered-storage-config Mode=Enable,InstanceMemoryAllocationPercentage=20 \
  --node-recovery Automatic
```

验证：

```bash
aws sagemaker describe-cluster \
  --cluster-name "${HYPERPOD_CLUSTER_NAME}" \
  --query 'TieredStorageConfig'

kubectl get ds -n aws-hyperpod ai-toolkit
kubectl get pods -n aws-hyperpod -l app=ai-toolkit -o wide
```

`update-cluster` 不能只传 `--tiered-storage-config`，需要同时传 `--node-recovery`
或其他可更新字段。

## 2. 安装 Inference Operator

推荐在 SageMaker 控制台安装：

```text
SageMaker Console -> HyperPod Clusters -> 选择集群 -> Inference -> Quick Install
```

安装后验证：

```bash
kubectl get pods -n hyperpod-inference-system

aws eks describe-addon \
  --cluster-name "${EKS_CLUSTER_NAME}" \
  --addon-name amazon-sagemaker-hyperpod-inference \
  --region "${REGION}" \
  --query 'addon.{Status:status,Health:health}' \
  --output table
```

如果用 CLI 安装，需要先准备 operator IAM role 和 `addon-config.json`，再执行：

```bash
aws eks create-addon \
  --cluster-name "${EKS_CLUSTER_NAME}" \
  --addon-name amazon-sagemaker-hyperpod-inference \
  --configuration-values file://addon-config.json \
  --region "${REGION}"
```

## 3. 安装 Curvine

```bash
helm repo add curvine https://curvineio.github.io/helm-charts
helm repo update

helm upgrade --install curvine curvine/curvine -n curvine --create-namespace --devel -f ./value.yaml
```

验证：

```bash
kubectl get pods -n "${CURVINE_NAMESPACE}" -o wide
kubectl get svc -n "${CURVINE_NAMESPACE}"
```

## 4. 创建 Curvine StorageClass 和 PVC

```bash
cat > curvine-sc-pvc.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: curvine-sc
provisioner: curvine
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
parameters:
  master-addrs: "${CURVINE_MASTER_ADDR}"
  fs-path: "${CURVINE_FS_PATH}"
  path-type: "DirectoryOrCreate"
  io-threads: "4"
  worker-threads: "8"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${CURVINE_PVC}
  namespace: ${NAMESPACE}
spec:
  storageClassName: curvine-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF

kubectl apply -f curvine-sc-pvc.yaml
kubectl get sc curvine-sc
kubectl get pvc "${CURVINE_PVC}" -n "${NAMESPACE}"
```

## 5. 部署 vLLM 推理端点

`curvine-qwen2-7b-kvcache.yaml`：

```yaml
apiVersion: inference.sagemaker.aws.amazon.com/v1
kind: InferenceEndpointConfig
metadata:
  name: ${ENDPOINT_NAME}
  namespace: ${NAMESPACE}
spec:
  modelName: ${MODEL_NAME}
  instanceType: ${INSTANCE_TYPE}
  invocationEndpoint: v1/chat/completions
  replicas: ${REPLICAS}

  modelSourceConfig:
    modelSourceType: s3
    s3Storage:
      bucketName: ${MODEL_BUCKET}
      region: ${REGION}
    modelLocation: ${MODEL_LOCATION}
    prefetchEnabled: false

  kvCacheSpec:
    enableL1Cache: true
    enableL2Cache: true
    l2CacheSpec:
      l2CacheBackend: "tieredstorage" # 占位，实际通过 patch 覆盖
      # l2CacheLocalUrl: "fs://localhost:0/mnt/curvine/l2cache/"

  intelligentRoutingSpec:
    enabled: true
    routingStrategy: ${ROUTING_STRATEGY}

  tlsConfig:
    tlsCertificateOutputS3Uri: ${CERT_S3_URI}

  metrics:
    enabled: true
    modelMetrics:
      port: 8000

  loadBalancer:
    healthCheckPath: /health

  worker:
    image: ${MODEL_IMAGE}
    args:
      - "--model"
      - "/opt/ml/model"
      - "--max-model-len"
      - "${MAX_MODEL_LEN}"
      - "--tensor-parallel-size"
      - "${TENSOR_PARALLEL_SIZE}"
    resources:
      limits:
        nvidia.com/gpu: "${GPU_COUNT}"
      requests:
        cpu: "${CPU_REQUEST}"
        memory: ${MEMORY_REQUEST}
        nvidia.com/gpu: "${GPU_COUNT}"
    modelInvocationPort:
      containerPort: 8000
      name: http
    modelVolumeMount:
      name: model-weights
      mountPath: /opt/ml/model
    environmentVariables:
      - name: OPTION_ROLLING_BATCH
        value: "vllm"
      - name: SAGEMAKER_SUBMIT_DIRECTORY
        value: "/opt/ml/model/code"
      - name: MODEL_CACHE_ROOT
        value: "/opt/ml/model"
      - name: SAGEMAKER_MODEL_SERVER_WORKERS
        value: "1"
      - name: SAGEMAKER_MODEL_SERVER_TIMEOUT
        value: "3600"
      # 部署后 Operator 会用 `LMCACHE_REMOTE_URL=sagemaker-hyperpod://$(NODE_IP):9200`
      # 覆盖用户设置，需 patch 替换回来。
      - name: LMCACHE_REMOTE_URL
        value: "fs://localhost:0/mnt/curvine/l2cache/"
      - name: LMCACHE_REMOTE_SERDE
        value: "naive"
```

部署：

```bash
envsubst < curvine-qwen2-7b-kvcache.yaml | kubectl apply -f -

kubectl get inferenceendpointconfig "${ENDPOINT_NAME}" -n "${NAMESPACE}"
kubectl get pods -n "${NAMESPACE}" -l app="${ENDPOINT_NAME}" -o wide
```

单副本部署不需要额外设置 `PYTHONHASHSEED`。如果改成多副本并验证跨 pod 共享，
需要给 worker 加上 `PYTHONHASHSEED=0`，否则不同 pod 的 cache key 可能不一致。

## 6. Patch Deployment 使用 Curvine L2

Operator 使用 `tieredstorage` 时会注入 `LMCACHE_REMOTE_URL=sagemaker-hyperpod://...`。
等 Deployment 生成后，把 PVC 挂进去，并把 LMCache remote URL 改回 Curvine。

```bash
kubectl get deployment -n "${NAMESPACE}" "${ENDPOINT_NAME}"
```

挂载 PVC：

```bash
kubectl patch deployment "${ENDPOINT_NAME}" -n "${NAMESPACE}" --type=json -p="[
  {
    \"op\": \"add\",
    \"path\": \"/spec/template/spec/volumes/-\",
    \"value\": {
      \"name\": \"curvine-cache\",
      \"persistentVolumeClaim\": {\"claimName\": \"${CURVINE_PVC}\"}
    }
  },
  {
    \"op\": \"add\",
    \"path\": \"/spec/template/spec/containers/0/volumeMounts/-\",
    \"value\": {
      \"name\": \"curvine-cache\",
      \"mountPath\": \"${CURVINE_MOUNT_PATH}\"
    }
  }
]"
```

覆盖 LMCache 环境变量：

```bash
kubectl set env deployment/"${ENDPOINT_NAME}" -n "${NAMESPACE}" \
  LMCACHE_REMOTE_URL="fs://localhost:0${CURVINE_MOUNT_PATH}/" \
  LMCACHE_REMOTE_SERDE="naive"

kubectl rollout status deployment/"${ENDPOINT_NAME}" -n "${NAMESPACE}"
```

确认 patch 生效：

```bash
kubectl get deployment "${ENDPOINT_NAME}" -n "${NAMESPACE}" -o yaml | grep -A2 'LMCACHE'
kubectl get deployment "${ENDPOINT_NAME}" -n "${NAMESPACE}" -o yaml | grep -A6 'curvine-cache'
```

## 7. 推理验证

直接请求 Worker：

```bash
WORKER_POD=$(kubectl get pods -n "${NAMESPACE}" -l app="${ENDPOINT_NAME}" \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n "${NAMESPACE}" "${WORKER_POD}" -- \
  curl -s http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"/opt/ml/model","messages":[{"role":"user","content":"Hello"}],"max_tokens":64}'
```

通过 Router：

```bash
ROUTER_POD=$(kubectl get pods -n hyperpod-inference-system \
  -l app="${ENDPOINT_NAME}-${NAMESPACE}-router" \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n hyperpod-inference-system "${ROUTER_POD}" -- \
  curl -s http://localhost:8081/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"/opt/ml/model","messages":[{"role":"user","content":"Hello"}],"max_tokens":64}'
```

检查 LMCache 日志：

```bash
kubectl logs -n "${NAMESPACE}" -l app="${ENDPOINT_NAME}" \
  -c "${ENDPOINT_NAME}" --tail=200 | grep -i 'lmcache\|external prefix cache'
```

常见有效日志：

```text
LMCache INFO: Stored 256 out of total 256 tokens.
LMCache INFO: Retrieved 256 out of 256 required tokens.
Engine 000: External prefix cache hit rate: 96.2%
```

确认 Curvine 目录有 cache 文件：

```bash
kubectl exec -n "${NAMESPACE}" "${WORKER_POD}" -- \
  sh -c "find ${CURVINE_MOUNT_PATH} -maxdepth 1 -type f | head"
```

## 8. 跨节点共享验证

跨节点共享需要至少 2 个副本分布在不同节点，并显式固定 Python hash seed：

```bash
kubectl scale deployment/"${ENDPOINT_NAME}" -n "${NAMESPACE}" --replicas=2
kubectl set env deployment/"${ENDPOINT_NAME}" -n "${NAMESPACE}" PYTHONHASHSEED="0"
kubectl rollout status deployment/"${ENDPOINT_NAME}" -n "${NAMESPACE}"
```

```bash
kubectl get pods -n "${NAMESPACE}" -l app="${ENDPOINT_NAME}" -o wide
```

执行两次相同 prompt，观察两个 pod 日志。期望：

```text
Pod 1: Stored 256 out of total 256 tokens
Pod 2: Retrieved 256 out of 256 required tokens
```

如果第二个 pod 没有命中，优先检查：

```bash
kubectl get deployment "${ENDPOINT_NAME}" -n "${NAMESPACE}" -o yaml | grep -A2 PYTHONHASHSEED
kubectl get deployment "${ENDPOINT_NAME}" -n "${NAMESPACE}" -o yaml | grep -A2 LMCACHE_REMOTE_URL
kubectl get pvc "${CURVINE_PVC}" -n "${NAMESPACE}"
```

## 9. 常用排障命令

```bash
# Operator
kubectl logs -n hyperpod-inference-system \
  deployment/hyperpod-inference-controller-manager --tail=100

# Worker pod
kubectl describe pod -n "${NAMESPACE}" -l app="${ENDPOINT_NAME}"
kubectl logs -n "${NAMESPACE}" -l app="${ENDPOINT_NAME}" --tail=100

# GPU
kubectl get nodes \
  -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia.com/gpu

# Events
kubectl get events -n "${NAMESPACE}" --sort-by='.lastTimestamp' | tail -20

# PVC / mount
kubectl get pvc "${CURVINE_PVC}" -n "${NAMESPACE}"
kubectl exec -n "${NAMESPACE}" "${WORKER_POD}" -- mount | grep curvine
kubectl exec -n "${NAMESPACE}" "${WORKER_POD}" -- df -h "${CURVINE_MOUNT_PATH}"
```

VPC Endpoint 安全组未放行时，新节点上的 `aws-node` 可能卡在
`Checking for IPAM connectivity...`。放行节点安全组到 VPC Endpoint 安全组的 443：

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "<VPCE_SG>" \
  --protocol tcp \
  --port 443 \
  --source-group "<NODE_SG>" \
  --region "${REGION}"
```

## 10. 关键配置速查

| 配置 | 建议值 | 说明 |
| --- | --- | --- |
| `l2CacheBackend` | `tieredstorage` | 通过 CRD 校验，后续 patch 到 Curvine |
| `LMCACHE_REMOTE_URL` | `fs://localhost:0/mnt/curvine/l2cache/` | LMCache 走 Curvine FUSE |
| `LMCACHE_REMOTE_SERDE` | `naive` | 文件后端实测可用 |
| `PYTHONHASHSEED` | `0` | 多副本跨 pod 共享时设置，保证 cache key 一致 |
| PVC access mode | `ReadWriteMany` | 多副本共享 L2 cache |
| routing strategy | `prefixaware` / `kvaware` | 多轮对话优先 `prefixaware`，通用缓存命中可试 `kvaware` |
