# AWS ParallelCluster for Distributed Training

![Architecture Diagram](img/architecture.png)

AWS ParallelCluster를 사용한 분산 학습 환경 구축을 위한 에셋입니다. GPU 및 CPU 인스턴스를 컴퓨트 노드로 활용할 수 있으며, 모니터링 스택과 자동화된 설정을 포함합니다.

## 🏗️ Architecture Overview

### 노드 역할

- **LoginNode Pool (Optional)**: 
  - 사용자 SSH 접근 및 작업 제출 전용
  - 데이터 전처리 및 간단한 작업 수행
  - HeadNode의 컴퓨팅 리소스 보호
  - Public Subnet (특정 IP만 SSH 허용)
  
- **HeadNode**: 
  - Slurm 스케줄러 및 작업 관리
  - NFS 서버 역할 (/home 공유)
  - Private Subnet에 위치 (보안)
  
- **ComputeNodes**: 
  - GPU 워크로드 실행 전용
  - Private Subnet에 위치
  - Auto-scaling 지원 (Slurm 연동)
  - EFA 네트워크로 노드 간 고속 통신

### 모니터링 아키텍처

**AWS Managed Services (권장)**:
- **Amazon Managed Prometheus (AMP)**: 메트릭 저장 및 쿼리
- **Amazon Managed Grafana (AMG)**: 대시보드 및 시각화
- **장점**: 관리 부담 없음, 고가용성, 자동 스케일링, AWS SSO 통합

**Self-hosting (대안)**:
- Standalone Monitoring Instance (t3.medium)
- Prometheus + Grafana 직접 운영
- ALB를 통한 HTTPS 접근
- 클러스터와 독립적으로 운영

## 📁 Directory Structure

```
.
├── README.md                                    # 이 파일
├── guide/                                       # 상세 가이드 문서
│   ├── AMP-AMG-SETUP.md                         # AWS Managed Prometheus + Grafana 설정
│   ├── DCGM-TO-CLOUDWATCH.md                    # GPU 메트릭 모니터링
│   ├── EFA-MONITORING.md                        # EFA 네트워크 모니터링
│   ├── NVLINK-MONITORING.md                     # NVLink 모니터링
│   ├── PROMETHEUS-METRICS.md                    # Prometheus 메트릭 가이드
│   ├── QUICKSTART-EFA-MONITORING.md             # 빠른 시작 가이드
│   ├── CLUSTER-RECREATION-GUIDE.md              # 클러스터 재생성 가이드
│   ├── TIMEOUT-CONFIGURATION.md                 # 타임아웃 설정 가이드
│   └── README.md                                # 가이드 목차
│
├── parallelcluster-infrastructure.yaml          # CloudFormation 인프라 템플릿
├── cluster-config.yaml.template                 # 클러스터 설정 템플릿
├── environment-variables.sh                     # 환경 변수 템플릿
├── environment-variables-bailey.sh              # 환경 변수 예제 (bailey)
│
├── config/                                      # 노드 설정 스크립트 (S3 업로드용)
│   ├── README.md                                # config 디렉토리 설명
│   ├── STRUCTURE-SUMMARY.md                     # 구조 요약
│   ├── monitoring/                              # 모니터링 인스턴스 설정
│   │   ├── README.md                            # UserData 자동 설치 방식 설명
│   │   └── setup-monitoring-instance.sh         # 수동 재설치용 (참고)
│   ├── headnode/                                # HeadNode 설정
│   │   └── setup-headnode.sh                    # Prometheus + CloudWatch
│   ├── loginnode/                               # LoginNode 설정
│   │   └── setup-loginnode.sh                   # 기본 도구 + CloudWatch
│   ├── compute/                                 # ComputeNode 설정
│   │   └── setup-compute-node.sh                # GPU/CPU 모드별 설치
│   ├── cloudwatch/                              # CloudWatch 설정
│   │   ├── dcgm-to-cloudwatch.sh                # DCGM 메트릭 전송
│   │   └── create-efa-dashboard.sh              # EFA 대시보드 생성
│   ├── nccl/                                    # NCCL 설치 스크립트
│   └── efa/                                     # EFA 드라이버 설치
│
├── scripts/                                     # 유틸리티 스크립트
│   ├── check-compute-setup.sh                   # ComputeNode 설정 확인
│   ├── monitor-compute-node-setup.sh            # 설치 진행 모니터링
│   └── upload-monitoring-scripts.sh             # S3 업로드 스크립트
│
└── security-best-practices/                     # 보안 가이드
    └── SECURITY.md                              # 보안 모범 사례
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

**모니터링 옵션**:
- `none`: 모니터링 없음 (최소 구성)
- `self-hosting`: Standalone Prometheus + Grafana (t3.medium 인스턴스)
- `amp-only`: AWS Managed Prometheus만 사용
- `amp+amg`: AWS Managed Prometheus + Grafana (권장)

```bash
# 현재 IP 확인
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "Your IP: $MY_IP"

