# AWS ParallelCluster for Distributed Training

![Architecture Diagram](img/architecture.png)

AWS ParallelCluster를 사용한 분산 학습 환경 구축 솔루션입니다. XPU 인스턴스 (예: p6-b200.48xlarge with B200 GPUs)에 최적화되어 있으며, 모니터링 스택과 성능 테스트를 포함합니다.

## 🏗️ Architecture Overview

### 주요 구성 요소

```
                    ┌──────────────────────┐
                    │  Application Load    │
                    │     Balancer         │
                    │  (Optional HTTPS)    │
                    └──────────┬───────────┘
                               │
                               │ Port 443/80
                               │
┌──────────────────────────────┼──────────────────────────────┐
│                              │                              │
│  ┌───────────────────────────▼──────┐                      │
│  │   Monitoring Instance             │                      │
│  │   (t3.medium - Standalone)        │                      │
│  ├───────────────────────────────────┤                      │
│  │ • Prometheus                      │                      │
│  │ • Grafana :3000                   │                      │
│  │ • Persistent Storage              │                      │
│  └───────────────────────────────────┘                      │
│                                                               │
│  ┌───────────────────────────────────┐                      │
│  │      LoginNode Pool               │                      │
│  │      (m5.large x2)                │                      │
│  ├───────────────────────────────────┤                      │
│  │ • User Access (SSH)               │                      │
│  │ • Job Submission                  │                      │
│  │ • Data Preprocessing              │                      │
│  └───────────────┬───────────────────┘                      │
│                  │                                           │
│       ┌──────────▼──────────┐    ┌─────────────────────────┐│
│       │   HeadNode          │    │   ComputeNodes          ││
│       │   (m5.8xlarge)      │    │   (p6-b200.48xlarge)    ││
│       ├─────────────────────┤    ├─────────────────────────┤│
│       │ • Slurm Master      │    │ • 8x B200 GPUs (192GB)  ││
│       │ • Job Scheduler     │    │ • 192 vCPUs, 2TB RAM    ││
│       │ • NFS Server        │    │ • 3.2Tbps Network       ││
│       └──────────┬──────────┘    └──────────┬──────────────┘│
│                  │                           │               │
│                  └───────────┬───────────────┘               │
│                              │                               │
│                 ┌────────────▼────────────┐                  │
│                 │   Shared Storage        │                  │
│                 │   (FSx Lustre)          │                  │
│                 │   • High-performance    │                  │
│                 │   • Multi-GB/s          │                  │
│                 └─────────────────────────┘                  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
                        VPC (10.0.0.0/16)
```

### 노드 역할

- **Monitoring Instance (독립형)**: 
  - 클러스터와 별도로 운영되는 모니터링 전용 서버
  - 클러스터 삭제 시에도 모니터링 데이터 유지
  - ALB를 통한 안전한 웹 접근 (직접 Public IP 노출 방지)
  
- **LoginNode Pool**: 
  - 사용자 접근 및 작업 제출 전용
  - 데이터 전처리 및 간단한 작업 수행
  - HeadNode의 컴퓨팅 리소스 보호
  
- **HeadNode**: 
  - Slurm 스케줄러 및 작업 관리
  - Private Subnet에 위치 (보안)
  - NFS 서버 역할
  
- **ComputeNodes**: 
  - GPU 워크로드 실행 전용 (인스턴스 타입별 설정: [가이드](guide/INSTANCE-TYPE-CONFIGURATION.md))
  - Private Subnet에 위치
  - Auto-scaling 지원

📖 **상세 아키텍처 설명**: [guide/ARCHITECTURE.md](guide/ARCHITECTURE.md)

## 📁 Directory Structure

