# ComputeNode 설치 진행 상황 모니터링 가이드

## 개요

ComputeNode 설치는 15-20분 소요되며, 다음 컴포넌트들이 순차적으로 설치됩니다:
1. EFA Driver (5-10분)
2. Docker + NVIDIA Container Toolkit (3분)
3. Pyxis (2분)
4. CloudWatch Agent (1분)
5. DCGM Exporter (1분)
6. Node Exporter (1분)
7. NCCL 설정 (5초, 있는 경우)

## 🔍 모니터링 방법

### 방법 1: 자동 모니터링 스크립트 (권장)

```bash
# 클러스터 생성 중 또는 생성 후 실행
bash scripts/monitor-compute-node-setup.sh p5en-48xlarge-cluster us-east-2
```

**출력 내용**:
- CloudFormation 스택 상태
- EC2 인스턴스 상태
- CloudWatch 로그에서 설치 진행 상황
- HeadNode 접근 방법

### 방법 2: CloudWatch Logs 실시간 모니터링

```bash
# 실시간 로그 스트리밍
aws logs tail /aws/parallelcluster/p5en-48xlarge-cluster \
  --region us-east-2 \
  --follow \
  --filter-pattern "Compute"

# 설치 단계만 필터링
aws logs tail /aws/parallelcluster/p5en-48xlarge-cluster \
  --region us-east-2 \
  --follow \
  --filter-pattern "\"Installing\" OR \"✓\" OR \"Complete\""
```

### 방법 3: 특정 컴포넌트 설치 확인

```bash
CLUSTER_NAME="p5en-48xlarge-cluster"
REGION="us-east-2"

# EFA 설치 확인
aws logs filter-log-events \
  --log-group-name "/aws/parallelcluster/${CLUSTER_NAME}" \
  --region ${REGION} \
  --filter-pattern "\"Installing EFA\" OR \"EFA installation complete\"" \
  --max-items 10

# Docker 설치 확인
aws logs filter-log-events \
  --log-group-name "/aws/parallelcluster/${CLUSTER_NAME}" \
  --region ${REGION} \
  --filter-pattern "\"Installing Docker\" OR \"Docker installation complete\"" \
  --max-items 10

# NCCL 설정 확인
aws logs filter-log-events \
  --log-group-name "/aws/parallelcluster/${CLUSTER_NAME}" \
  --region ${REGION} \
  --filter-pattern "\"NCCL\" OR \"nccl\"" \
  --max-items 10
```

### 방법 4: EC2 인스턴스 상태 확인

```bash
# ComputeNode 인스턴스 목록
aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=${CLUSTER_NAME}" \
            "Name=tag:Name,Values=Compute" \
  --region ${REGION} \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,IP:PrivateIpAddress,LaunchTime:LaunchTime}' \
  --output table

# 인스턴스가 shutting-down이면 타임아웃 발생
# running 상태가 유지되면 정상 진행 중
```

### 방법 5: HeadNode에서 직접 확인

```bash
# HeadNode SSH 접속
ssh headnode

# Slurm 노드 상태 확인
sinfo -N -l

# ComputeNode에서 설치 상태 확인 스크립트 실행
srun --nodes=1 bash /fsx/scripts/check-compute-setup.sh

# 모든 ComputeNode 확인
srun --nodes=ALL bash /fsx/scripts/check-compute-setup.sh
```

## 📊 설치 진행 단계별 로그 메시지

### 1. 초기화 단계
```
=== Compute Node Setup Started ===
Cluster Name: p5en-48xlarge-cluster
Region: us-east-2
Checking FSx Lustre mount...
✓ FSx Lustre mounted at /fsx
```

### 2. 병렬 설치 단계
```
Installing EFA...
Installing Docker + NVIDIA Container Toolkit...
Installing CloudWatch Agent...
```

### 3. EFA 설치 (가장 오래 걸림)
```
GPU detected - installing with GPU support
Installed EFA packages:
✓ EFA installation complete
```

### 4. Docker 설치
```
✓ Docker + NVIDIA Container Toolkit installation complete
```

### 5. Pyxis 설치
```
Installing Pyxis (Slurm container plugin)...
✓ Pyxis installation complete
(또는)
⚠️  Pyxis build failed (non-critical)
```

### 6. 모니터링 설정
```
Configuring DCGM Exporter...
✓ DCGM Exporter configured (port 9400)
Installing Node Exporter...
✓ Node Exporter configured (port 9100)
```

### 7. NCCL 설정 (있는 경우)
```
Checking for shared NCCL installation...
Found shared NCCL, configuring environment...
✓ Shared NCCL configured
(또는)
⚠️  Shared NCCL not found in /fsx/nccl/
```

### 8. 완료
```
✓ Compute Node Setup Complete
Installed components:
  - EFA Driver + libfabric
  - Docker + NVIDIA Container Toolkit
  - Pyxis (Slurm container plugin)
  - CloudWatch Agent
  - DCGM Exporter (port 9400) - GPU metrics
  - Node Exporter (port 9100) - System metrics
```

## 🚨 문제 발생 시 확인 사항

### 타임아웃 발생 (노드가 shutting-down)

```bash
# CloudFormation 이벤트 확인
aws cloudformation describe-stack-events \
  --stack-name ${CLUSTER_NAME} \
  --region ${REGION} \
  --query 'StackEvents[?contains(ResourceStatusReason, `timeout`) || contains(ResourceStatusReason, `Timeout`)]'

# 마지막 로그 확인 (어디서 멈췄는지)
aws logs get-log-events \
  --log-group-name "/aws/parallelcluster/${CLUSTER_NAME}" \
  --log-stream-name "ip-10-1-XX-XX.i-XXXXX.cloud-init-output" \
  --region ${REGION} \
  --limit 100 \
  --start-from-head \
  --query 'events[-20:].message' \
  --output text
```