# [none] 기본 배포 (최소 설정)
REGION="us-east-2"
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=none \
  --capabilities CAPABILITY_IAM

# [self-hosting] Self-hosted monitoring (EC2+ALB)
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

# [amp-only] AWS Managed Prometheus (AMP) 사용 (자동 생성)
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=amp-only \
  --capabilities CAPABILITY_IAM

## AMP Workspace 정보 확인
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

## AMP Workspace 상태 확인 (ACTIVE여야 정상)
aws amp describe-workspace --workspace-id $AMP_WORKSPACE_ID \
  --query 'workspace.status.statusCode' --output text

# [amp+amg] 완전 관리형 모니터링 배포 (AMP + AMG, 권장)
aws cloudformation create-stack \
  --stack-name parallelcluster-infra \
  --region $REGION \
  --template-body file://parallelcluster-infrastructure.yaml \
  --parameters \
    ParameterKey=PrimarySubnetAZ,ParameterValue=${REGION}a \
    ParameterKey=MonitoringType,ParameterValue=amp+amg \
  --capabilities CAPABILITY_NAMED_IAM

# 배포 완료 대기 (~5-8분)
aws cloudformation wait stack-create-complete \
  --stack-name parallelcluster-infra \
  --region $REGION
```

### 2. S3 버킷 및 config 업로드

```bash
# S3 버킷 생성
aws s3 mb s3://my-pcluster-scripts --region us-east-2

# config 디렉토리 업로드 (노드 설정 스크립트)
# ⚠️ 중요: CustomActions가 이 스크립트들을 참조합니다
aws s3 sync config/ s3://my-pcluster-scripts/config/ --region us-east-2

# 업로드 확인
aws s3 ls s3://my-pcluster-scripts/config/ --recursive

# 예상 출력:
# config/headnode/setup-headnode.sh
# config/loginnode/setup-loginnode.sh
# config/compute/setup-compute-node.sh
# config/cloudwatch/dcgm-to-cloudwatch.sh
# config/cloudwatch/create-efa-dashboard.sh
# ... (기타 파일들)
```

**config 디렉토리 구조**:
- `headnode/`: HeadNode 설정 (Prometheus + CloudWatch)
- `loginnode/`: LoginNode 설정 (기본 도구 + CloudWatch)
- `compute/`: ComputeNode 설정 (GPU/CPU 모드별 설치)
- `cloudwatch/`: CloudWatch 관련 스크립트
- `nccl/`: NCCL 설치 스크립트
- `efa/`: EFA 드라이버 설치

📖 **상세 구조**: [config/README.md](config/README.md)

### 3. 클러스터 설정 생성

```bash
# 환경 변수 설정
vim environment-variables.sh
# 필수 수정 항목:
# - STACK_NAME
# - KEY_PAIR_NAME
# _ CLUSTER_NAME
# - S3_BUCKET

# 커스텀 항목
# HeadNode Configuration
# LoginNode Configuration
# Compute Queue Configuration
# ComputeResource Configuration
# CustomActions Enable/Disable


# 환경 변수 로드 및 설정 생성
source environment-variables.sh
envsubst < cluster-config.yaml.template > cluster-config.yaml
```

### 4. 클러스터 생성

```bash
# 클러스터 생성 (my-cluster는 CLUSTER_NAME과 동일해야함)
pcluster create-cluster \
  --cluster-name my-cluster \
  --cluster-configuration cluster-config.yaml