```
.
├── README.md                                    # 이 파일
├── guide/                                       # 상세 가이드 문서
│   ├── ARCHITECTURE.md                          # 아키텍처 상세 설명
│   ├── CONFIGURATION.md                         # 클러스터 설정 가이드
│   ├── INSTALLATION.md                          # 설치 가이드
│   ├── MONITORING.md                            # 모니터링 설정
│   ├── SECURITY.md                              # 보안 가이드
│   └── TROUBLESHOOTING.md                       # 문제 해결
│
├── parallelcluster-infrastructure.yaml          # CloudFormation 인프라 템플릿
├── cluster-config.yaml.template                 # 클러스터 설정 템플릿
├── environment-variables.sh                     # 환경 변수 템플릿
│
├── config/                                      # 설정 스크립트
│   ├── monitoring/                              # 모니터링 인스턴스 (참고용)
│   │   ├── README.md                            # ⚠️ UserData 자동 설치 방식 설명
│   │   └── setup-monitoring-instance.sh         # 수동 재설치용 (참고)
│   ├── headnode/                                # HeadNode 설정
│   ├── loginnode/                               # LoginNode 설정
│   ├── compute/                                 # ComputeNode 설정
│   └── cloudwatch/                              # CloudWatch 설정
│
├── scripts/                                     # 설치 스크립트 (S3 업로드용)
│   ├── nccl/                                    # NCCL 설치 및 테스트
│   ├── efa/                                     # EFA 드라이버
│   ├── cloudwatch/                              # CloudWatch 에이전트
│   └── shared-storage/                          # 공유 스토리지 설정
│
└── tests/                                       # 성능 테스트
    └── nccl/                                    # NCCL 벤치마크
```

## 📦 Prerequisites

```bash
# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# AWS ParallelCluster CLI v3.14.0 in virtual environment
python3 -m venv pcluster-venv
source pcluster-venv/bin/activate
pip install --upgrade "aws-parallelcluster==3.14.0"

# envsubst (템플릿 변수 치환)
# MacOS
curl -L https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-`uname -s`-`uname -m` -o envsubst
chmod +x envsubst && sudo mv envsubst /usr/local/bin

# Linux (CloudShell에는 기본 설치됨)
sudo yum install -y gettext  # Amazon Linux
# sudo apt-get install -y gettext-base  # Ubuntu

# AWS 자격 증명 설정
# region은 클러스터를 배포할 리전과 일치해야함, cluster-config.yaml 파일에서 참조함
aws configure
```

## 🚀 Quick Start

### 1. 인프라 배포

```bash
# 현재 IP 확인
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "Your IP: $MY_IP"

# 기본 배포 (ALB 없음)
REGION="us-east-2"
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=none \
  --capabilities CAPABILITY_IAM

# Self-hosted monitoring with ALB (권장)
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=self-hosting \
    ParameterKey=SecondarySubnetAZ,ParameterValue=${REGION}b \
    ParameterKey=S3BucketName,ParameterValue=my-pcluster-scripts \
    ParameterKey=MonitoringKeyPair,ParameterValue=your-key \
    ParameterKey=AllowedIPsForMonitoringSSH,ParameterValue="${MY_IP}/32" \
    ParameterKey=AllowedIPsForALB,ParameterValue="${MY_IP}/32" \
  --capabilities CAPABILITY_IAM

# AWS Managed Prometheus (AMP) 사용 (자동 생성)
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=amp \
  --capabilities CAPABILITY_IAM

# AMP Workspace 정보 확인
AMP_WORKSPACE_ID=$(aws cloudformation describe-stacks \
  --stack-name parallelcluster-infra \
  --query 'Stacks[0].Outputs[?OutputKey==`AMPWorkspaceId`].OutputValue' \
  --output text)

AMP_ENDPOINT=$(aws cloudformation describe-stacks \
  --stack-name parallelcluster-infra \
  --query 'Stacks[0].Outputs[?OutputKey==`AMPPrometheusEndpoint`].OutputValue' \
  --output text)

echo "AMP Workspace ID: $AMP_WORKSPACE_ID"
echo "AMP Endpoint: $AMP_ENDPOINT"

# ⚠️ 참고: AMP Endpoint를 브라우저로 접근하면 <HttpNotFoundException/>가 표시됩니다.
# 이는 정상 동작입니다! AMP는 Prometheus remote_write API만 제공하며,
# 메트릭 조회는 Grafana를 통해서만 가능합니다.

# AMP Workspace 상태 확인 (ACTIVE여야 정상)
aws amp describe-workspace --workspace-id $AMP_WORKSPACE_ID \
  --query 'workspace.status.statusCode' --output text

# 완전 관리형 모니터링 배포 (AMP + AMG, 권장)
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=amp+amg \
  --capabilities CAPABILITY_NAMED_IAM

# 모니터링 없이 배포 (최소 설정)
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=none \
  --capabilities CAPABILITY_IAM

# 배포 완료 대기 (~5-8분)
aws cloudformation wait stack-create-complete \
  --stack-name parallelcluster-infra \
  --region $REGION
```