**일반적인 타임아웃 원인**:
1. EFA 설치 실패 (네트워크 문제)
2. Docker 설치 실패
3. Pyxis 빌드 실패 (Slurm 헤더 없음) ← 이미 수정됨
4. 타임아웃 설정이 너무 짧음 ← DevSettings.Timeouts 확인

### 설치 에러 확인

```bash
# 에러 메시지 검색
aws logs filter-log-events \
  --log-group-name "/aws/parallelcluster/${CLUSTER_NAME}" \
  --region ${REGION} \
  --filter-pattern "\"Error\" OR \"Failed\" OR \"❌\" OR \"fatal\"" \
  --max-items 50

# 경고 메시지 검색
aws logs filter-log-events \
  --log-group-name "/aws/parallelcluster/${CLUSTER_NAME}" \
  --region ${REGION} \
  --filter-pattern "\"Warning\" OR \"⚠️\"" \
  --max-items 50
```

### 특정 컴포넌트 설치 실패

```bash
# HeadNode에서 수동으로 재설치 가능
ssh headnode

# 특정 ComputeNode에 접속
srun --nodes=1 --nodelist=compute-node-1 bash

# 수동 설치 (예: Docker)
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
```

## 📈 설치 완료 확인

### 모든 컴포넌트 확인

```bash
# HeadNode에서 실행
srun --nodes=ALL bash /fsx/scripts/check-compute-setup.sh
```

**예상 출력**:
```
========================================
ComputeNode Setup Status
========================================
Hostname: compute-node-1
Date: Wed Nov 20 07:30:00 UTC 2025
========================================

=== System Information ===
OS:                           ✓ Installed
  PRETTY_NAME="Ubuntu 22.04.3 LTS"
Kernel:                       ✓ Installed
  6.8.0-1039-aws

=== GPU & Drivers ===
NVIDIA Driver:                ✓ Installed
  570.172.08
CUDA:                         ✓ Installed
  release 12.3
GPU Count:                    ✓ Installed
  8

=== EFA ===
EFA Installer:                ✓ Installed
Libfabric:                    ✓ Installed
EFA Devices:                  ✓ Installed

=== Container Runtime ===
Docker:                       ✓ Installed
  Docker version 24.0.5
NVIDIA Container Toolkit:     ✓ Installed

=== Monitoring ===
DCGM Exporter:                ✓ Running
Node Exporter:                ✓ Running

=== NCCL ===
NCCL Profile Script:          ✓ Installed
NCCL Version:                 ✓ Installed
  v2.28.7-1

========================================
Setup Summary
========================================

Installation Progress: 9/9 components (100%)

✓ All components installed successfully!
```

### 개별 컴포넌트 테스트

```bash
# GPU 테스트
srun --nodes=1 --gpus=1 nvidia-smi

# Docker 테스트
srun --nodes=1 docker run --rm hello-world

# NCCL 테스트
srun --nodes=2 --ntasks=16 --gpus-per-task=1 \
  /opt/nccl-tests/build/all_reduce_perf -b 8 -e 128M -f 2 -g 1

# EFA 테스트
srun --nodes=2 --ntasks=2 \
  /opt/amazon/efa/bin/fi_pingpong -p efa
```

## 🎯 빠른 체크리스트

클러스터 생성 후 다음 순서로 확인:

1. ✅ **CloudFormation 스택 상태**
   ```bash
   aws cloudformation describe-stacks --stack-name ${CLUSTER_NAME} --region ${REGION} --query 'Stacks[0].StackStatus'
   ```
   → `CREATE_COMPLETE` 또는 `CREATE_IN_PROGRESS`

2. ✅ **ComputeNode 인스턴스 상태**
   ```bash
   aws ec2 describe-instances --filters "Name=tag:Name,Values=Compute" --query 'Reservations[*].Instances[*].State.Name'
   ```
   → `running` (shutting-down이면 타임아웃)

3. ✅ **CloudWatch 로그 확인**
   ```bash
   aws logs tail /aws/parallelcluster/${CLUSTER_NAME} --region ${REGION} --since 10m
   ```
   → 설치 진행 메시지 확인

4. ✅ **HeadNode에서 Slurm 확인**
   ```bash
   ssh headnode
   sinfo -N -l
   ```
   → ComputeNode 상태 확인

5. ✅ **설치 상태 확인**
   ```bash
   srun --nodes=1 bash /fsx/scripts/check-compute-setup.sh
   ```
   → 100% 완료 확인

## 📚 관련 문서

- [TIMEOUT-CONFIGURATION.md](TIMEOUT-CONFIGURATION.md) - 타임아웃 설정
- [config/headnode/README.md](config/headnode/README.md) - NCCL 설치
- [config/compute/setup-compute-node.sh](config/compute/setup-compute-node.sh) - 설치 스크립트
- [TROUBLESHOOTING.md](guide/TROUBLESHOOTING.md) - 문제 해결

## 💡 팁

1. **실시간 모니터링**: 클러스터 생성 시작과 동시에 로그 모니터링 시작
2. **타임아웃 여유**: DevSettings.Timeouts를 충분히 설정 (40분 권장)
3. **에러 무시**: 일부 optional 컴포넌트(Pyxis) 실패는 정상
4. **자동 재시도**: ParallelCluster가 실패한 노드를 자동으로 재시작
5. **수동 확인**: 의심스러우면 HeadNode에서 직접 확인