# 생성 상태 확인
pcluster describe-cluster --cluster-name my-cluster
```

**클러스터 생성 중 문제가 발생한 경우**:
- 📖 **클러스터 상태 모니터링 및 로그 확인**: [아래 모니터링 섹션 참조](#클러스터-상태-모니터링)
- 📖 **로그 내보내기 상세 가이드**: [AWS ParallelCluster 로그 내보내기](https://docs.aws.amazon.com/ko_kr/parallelcluster/latest/ug/pcluster.export-cluster-logs-v3.html)

### 5. 소프트웨어 설치

세 가지 방법 중 선택하여 사용하세요:

**방법 선택 가이드**:

| 방법 | 설치 시점 | 설치 시간 | 타임아웃 위험 | 권장 용도 |
|------|-----------|-----------|---------------|-----------|
| **1. CustomActions** | 클러스터 생성 시 | 15-20분 | 중간 | 기본 GPU/CPU 환경 |
| **2. FSx 공유** | 클러스터 생성 후 | 10-15분 (1회) | 없음 | NCCL 등 대용량 라이브러리 |
| **3. 컨테이너** | 실행 시 | 즉시 | 없음 | 완전한 재현성 필요 시 |

**조합 추천**:
- 방법 1 (기본 환경) + 방법 2 (NCCL) + 방법 3 (워크로드)
- 또는 방법 3만 사용 (가장 간단)

#### 방법 1: CustomActions 자동 설치 (Timeout 방지를 위해 경량화 추천)

클러스터 생성 시 `environment-variables.sh`에서 설정:

```bash
# environment-variables.sh 설정
export COMPUTE_SETUP_TYPE="gpu"  # GPU 인스턴스용
# 또는
export COMPUTE_SETUP_TYPE="cpu"  # CPU 인스턴스용
```

**GPU 모드 (`COMPUTE_SETUP_TYPE="gpu"`)** - GPU 인스턴스용 (p5, p4d, g5, g4dn):
- Docker + Pyxis (컨테이너 실행)
- EFA Installer (고속 네트워킹)
- DCGM Exporter (GPU 메트릭)
- Node Exporter (시스템 메트릭)
- CloudWatch Agent
- 설치 시간: ~15-20분

**CPU 모드 (`COMPUTE_SETUP_TYPE="cpu"`)** - CPU 인스턴스용 (c5, m5, r5):
- Docker + Pyxis (컨테이너 실행)
- CloudWatch Agent
- 설치 시간: ~5-10분

**비활성화 (`COMPUTE_SETUP_TYPE=""`)** - 최소 설정:
- CustomActions 수행 하지 않음
- 설치 시간: ~2-3분

#### 방법 2: FSx 공유 스토리지 활용 (NCCL 설치 권장)

FSx Lustre에 한 번만 설치하고 모든 ComputeNode에서 참조:

```bash
# 1. HeadNode에 SSH 접속
ssh -i your-key.pem ubuntu@<headnode-ip>

# 2. NCCL 설치 스크립트 다운로드 (config/nccl/ 디렉토리에 있음)
# 또는 S3에서 다운로드
aws s3 cp s3://my-pcluster-scripts/config/nccl/install-nccl-shared.sh /fsx/nccl/
chmod +x /fsx/nccl/install-nccl-shared.sh

# 3. FSx에 NCCL 설치 (한 번만, 10-15분 소요)
sudo bash /fsx/nccl/install-nccl-shared.sh v2.28.7-1 v1.17.2-aws /fsx

# 설치 완료 후 생성되는 파일:
# /fsx/nccl/setup-nccl-env.sh  ← 모든 노드에서 source하여 사용
```

**ComputeNode 자동 감지**:
- ✅ 자동으로 `/fsx/nccl/setup-nccl-env.sh` 감지 및 설정
- ⚠️ **이미 실행 중인 노드**: 수동 적용 필요

```bash
# 이미 실행 중인 ComputeNode에 적용 (클러스터 생성 후 NCCL 설치한 경우)
bash /fsx/nccl/apply-nccl-to-running-nodes.sh