### 2. S3 버킷 및 스크립트 업로드

```bash
# S3 버킷 생성
aws s3 mb s3://my-pcluster-scripts --region us-east-2

# 스크립트 업로드
aws s3 sync scripts/ s3://my-pcluster-scripts/scripts/ --region us-east-2
```

### 3. 클러스터 설정 생성

```bash
# 환경 변수 설정
vim environment-variables.sh
# 필수 수정 항목:
# - STACK_NAME
# - KEY_PAIR_NAME
# - S3_BUCKET

# 환경 변수 로드 및 설정 생성
source environment-variables.sh
envsubst < cluster-config.yaml.template > cluster-config.yaml
```

**⚠️ AWS Managed Prometheus 사용 시 추가 설정 필요**

`MonitoringType=amp-only` 또는 `amp+amg`를 선택한 경우, 모든 노드에 AMP remote_write IAM Policy를 추가해야 합니다:

```bash
# AMP Remote Write Policy ARN 확인
AMP_POLICY_ARN=$(aws cloudformation describe-stacks \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`AMPRemoteWritePolicyArn`].OutputValue' \
  --output text)

echo "AMP Policy ARN: $AMP_POLICY_ARN"

# cluster-config.yaml에 수동으로 추가
# HeadNode, LoginNodes, ComputeNodes의 Iam.AdditionalIamPolicies에 추가:
#   - Policy: arn:aws:iam::123456789012:policy/parallelcluster-infra-amp-remote-write
```

또는 environment-variables.sh에 추가하여 자동화:

```bash
# environment-variables.sh에 추가
export AMP_POLICY_ARN="arn:aws:iam::123456789012:policy/parallelcluster-infra-amp-remote-write"

# cluster-config.yaml.template에서 사용
# Iam:
#   AdditionalIamPolicies:
#     - Policy: ${AMP_POLICY_ARN}
```

📖 **상세 설정 가이드**: [guide/CONFIGURATION.md](guide/CONFIGURATION.md)

### 4. 클러스터 생성

```bash
# 클러스터 생성 (WaitCondition 타임아웃 방지를 위해 최소 설치만 수행)
pcluster create-cluster \
  --cluster-name my-cluster \
  --cluster-configuration cluster-config.yaml

# 생성 상태 확인
pcluster describe-cluster --cluster-name my-cluster
```

### 5. 소프트웨어 설치

세 가지 방법 중 선택하여 사용하세요:

#### 방법 1: 공유 스토리지 활용 (권장)

FSx Lustre에 한 번만 설치하고 모든 노드에서 참조:

```bash
# HeadNode에서 NCCL 설치 (한 번만, 10-15분 소요)
ssh headnode
sudo bash /fsx/nccl/install-nccl-shared.sh v2.28.7-1 v1.17.2-aws /fsx
```

**ComputeNode 자동 감지**:
- ✅ **새로 시작되는 노드**: 자동으로 `/fsx/nccl/setup-nccl-env.sh` 감지 및 설정
- ⚠️ **이미 실행 중인 노드**: 수동 적용 필요

```bash
# 이미 실행 중인 ComputeNode에 적용 (클러스터 생성 후 NCCL 설치한 경우)
bash /fsx/nccl/apply-nccl-to-running-nodes.sh

# 또는 수동으로
srun --nodes=ALL bash -c 'cat > /etc/profile.d/nccl-shared.sh << "EOF"
source /fsx/nccl/setup-nccl-env.sh
EOF
chmod +x /etc/profile.d/nccl-shared.sh'
```

**권장 워크플로우**:
1. 클러스터 생성 (ComputeNode MinCount=0으로 설정)
2. HeadNode에서 NCCL 설치
3. Slurm job 제출 → ComputeNode 자동 시작 → NCCL 자동 감지 ✅

**장점**: 
- 빠른 설치 (10-15분, 한 번만)
- 스토리지 효율 (모든 노드가 공유)
- 버전 일관성
- 새 노드 자동 감지

