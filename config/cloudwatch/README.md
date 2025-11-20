# ParallelCluster CloudWatch 모니터링

분산학습 클러스터를 위한 종합 모니터링 솔루션입니다.

## 📋 목차

- [빠른 시작](#빠른-시작)
- [대시보드 구성](#대시보드-구성)
- [설치 방법](#설치-방법)
- [인스턴스 타입별 설정](#인스턴스-타입별-설정)
- [수집되는 메트릭](#수집되는-메트릭)
- [파일 구조](#파일-구조)
- [대시보드 기능 상세](#대시보드-기능-상세)
- [트러블슈팅](#트러블슈팅)

## 🚀 빠른 시작 (3분)

### 1단계: S3 배포
```bash
cd parallelcluster-for-llm
source environment-variables-bailey.sh
bash config/cloudwatch/deploy-to-s3.sh
```

### 2단계: 클러스터 생성 (모든 것이 자동)
```bash
pcluster create-cluster \
    --cluster-name ${CLUSTER_NAME} \
    --cluster-configuration cluster-config.yaml
```

**자동으로 수행되는 작업:**
- ✅ CloudWatch Agent 설치 (모든 노드)
- ✅ Slurm 메트릭 수집기 설치 (HeadNode)
- ✅ DCGM/Node Exporter 설치 (ComputeNode, GPU 모드)
- ✅ **대시보드 자동 생성** (HeadNode에서 백그라운드)

### 3단계: 대시보드 확인 (1-2분 후)

대시보드는 HeadNode 시작 후 자동으로 생성됩니다 (약 1-2분 소요).

```bash
# 대시보드 생성 로그 확인
ssh headnode
tail -f /var/log/dashboard-creation.log
```

**대시보드 URL:**
```
https://console.aws.amazon.com/cloudwatch/home?region=us-east-2#dashboards:
```

**수동 생성 (필요시):**
```bash
# 로컬에서 실행
bash config/cloudwatch/create-dashboard.sh ${CLUSTER_NAME} ${AWS_REGION}
bash config/cloudwatch/create-advanced-dashboard.sh ${CLUSTER_NAME} ${AWS_REGION}
```

## 📊 대시보드 구성

### 기본 대시보드 (13개 위젯)
인프라 관리자와 모델 학습자 모두를 위한 종합 모니터링:
- CPU/메모리/디스크 사용률
- 네트워크 및 FSx Lustre I/O
- Slurm 로그 (에러, resume, suspend)
- GPU 모니터링 (DCGM)
- 클러스터 관리 로그

### 고급 대시보드 (12개 위젯)
Slurm 작업 큐 및 노드 상태 실시간 모니터링:
- Slurm 노드 상태 (Total/Idle/Allocated/Down)
- 작업 큐 상태 (Running/Pending/Total)
- 노드 활용률 계산
- 작업 완료/실패 로그
- GPU 상태 모니터링

## 🔧 설치 방법

### 자동 설치 (권장)

클러스터 생성 시 자동으로 설치됩니다:

- **HeadNode**: CloudWatch Agent + Slurm 메트릭 수집기 + Prometheus
- **ComputeNode**: CloudWatch Agent + DCGM Exporter (선택) + Node Exporter (선택)
- **LoginNode**: CloudWatch Agent

### 수동 설치

필요한 경우 수동으로 설치할 수 있습니다:

```bash
# HeadNode에서
aws s3 cp s3://${S3_BUCKET}/config/cloudwatch/install-cloudwatch-agent.sh /tmp/
bash /tmp/install-cloudwatch-agent.sh ${CLUSTER_NAME} ${AWS_REGION} ${S3_BUCKET}

aws s3 cp s3://${S3_BUCKET}/config/cloudwatch/install-slurm-metrics.sh /tmp/
bash /tmp/install-slurm-metrics.sh ${CLUSTER_NAME} ${AWS_REGION} ${S3_BUCKET}

# ComputeNode에서
aws s3 cp s3://${S3_BUCKET}/config/cloudwatch/install-cloudwatch-agent.sh /tmp/
bash /tmp/install-cloudwatch-agent.sh ${CLUSTER_NAME} ${AWS_REGION} ${S3_BUCKET}
```

## 🔧 인스턴스 타입별 설정

Compute node 타입에 따라 설치할 컴포넌트를 선택할 수 있습니다.

### 빠른 설정

```bash
# environment-variables-bailey.sh

# GPU 인스턴스 (p5, p4d, g5, g4dn)
export COMPUTE_SETUP_TYPE="gpu"

# CPU 인스턴스 (c5, m5, r5)
export COMPUTE_SETUP_TYPE="cpu"

# 최소 설정 (테스트)
export COMPUTE_SETUP_TYPE=""
```

| 설정 | 설치 항목 | 모니터링 |
|------|-----------|----------|
| `"gpu"` | Docker + Pyxis + EFA + DCGM + Node Exporter | ✅ 전체 |
| `"cpu"` | Docker + Pyxis | ⚠️ CloudWatch만 |
| `""` | 없음 | ⚠️ CloudWatch 기본만 |

**상세 가이드**: [인스턴스 타입별 설정 가이드](../../guide/INSTANCE-TYPE-CONFIGURATION.md)

## 📈 수집되는 메트릭

### CloudWatch Agent (자동 수집)
- **CPU**: usage_idle, usage_iowait
- **Memory**: used_percent, available, used
- **Disk**: used_percent, free, used, I/O
- **Network**: tcp_established, tcp_time_wait
- **Swap**: used_percent

### Slurm 메트릭 (1분마다 수집)
- **NodesTotal**: 전체 노드 수
- **NodesIdle**: 유휴 노드
- **NodesAllocated**: 작업 실행 중 노드
- **NodesDown**: 장애 노드
- **JobsRunning**: 실행 중인 작업
- **JobsPending**: 대기 중인 작업
- **JobsTotal**: 전체 작업 수

### 로그 수집 (7개 로그 그룹)
- `/var/log/slurmctld.log` → `/aws/parallelcluster/${CLUSTER_NAME}/slurm`
- `/var/log/slurmd.log` → `/aws/parallelcluster/${CLUSTER_NAME}/slurm`
- `/var/log/parallelcluster/slurm_resume.log` → `/aws/parallelcluster/${CLUSTER_NAME}/slurm-resume`
- `/var/log/parallelcluster/slurm_suspend.log` → `/aws/parallelcluster/${CLUSTER_NAME}/slurm-suspend`
- `/var/log/dcgm/nv-hostengine.log` → `/aws/parallelcluster/${CLUSTER_NAME}/dcgm`
- `/var/log/nvidia-installer.log` → `/aws/parallelcluster/${CLUSTER_NAME}/nvidia`
- `/var/log/parallelcluster/clustermgtd` → `/aws/parallelcluster/${CLUSTER_NAME}/clustermgtd`

## 📁 파일 구조

```
config/cloudwatch/
├── README.md                          # 이 파일
├── cloudwatch-agent-config.json       # CloudWatch Agent 설정
├── install-cloudwatch-agent.sh        # CloudWatch Agent 설치
├── slurm-metrics-collector.sh         # Slurm 메트릭 수집 (cron)
├── install-slurm-metrics.sh           # Slurm 메트릭 수집기 설치
├── create-dashboard.sh                # 기본 대시보드 생성
├── create-advanced-dashboard.sh       # 고급 대시보드 생성
└── deploy-to-s3.sh                    # S3 배포 스크립트
```

## 🎨 대시보드 기능 상세

### 기본 대시보드 위젯

#### 1. 클러스터 CPU 사용률
- HeadNode와 Compute Nodes의 CPU 사용률
- 과부하 감지 (CPU > 90%)
- 5분 평균값

#### 2. 메모리 사용률
- 전체 노드의 메모리 사용률
- OOM 위험 감지 (Memory > 95%)
- 실시간 모니터링

#### 3. Slurm 에러 로그
- 최근 50개 에러 메시지
- 작업 실패 원인 분석
- 실시간 업데이트

#### 4. 네트워크 트래픽
- EFA 네트워크 활용 확인
- 분산 학습 통신 모니터링
- NetworkIn/NetworkOut

#### 5. 디스크 사용률
- 디스크 공간 부족 경고 (> 85%)
- 로그 파일 증가 추적
- 체크포인트 저장 공간 확인

#### 6. GPU 모니터링 (DCGM)
- GPU 에러 감지
- GPU 온도/전력 모니터링
- GPU 메모리 사용률

#### 7. FSx Lustre I/O
- 공유 스토리지 성능
- 데이터셋 로딩 속도
- 병목 현상 감지

### 고급 대시보드 위젯

#### 1. Slurm 노드 상태
```
Total: 10 nodes
Idle: 3 nodes (30%)
Allocated: 6 nodes (60%)
Down: 1 node (10%)
```

#### 2. Slurm 작업 큐 상태
```
Running: 15 jobs
Pending: 5 jobs (대기 중)
Total: 20 jobs
```

#### 3. 노드 활용률
- 계산식: `(NodesAllocated / NodesTotal) * 100`
- 목표: 70-90% (최적 활용률)
- 비용 효율성 분석

## 🔍 모니터링 확인

### CloudWatch Agent 상태 확인

HeadNode 또는 ComputeNode에서:

```bash
# Agent 상태
sudo systemctl status amazon-cloudwatch-agent

# Agent 로그
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Slurm 메트릭 확인

HeadNode에서:

```bash
# 메트릭 수집 로그
tail -f /var/log/slurm-metrics.log

# 수동 실행 테스트
sudo /usr/local/bin/slurm-metrics-collector.sh ${CLUSTER_NAME} ${AWS_REGION}
```

### CloudWatch 메트릭 확인

```bash
# Slurm 메트릭 확인
aws cloudwatch get-metric-statistics \
    --namespace "ParallelCluster/${CLUSTER_NAME}/Slurm" \
    --metric-name NodesTotal \
    --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 60 \
    --statistics Average \
    --region ${AWS_REGION}
```

## 🛠️ 트러블슈팅

### 문제: 대시보드에 데이터가 없음

**해결 방법:**

1. CloudWatch Agent 상태 확인:
```bash
ssh headnode
sudo systemctl status amazon-cloudwatch-agent
```

2. Slurm 메트릭 수집기 확인:
```bash
ssh headnode
tail -f /var/log/slurm-metrics.log
```

3. IAM 권한 확인:
```bash
# HeadNode IAM 역할에 CloudWatchAgentServerPolicy가 있는지 확인
aws iam list-attached-role-policies --role-name <HeadNode-Role-Name>
```

### 문제: Slurm 메트릭이 표시되지 않음

**해결 방법:**

1. Cron job 확인:
```bash
ssh headnode
cat /etc/cron.d/slurm-metrics
```

2. 수동 실행 테스트:
```bash
ssh headnode
sudo /usr/local/bin/slurm-metrics-collector.sh ${CLUSTER_NAME} ${AWS_REGION}
```

3. CloudWatch에 메트릭이 전송되었는지 확인:
```bash
aws cloudwatch list-metrics \
    --namespace "ParallelCluster/${CLUSTER_NAME}/Slurm" \
    --region ${AWS_REGION}
```

### 문제: GPU 메트릭이 표시되지 않음

**해결 방법:**

1. DCGM Exporter 상태 확인:
```bash
ssh compute-node
sudo systemctl status dcgm-exporter
```

2. Prometheus가 메트릭을 수집하는지 확인:
```bash
ssh headnode
curl http://localhost:9090/api/v1/targets
```

### 문제: 대시보드가 자동 생성되지 않음

**해결 방법:**

1. 대시보드 생성 로그 확인:
```bash
ssh headnode
tail -f /var/log/dashboard-creation.log
```

2. 수동으로 대시보드 생성:
```bash
# 로컬에서 실행
bash config/cloudwatch/create-dashboard.sh ${CLUSTER_NAME} ${AWS_REGION}
bash config/cloudwatch/create-advanced-dashboard.sh ${CLUSTER_NAME} ${AWS_REGION}
```

## 💡 팁

### 대시보드 커스터마이징
`create-dashboard.sh`를 수정하여 원하는 메트릭 추가

### 알람 설정
CloudWatch Alarms를 사용하여 임계값 초과 시 알림:
```bash
aws cloudwatch put-metric-alarm \
    --alarm-name high-cpu-usage \
    --alarm-description "Alert when CPU exceeds 80%" \
    --metric-name CPU_IDLE \
    --namespace "ParallelCluster/${CLUSTER_NAME}" \
    --statistic Average \
    --period 300 \
    --threshold 20 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 2
```

### 로그 쿼리
CloudWatch Logs Insights로 고급 로그 분석:
```
# Slurm 작업 실패 분석
fields @timestamp, @message
| filter @message like /FAILED|ERROR/
| stats count() by bin(5m)
```

### 비용 최적화
- 로그 보관 기간: 7일 (기본값, `cloudwatch-agent-config.json`에서 변경 가능)
- 메트릭 수집 주기: 60초 (필요시 조정)
- 불필요한 로그 필터링

## 📚 관련 문서

- [인스턴스 타입별 설정 가이드](../../guide/INSTANCE-TYPE-CONFIGURATION.md)
- [클러스터 설정 가이드](../README.md)
- [ParallelCluster 모니터링](https://docs.aws.amazon.com/parallelcluster/latest/ug/cloudwatch-logs.html)
- [CloudWatch Agent 설정](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