# 또는 수동으로 모든 노드에 적용
srun --nodes=ALL bash -c 'cat > /etc/profile.d/nccl-shared.sh << "EOF"
source /fsx/nccl/setup-nccl-env.sh
EOF
chmod +x /etc/profile.d/nccl-shared.sh'

# 적용 확인
srun --nodes=ALL bash -c 'source /etc/profile.d/nccl-shared.sh && echo "NCCL: $LD_LIBRARY_PATH"'
```

**권장 워크플로우**:
1. 클러스터 생성 (ComputeNode MinCount=0으로 설정)
2. HeadNode에서 NCCL을 FSx에 설치
3. Slurm job 제출 → ComputeNode 자동 시작 → NCCL 자동 감지 ✅

**장점**: 
- 빠른 설치 (10-15분, 한 번만)
- 스토리지 효율 (모든 노드가 공유)
- 버전 일관성 보장
- 새 노드 자동 감지
- CustomActions 타임아웃 회피

**NCCL 버전 확인**:
```bash
# 설치된 NCCL 버전 확인
ls -la /fsx/nccl/
cat /fsx/nccl/setup-nccl-env.sh
```

📖 **상세 NCCL 설치 가이드**: [config/nccl/README.md](config/nccl/README.md)  
📖 **NCCL 컨테이너 사용**: [config/nccl/README-CONTAINER.md](config/nccl/README-CONTAINER.md)  
📖 **NCCL 설치 타이밍**: [guide/NCCL-INSTALLATION-TIMING.md](guide/NCCL-INSTALLATION-TIMING.md)

#### 방법 3: 컨테이너 사용

사전 구성된 컨테이너로 소프트웨어 설치 불필요:

```bash
# Slurm job에서 컨테이너 실행
srun --container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
     --container-mounts=/fsx:/fsx \
     python /fsx/train.py
```

**장점**: 설치 불필요, 재현 가능, 버전 관리 용이

### Bootstrap 타임아웃 설정

ParallelCluster는 노드 초기화 시 CloudFormation WaitCondition을 사용하며, 기본 타임아웃은 30분입니다. GPU 인스턴스(특히 p5en.48xlarge)는 EFA 드라이버와 NVIDIA 소프트웨어 설치에 시간이 더 걸리므로 사전 테스트 후 타임아웃을 늘리시길 바랍니다.

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

### 7. NCCL 성능 테스트

```bash
# NCCL 테스트 설치 (FSx 공유 스토리지에)
bash /fsx/nccl/install-nccl-tests.sh

# 단계별 벤치마크 실행
# Phase 1: 단일 노드 기본 성능
sbatch /fsx/nccl/phase1-baseline.sbatch

# Phase 2: 멀티 노드 확장성
sbatch /fsx/nccl/phase2-multinode.sbatch

# Phase 3: 실제 워크로드 시뮬레이션
sbatch /fsx/nccl/phase3-workload.sbatch

# Phase 4: 최적화된 설정
sbatch /fsx/nccl/phase4-optimization.sbatch

# 작업 상태 확인
squeue

# 결과 확인
ls -lh /fsx/nccl-tests/results/
```

**컨테이너 기반 테스트**:
```bash
# NVIDIA PyTorch 컨테이너로 테스트
sbatch /fsx/nccl/phase1-baseline-container.sbatch
sbatch /fsx/nccl/phase3-workload-container.sbatch
sbatch /fsx/nccl/phase4-optimization-container.sbatch
```

📖 **NCCL 성능 테스트 완전 가이드**: [guide/NCCL-PERFORMANCE-TESTING.md](guide/NCCL-PERFORMANCE-TESTING.md)  
📖 **NCCL 설치 가이드**: [config/nccl/README.md](config/nccl/README.md)

## 📡 Monitoring

### 클러스터 상태 모니터링

클러스터 생성 및 운영 중 상태를 확인하고 문제를 해결하는 방법입니다.

#### 기본 상태 확인

```bash
# 클러스터 전체 상태
pcluster describe-cluster --cluster-name my-cluster