#### 방법 2: 클러스터 생성 시 자동 설치

`cluster-config.yaml`의 CustomActions로 빠른 설치 자동화:

```yaml
ComputeResources:
  - Name: distributed-ml
    CustomActions:
      OnNodeConfigured:
        Script: s3://my-bucket/config/compute/install-pyxis.sh
```

**주의**: NCCL 같은 시간이 오래 걸리는 작업(10-15분)은 WaitCondition 타임아웃(30분)을 유발할 수 있으므로 별도 설치 권장

#### 방법 3: 컨테이너 사용

사전 구성된 컨테이너로 소프트웨어 설치 불필요:

```bash
# Slurm job에서 컨테이너 실행
srun --container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
     --container-mounts=/fsx:/fsx \
     python /fsx/train.py
```

**장점**: 설치 불필요, 재현 가능, 버전 관리 용이

📖 **상세 설치 가이드**: [guide/INSTALLATION.md](guide/INSTALLATION.md)

### Bootstrap 타임아웃 설정

ParallelCluster는 노드 초기화 시 CloudFormation WaitCondition을 사용하며, 기본 타임아웃은 30분입니다. GPU 인스턴스(특히 p5en.48xlarge)는 EFA 드라이버와 NVIDIA 소프트웨어 설치에 시간이 더 걸리므로 타임아웃을 늘려야 합니다.

**현재 설정** (`cluster-config.yaml`):

```yaml
DevSettings:
  Timeouts:
    HeadNodeBootstrapTimeout: 3600      # 60분
    ComputeNodeBootstrapTimeout: 2400   # 40분
```

**타임아웃 근거**:

| 노드 타입 | 실제 설치 시간 | 타임아웃 설정 | 안전 마진 |
|-----------|----------------|---------------|-----------|
| **HeadNode** | ~5분 | 60분 | 12× |
| **ComputeNode** | 15-20분 | 40분 | 2× |

**ComputeNode 설치 시간 상세**:

```
EFA Driver:              5-10분  ← 가장 오래 걸림
Docker + NVIDIA Toolkit:  3분
Pyxis:                    2분
CloudWatch Agent:         1분
DCGM Exporter:            1분
Node Exporter:            1분
NCCL 설정:                5초
─────────────────────────────
총 실제 시간:            15-20분
타임아웃 설정:            40분
안전 마진:               20분
```

**타임아웃 증상**:
- ComputeNode가 `running` 상태에서 곧바로 `shutting-down`으로 전환
- CloudWatch 로그에서 설치가 중간에 중단됨
- CloudFormation 이벤트에 "timeout" 메시지

**타임아웃 조정이 필요한 경우**:
- ✅ 느린 네트워크 환경
- ✅ 대형 인스턴스 타입 (더 많은 드라이버 설치)
- ✅ 복잡한 CustomActions 스크립트
- ✅ 추가 소프트웨어 설치

**타임아웃 모니터링**:

```bash
# CloudFormation 이벤트 확인
aws cloudformation describe-stack-events \
  --stack-name p5en-48xlarge-cluster \
  --region us-east-2 \
  --query 'StackEvents[?contains(ResourceStatusReason, `timeout`)]'

# 인스턴스 상태 확인
aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=p5en-48xlarge-cluster" \
  --region us-east-2 \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,LaunchTime:LaunchTime}'

# CloudWatch 로그 확인
aws logs tail /aws/parallelcluster/p5en-48xlarge-cluster --region us-east-2 --since 1h
```

📖 **타임아웃 상세 가이드**: [guide/TIMEOUT-CONFIGURATION.md](guide/TIMEOUT-CONFIGURATION.md)

### 6. 모니터링 접근

#### Option 1: Amazon Managed Grafana (권장)

```bash
# Grafana 접속 정보 확인 (amp+amg 옵션 사용 시)
aws cloudformation describe-stacks \
  --stack-name parallelcluster-infra \
  --query 'Stacks[0].Outputs[?OutputKey==`GrafanaAccessInstructions`].OutputValue' \
  --output text

# 또는 URL만 확인
GRAFANA_URL=$(aws cloudformation describe-stacks \
  --stack-name parallelcluster-infra \
  --query 'Stacks[0].Outputs[?OutputKey==`ManagedGrafanaWorkspaceEndpoint`].OutputValue' \
  --output text)

echo "Grafana: https://${GRAFANA_URL}"
# AWS SSO로 로그인 (권한 부여 후)
```

#### Option 2: Self-hosting (ALB)

```bash
# ALB DNS 확인
aws cloudformation describe-stacks \
  --stack-name parallelcluster-infra \
  --query 'Stacks[0].Outputs[?OutputKey==`ALBDNSName`].OutputValue' \
  --output text

# 접속: https://<ALB-DNS>/grafana/
# 기본 로그인: admin / Grafana4PC!
```

📖 **모니터링 설정 가이드**: [guide/MONITORING.md](guide/MONITORING.md)

### 7. NCCL 성능 테스트

```bash
# 테스트 스크립트 복사
cp -r tests/nccl/ /fsx/nccl-tests/

# 벤치마크 실행
sbatch /fsx/nccl-tests/nccl-benchmark-suite.sbatch

# 작업 상태 확인
squeue
```

## 📡 Monitoring

### 통합 모니터링 스택

이 솔루션은 GPU, 시스템, 네트워크 성능을 포괄하는 완전한 모니터링 스택을 제공합니다:

| 모니터링 영역 | 도구 | 메트릭 | 포트 |
|--------------|------|--------|------|
| **GPU 성능** | DCGM Exporter | GPU 사용률, 메모리, 온도, 전력 | 9400 |
| **NVLink** | DCGM | GPU 간 통신 대역폭 | - |
| **EFA 네트워크** | EFA Monitor | 노드 간 네트워크 처리량, 패킷 속도 | - |
| **시스템** | Node Exporter | CPU, 메모리, 디스크 | 9100 |
| **Slurm** | Custom Collector | 작업 큐, 노드 상태 | - |

### 자동 설치

모든 모니터링 컴포넌트는 클러스터 생성 시 자동으로 설치됩니다:

- **HeadNode**: Prometheus (메트릭 수집 및 저장)
- **ComputeNode (GPU)**: DCGM Exporter + Node Exporter + EFA Monitor
- **ComputeNode (CPU)**: Node Exporter만 설치

### 모니터링 가이드

- [CloudWatch 모니터링](guide/MONITORING.md) - 기본 모니터링 설정
- [DCGM GPU 모니터링](guide/DCGM-TO-CLOUDWATCH.md) - GPU 메트릭 상세
- [NVLink 모니터링](guide/NVLINK-MONITORING.md) - GPU 간 통신
- [EFA 네트워크 모니터링](guide/EFA-MONITORING.md) - 노드 간 네트워크
- [Prometheus 메트릭](guide/PROMETHEUS-METRICS.md) - 메트릭 쿼리 가이드
- [AMP + AMG 설정](guide/AMP-AMG-SETUP.md) - AWS 관리형 모니터링

### 대시보드 접근

```bash
# CloudWatch 대시보드 (자동 생성)
# - ParallelCluster-<cluster-name>: 기본 대시보드
# - ParallelCluster-<cluster-name>-Advanced: 고급 메트릭
# - ParallelCluster-<cluster-name>-EFA: EFA 네트워크

# Grafana (self-hosting 또는 AMG)
# - GPU Performance
# - NVLink Bandwidth
# - EFA Network
# - Slurm Jobs
```

## 🔧 주요 설정

### Capacity Block과 Placement Group

> ⚠️ **중요**: Capacity Block과 Placement Group은 동시에 사용할 수 없습니다.

**Capacity Block 사용 시**:
- `cluster-config.yaml`에서 `PlacementGroup.Enabled: false` 설정 필수
- Single Spine 구성이 필요한 경우 Capacity Block 예약 전 AWS Account Team에 문의
- 토폴로지 확인: [EC2 Instance Topology](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-topology.html)

**On-Demand/Spot 사용 시**:
- Placement Group 활성화 권장 (최적의 네트워크 성능)