# 주요 상태 값:
# - CREATE_IN_PROGRESS: 생성 중
# - CREATE_COMPLETE: 생성 완료
# - CREATE_FAILED: 생성 실패
# - UPDATE_IN_PROGRESS: 업데이트 중
# - UPDATE_COMPLETE: 업데이트 완료
```

#### 실시간 로그 확인

```bash
# CloudWatch 로그 스트림 확인 (실시간)
pcluster get-cluster-log-events \
  --cluster-name my-cluster \
  --log-stream-name cfn-init

# 최근 1시간 로그
pcluster get-cluster-log-events \
  --cluster-name my-cluster \
  --log-stream-name cfn-init \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%S.000Z')

# 특정 노드 로그 확인
pcluster get-cluster-log-events \
  --cluster-name my-cluster \
  --log-stream-name ip-10-0-16-123.cfn-init  # 노드 IP 기반
```

#### 로그 전체 내보내기

문제 해결 시 유용한 전체 로그 다운로드:

```bash
# 모든 로그를 로컬로 다운로드
pcluster export-cluster-logs \
  --cluster-name my-cluster \
  --output-file my-cluster-logs.tar.gz

# 압축 해제 및 확인
tar -xzf my-cluster-logs.tar.gz
ls -la my-cluster-logs/

# 로그 구조:
# my-cluster-logs/
# ├── cfn-init.log           # CloudFormation 초기화
# ├── cloud-init.log         # 인스턴스 부팅
# ├── clustermgtd.log        # 클러스터 관리 데몬
# ├── slurm_resume.log       # Slurm 노드 시작
# ├── slurm_suspend.log      # Slurm 노드 중지
# └── compute/               # ComputeNode 로그
#     └── ip-10-0-16-*.log
```

**특정 기간 로그 내보내기**:
```bash
# 최근 1시간 로그만
pcluster export-cluster-logs \
  --cluster-name my-cluster \
  --output-file recent-logs.tar.gz \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%S.000Z')

# 특정 기간 로그
pcluster export-cluster-logs \
  --cluster-name my-cluster \
  --output-file period-logs.tar.gz \
  --start-time 2024-01-15T10:00:00.000Z \
  --end-time 2024-01-15T12:00:00.000Z
```

#### 로그 필터링 및 분석

```bash
# 에러 메시지 검색
pcluster get-cluster-log-events \
  --cluster-name my-cluster \
  --log-stream-name cfn-init \
  --query 'events[?contains(message, `ERROR`)]'

# 특정 키워드 검색 (예: NCCL)
pcluster get-cluster-log-events \
  --cluster-name my-cluster \
  --log-stream-name cfn-init | grep -i nccl

# 타임아웃 관련 로그 확인
tar -xzf my-cluster-logs.tar.gz
grep -r "timeout\|timed out" my-cluster-logs/
```

#### 문제 해결 체크리스트

```bash
# 1. 클러스터 상태 확인
pcluster describe-cluster --cluster-name my-cluster

# 2. CloudFormation 스택 이벤트 확인
aws cloudformation describe-stack-events \
  --stack-name my-cluster \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'

# 3. 로그 내보내기 및 분석
pcluster export-cluster-logs \
  --cluster-name my-cluster \
  --output-file debug-logs.tar.gz

# 4. 에러 메시지 검색
tar -xzf debug-logs.tar.gz
grep -r "ERROR\|FAILED\|timeout" debug-logs/
```

#### 일반적인 생성 실패 원인

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `CREATE_FAILED` | CustomActions 타임아웃 | `COMPUTE_SETUP_TYPE=""` 설정 후 재생성 |
| `CREATE_FAILED` | 용량 부족 | 다른 AZ 시도 또는 인스턴스 타입 변경 |
| `CREATE_FAILED` | IAM 권한 부족 | CloudFormation 스택 이벤트 확인 |
| ComputeNode 시작 안됨 | Slurm 설정 오류 | `sinfo`, `squeue` 확인 |
| 느린 생성 속도 | CustomActions 실행 중 | 정상, 로그로 진행 상황 확인 |

📖 **로그 내보내기 상세 가이드**: [AWS ParallelCluster 로그 내보내기](https://docs.aws.amazon.com/ko_kr/parallelcluster/latest/ug/pcluster.export-cluster-logs-v3.html)

---

### 통합 모니터링 스택

이 아키텍처는 GPU, 시스템, 네트워크 성능을 포괄하는 완전한 모니터링 스택을 제공합니다:

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

## ⚠️ 고려사항

### Capacity Block과 Placement Group

> **중요**: Capacity Block과 Placement Group은 동시에 사용할 수 없습니다.

**Capacity Block 사용 시**:
- `cluster-config.yaml`에서 `PlacementGroup.Enabled: false` 설정 필수
- Single Spine 구성이 필요한 경우 Capacity Block 예약 전 AWS Account Team에 문의
- 토폴로지 확인: [EC2 Instance Topology](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-topology.html)

**On-Demand/Spot 사용 시**:
- Placement Group 활성화 권장 (최적의 네트워크 성능)

### 인스턴스 타입 선택

**HeadNode와 LoginNode는 GPU가 필요 없습니다** - 비용 최적화를 위해 CPU 인스턴스 사용을 권장합니다.

| 노드 타입 | 권장 인스턴스 | 용도 | 비용 절감 |
|-----------|---------------|------|-----------|
| HeadNode | m5.2xlarge ~ m5.8xlarge | Slurm 스케줄러 | ~99% |
| LoginNode | m5.large ~ m5.2xlarge | 사용자 접근, 전처리 | ~99% |
| ComputeNode | p5en.48xlarge, p6-b200.48xlarge | GPU 워크로드 | - |
| Monitoring | t3.medium | 모니터링 전용 | - |

📖 **인스턴스 타입 상세 가이드**: [guide/INSTANCE-TYPE-CONFIGURATION.md](guide/INSTANCE-TYPE-CONFIGURATION.md)

### 스토리지 구성

#### 고성능 공유 스토리지
- **FSx Lustre** (`/fsx`): 데이터셋, 모델, 체크포인트
  - 멀티 GB/s 처리량
  - 병렬 I/O 최적화
  - S3 연동 가능

#### Home 디렉토리 공유

**옵션 1: HeadNode NFS** (`/home`) - 권장
- 사용자 파일, 스크립트, 환경 설정
- 추가 비용 없음
- 설정 간단
- **대부분의 경우 충분한 성능**

**옵션 2: FSx OpenZFS** (`/home`) - 특수한 경우
- 고성능 Home 디렉토리가 필요한 경우
- 많은 사용자 동시 접속 시
- 추가 비용 발생
- 설정 복잡

> 💡 **권장사항**: 특별한 요구사항이 없다면 HeadNode NFS로 충분합니다. FSx OpenZFS는 다음과 같은 경우에만 고려하세요:
> - 수십 명 이상의 사용자가 동시에 Home 디렉토리에 집중적으로 I/O 수행
> - Home 디렉토리에서 높은 IOPS가 필요한 작업 수행
> - 스냅샷, 복제 등 고급 파일시스템 기능 필요

#### 로컬 스토리지
- **EBS**: 루트 볼륨 및 로컬 스크래치
  - ComputeNode: 200GB+ 권장 (컨테이너 이미지용)
  - HeadNode: 500GB+ 권장 (로그, 패키지용)

### WaitCondition 타임아웃 관리

ParallelCluster는 노드 배포 시 CloudFormation WaitCondition을 사용하며, 기본 타임아웃은 30분입니다.

**권장 전략**:
1. ✅ **클러스터 생성 시**: 최소 설치만 수행 (빠른 배포)
   - CustomActions는 경량 작업만 (Docker, Pyxis, 모니터링)
   - NCCL 같은 대용량 설치는 제외

2. ✅ **생성 완료 후**: 필요한 소프트웨어 수동 설치
   - NCCL을 FSx에 설치하여 공유
   - 또는 컨테이너 이미지 사용

3. ✅ **공유 스토리지 활용**: 한 번 설치하여 모든 노드에서 참조
   - `/fsx/nccl/` - NCCL 라이브러리
   - `/fsx/containers/` - 컨테이너 이미지
   - `/fsx/software/` - 기타 소프트웨어

4. ✅ **컨테이너 사용**: 사전 구성된 이미지 활용
   - NVIDIA NGC 컨테이너 (PyTorch, TensorFlow 등)
   - 재현성 보장
   - 설치 시간 제로

**다수의 ComputeNode 관리**:
- ✅ **FSx 공유 스토리지 활용**: NCCL 등을 `/fsx`에 한 번만 설치하여 모든 노드에서 참조
- ✅ **Slurm job 일괄 적용**: 개별 SSH 접속 대신 `srun --nodes=ALL` 사용
- ✅ **컨테이너 사용**: Docker/Singularity로 사전 구성된 환경 배포

📖 **타임아웃 상세 가이드**: [guide/TIMEOUT-CONFIGURATION.md](guide/TIMEOUT-CONFIGURATION.md)

## 📊 예상 성능

### GPU 인스턴스 사양 예시

**p5en.48xlarge** (H100 기반):
| 항목 | 사양 |
|------|------|
| vCPUs | 192 |
| Memory | 2,048 GiB (2TB DDR5) |
| GPUs | 8x NVIDIA H100 (80GB HBM3 each) |
| Network | 3,200 Gbps (EFA) |
| NVLink | 900 GB/s per direction |
| Storage | 8x 3.84TB NVMe SSD |

**p6-b200.48xlarge** (B200 기반):
| 항목 | 사양 |
|------|------|
| vCPUs | 192 |
| Memory | 2,048 GiB (2TB DDR5) |
| GPUs | 8x NVIDIA B200 (192GB HBM3e each) |
| Network | 3,200 Gbps (EFA) |
| NVLink | 900 GB/s per direction |
| Storage | 8x 3.84TB NVMe SSD |

### NCCL 성능 지표

**단일 노드 (NVLink)**:
- AllReduce: 800-1200 GB/s (1GB 메시지)
- AllToAll: 200-400 GB/s (128MB 메시지)
- 레이턴시: <100μs (소형 메시지)

**멀티 노드 (EFA)**:
- AllReduce: >90% 확장 효율성
- 네트워크 활용: >80% of 3.2Tbps
- 레이턴시 증가: <20μs vs 단일 노드

📖 **NCCL 성능 테스트**: [guide/NCCL-PERFORMANCE-TESTING.md](guide/NCCL-PERFORMANCE-TESTING.md)

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

📖 **보안 가이드**: [security-best-practices/SECURITY.md](security-best-practices/SECURITY.md)

## 🔍 Troubleshooting

**빠른 문제 해결**:

```bash
# 클러스터 상태 확인
pcluster describe-cluster --cluster-name my-cluster