📖 **상세 가이드**: [guide/CONFIGURATION.md](guide/CONFIGURATION.md#️-capacity-block과-placement-group-제약사항)

### 인스턴스 타입 선택

**HeadNode와 LoginNode는 GPU가 필요 없습니다** - 비용 최적화를 위해 CPU 인스턴스 사용을 권장합니다.

| 노드 타입 | 권장 인스턴스 | 용도 | 비용 절감 |
|-----------|---------------|------|-----------|
| HeadNode | m5.2xlarge ~ m5.8xlarge | Slurm 스케줄러 | ~99% |
| LoginNode | m5.large ~ m5.2xlarge | 사용자 접근, 전처리 | ~99% |
| ComputeNode | p6-b200.48xlarge | GPU 워크로드 | - |
| Monitoring | t3.medium | 모니터링 전용 | - |

### 스토리지 구성

- **FSx Lustre** (`/fsx`): 고성능 공유 스토리지
  - 데이터셋, 모델 체크포인트, 학습 출력
  - 멀티 GB/s 처리량
  
- **HeadNode NFS** (`/home`): 기본 공유 디렉토리
  - 사용자 파일, 스크립트
  - 추가 비용 없음
  
- **EBS**: 루트 볼륨 및 로컬 스크래치

### WaitCondition 타임아웃 관리

ParallelCluster는 노드 배포 시 30분 WaitCondition 제한이 있습니다.

**권장 전략**:
1. ✅ **클러스터 생성 시**: 최소 설치만 수행 (빠른 배포)
2. ✅ **생성 완료 후**: 필요한 소프트웨어 수동 설치
3. ✅ **공유 스토리지 활용**: 한 번 설치하여 모든 노드에서 참조
4. ✅ **컨테이너 사용**: 사전 구성된 이미지 활용

**다수의 ComputeNode 관리**:
- 개별 SSH 접속 대신 Slurm job으로 일괄 설치
- 공유 스토리지에 소프트웨어 설치 후 참조
- Docker/Singularity 컨테이너 사용

📖 **상세 가이드**: [guide/INSTALLATION.md](guide/INSTALLATION.md)

## 📊 Expected Performance

### p6-b200.48xlarge 사양

| 항목 | 사양 |
|------|------|
| vCPUs | 192 |
| Memory | 2,048 GiB (2TB DDR5) |
| GPUs | 8x NVIDIA B200 (192GB HBM3e each) |
| Network | 3,200 Gbps |
| NVLink | 900 GB/s per direction |
| Storage | 8x 3.84TB NVMe SSD |

### 성능 지표

- **단일 노드**: 1.2-1.4 TB/s NCCL 대역폭
- **멀티 노드**: >90% 확장 효율성
- **네트워크 지연**: 2-5μs (inter-node)

## 🛡️ Security

### 보안 체크리스트

- [ ] SSH 접근을 특정 IP로 제한 (`AllowedIPsForLoginNodeSSH`)
- [ ] Monitoring Instance는 ALB를 통해서만 접근
- [ ] Grafana 기본 비밀번호 변경
- [ ] SSM Session Manager 사용 (SSH 대신)
- [ ] HeadNode/ComputeNode는 Private Subnet에 배치

### 안전한 접근 방법

```bash
# SSM Session Manager (권장)
aws ssm start-session --target <Instance-ID>

# Grafana 포트 포워딩
aws ssm start-session \
  --target <Monitoring-Instance-ID> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3000"],"localPortNumber":["3000"]}'
```

📖 **보안 가이드**: [guide/SECURITY.md](guide/SECURITY.md)

## 🔍 Troubleshooting

일반적인 문제 해결은 [guide/TROUBLESHOOTING.md](guide/TROUBLESHOOTING.md)를 참조하세요.

**빠른 문제 해결**:

```bash
# 클러스터 상태 확인
pcluster describe-cluster --cluster-name my-cluster

# 로그 확인
pcluster get-cluster-log-events --cluster-name my-cluster

# 설정 검증
pcluster validate-cluster-configuration --cluster-configuration cluster-config.yaml
```

## 📚 Additional Resources

- [AWS ParallelCluster User Guide](https://docs.aws.amazon.com/parallelcluster/)
- [NVIDIA B200 Documentation](https://www.nvidia.com/en-us/data-center/b200/)
- [NCCL Developer Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [EFA User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)

## 📄 License

This project is licensed under the MIT-0 License.

## 🏷️ Tags

`aws` `parallelcluster` `p6` `b200` `gpu` `hpc` `machine-learning` `nccl` `slurm` `efa`