# 로그 확인
pcluster get-cluster-log-events --cluster-name my-cluster

# 설정 검증
pcluster validate-cluster-configuration --cluster-configuration cluster-config.yaml
```

## 📝 Additional Guides

- [빠른 시작 가이드](guide/QUICKSTART-EFA-MONITORING.md) - EFA 모니터링 포함 빠른 설정
- [클러스터 재생성 가이드](guide/CLUSTER-RECREATION-GUIDE.md) - 클러스터 삭제 및 재생성 절차
- [CloudWatch 모니터링 완료](guide/CLOUDWATCH-MONITORING-COMPLETE.md) - CloudWatch 통합 설정
- [선택적 컴포넌트 업데이트](guide/OPTIONAL-COMPONENTS-UPDATE.md) - 추가 기능 설치
- [변경 이력](guide/CHANGELOG-EFA-MONITORING.md) - EFA 모니터링 업데이트 내역

## 📚 Additional Resources

- [AWS ParallelCluster User Guide](https://docs.aws.amazon.com/parallelcluster/)
- [NVIDIA B200 Documentation](https://www.nvidia.com/en-us/data-center/b200/)
- [NCCL Developer Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [EFA User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)

## 📄 License

This project is licensed under the MIT-0 License.

## 🏷️ Tags

`aws` `parallelcluster` `p6` `b200` `gpu` `hpc` `machine-learning` `nccl` `slurm` `efa`
