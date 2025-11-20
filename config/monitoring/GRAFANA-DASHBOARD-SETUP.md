# Grafana 대시보드 설정 가이드

## 빠른 시작

Grafana Workspace가 생성되었지만 대시보드가 비어있습니다. 다음 방법으로 대시보드를 추가하세요.

## 방법 1: 커뮤니티 대시보드 Import (추천 ⭐)

### 1. Node Exporter Full (시스템 메트릭)

**Grafana UI에서:**
1. **Dashboards** → **New** → **Import**
2. Dashboard ID: **`1860`** 입력
3. **Load** 클릭
4. Data source: **Amazon Managed Service for Prometheus** 선택
5. **Import** 클릭

**포함된 메트릭:**
- ✅ CPU 사용률 (전체, 코어별)
- ✅ Memory 사용률 (Used, Free, Cached)
- ✅ Disk I/O (Read/Write)
- ✅ Network Traffic (In/Out)
- ✅ System Load (1m, 5m, 15m)
- ✅ Filesystem Usage
- ✅ Process Count

**대시보드 링크:** https://grafana.com/grafana/dashboards/1860

---

### 2. NVIDIA DCGM Exporter (GPU 메트릭)

**Grafana UI에서:**
1. **Dashboards** → **New** → **Import**
2. Dashboard ID: **`12239`** 입력
3. **Load** 클릭
4. Data source: **Amazon Managed Service for Prometheus** 선택
5. **Import** 클릭

**포함된 메트릭:**
- ✅ GPU Utilization (%)
- ✅ GPU Memory Usage (Used/Total)
- ✅ GPU Temperature (°C)
- ✅ GPU Power Usage (W)
- ✅ GPU Clock Speed (MHz)
- ✅ PCIe Throughput (TX/RX)
- ✅ NVLink Throughput
- ✅ GPU Errors

**대시보드 링크:** https://grafana.com/grafana/dashboards/12239

---

### 3. Prometheus Stats (Prometheus 모니터링)

**Grafana UI에서:**
1. **Dashboards** → **New** → **Import**
2. Dashboard ID: **`2`** 입력
3. **Load** 클릭
4. Data source: **Amazon Managed Service for Prometheus** 선택
5. **Import** 클릭

**포함된 메트릭:**
- ✅ Prometheus 상태
- ✅ Scrape 성공률
- ✅ 메트릭 수집 지연
- ✅ 저장된 샘플 수

**대시보드 링크:** https://grafana.com/grafana/dashboards/2

---

## 방법 2: 커스텀 대시보드 Import

이 레포지토리에 포함된 간단한 대시보드를 사용하세요.

### ParallelCluster Overview Dashboard

**파일 위치:** `config/monitoring/parallelcluster-dashboard.json`

**Import 방법:**
1. **Dashboards** → **New** → **Import**
2. **Upload JSON file** 클릭
3. `parallelcluster-dashboard.json` 파일 선택
4. Data source: **Amazon Managed Service for Prometheus** 선택
5. **Import** 클릭

**포함된 패널:**
- CPU Usage (전체 노드)
- Memory Usage (전체 노드)
- GPU Utilization (전체 GPU)
- GPU Temperature (전체 GPU)
- Active Compute Nodes (카운트)
- Total GPUs (카운트)

---

## 방법 3: 직접 대시보드 만들기

### 새 대시보드 생성

1. **Dashboards** → **New Dashboard**
2. **Add visualization** 클릭
3. Data source: **Amazon Managed Service for Prometheus** 선택
4. Query 입력 (아래 예시 참고)
5. **Apply** 클릭

### 유용한 PromQL 쿼리 예시

#### 시스템 메트릭

```promql
# CPU 사용률 (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory 사용률 (%)
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))

# Disk 사용률 (%)
100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)

# Network 수신 (bytes/s)
rate(node_network_receive_bytes_total{device!="lo"}[5m])

# Network 송신 (bytes/s)
rate(node_network_transmit_bytes_total{device!="lo"}[5m])

# System Load (1분 평균)
node_load1

# 활성 노드 수
count(up{job="compute-nodes"} == 1)
```

#### GPU 메트릭

```promql
# GPU 사용률 (%)
DCGM_FI_DEV_GPU_UTIL

# GPU 메모리 사용률 (%)
(DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_FREE) * 100

# GPU 온도 (°C)
DCGM_FI_DEV_GPU_TEMP

# GPU 전력 소비 (W)
DCGM_FI_DEV_POWER_USAGE

# GPU 메모리 사용량 (MB)
DCGM_FI_DEV_FB_USED

# GPU 클럭 속도 (MHz)
DCGM_FI_DEV_SM_CLOCK

# 총 GPU 수
count(DCGM_FI_DEV_GPU_UTIL)

# GPU별 평균 사용률
avg by (gpu, instance) (DCGM_FI_DEV_GPU_UTIL)
```

#### Prometheus 메트릭

```promql
# Scrape 성공률
rate(prometheus_target_scrapes_sample_out_of_order_total[5m])

# 수집된 샘플 수
rate(prometheus_tsdb_head_samples_appended_total[5m])

# 활성 타겟 수
count(up == 1)

# 실패한 타겟 수
count(up == 0)
```

---

## 데이터가 보이지 않을 때

### 1. 클러스터가 생성되었는지 확인

```bash
pcluster describe-cluster --cluster-name YOUR_CLUSTER_NAME
```

**Status가 `CREATE_COMPLETE`여야 합니다.**

### 2. HeadNode에서 Prometheus 상태 확인

```bash
# HeadNode SSH 접속
pcluster ssh --cluster-name YOUR_CLUSTER_NAME -i ~/.ssh/key.pem

# Prometheus 상태 확인
sudo systemctl status prometheus

# Prometheus 로그 확인
sudo journalctl -u prometheus -n 50

# remote_write 설정 확인
grep -A 10 "remote_write" /etc/prometheus/prometheus.yml
```

### 3. ComputeNode가 실행 중인지 확인

```bash
# Slurm 노드 상태 확인
sinfo

# 예상 출력:
# PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
# gpu          up   infinite      2   idle compute-dy-gpu-[1-2]
```

### 4. Grafana Explore에서 직접 쿼리

**Grafana UI에서:**
1. **Explore (🔍)** 메뉴 클릭
2. Data source: **Amazon Managed Service for Prometheus** 선택
3. Query 입력: `up`
4. **Run query** 클릭

**예상 결과:**
```
up{instance="10.0.1.100:9100", job="compute-nodes"} 1
up{instance="10.0.1.101:9100", job="compute-nodes"} 1
up{instance="10.0.1.100:9400", job="dcgm"} 1
up{instance="10.0.1.101:9400", job="dcgm"} 1
```

**`up` 값이 1이면 정상, 0이면 문제 있음**

---

## 알림 설정 (선택사항)

### SNS Topic 생성

```bash
# SNS Topic 생성
aws sns create-topic --name pcluster-alerts --region YOUR_REGION

# 이메일 구독
aws sns subscribe \
  --topic-arn arn:aws:sns:YOUR_REGION:YOUR_ACCOUNT:pcluster-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com

# 이메일 확인 (받은 메일에서 "Confirm subscription" 클릭)
```

### Grafana 알림 채널 설정

**Grafana UI에서:**
1. **Alerting** → **Notification channels** → **New channel**
2. **Name**: `SNS Alerts`
3. **Type**: `AWS SNS`
4. **Topic ARN**: `arn:aws:sns:YOUR_REGION:YOUR_ACCOUNT:pcluster-alerts`
5. **Auth Provider**: `AWS SDK Default`
6. **Save**

### 알림 규칙 예시

**GPU 온도 알림:**
```promql
DCGM_FI_DEV_GPU_TEMP > 85
```

**노드 다운 알림:**
```promql
up{job="compute-nodes"} == 0
```

**높은 메모리 사용률 알림:**
```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 90
```

**GPU 메모리 부족 알림:**
```promql
(DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_FREE) * 100 > 95
```

---

## 추가 리소스

### 커뮤니티 대시보드 검색

- **Grafana Dashboards**: https://grafana.com/grafana/dashboards/
- **검색 키워드**: `node exporter`, `nvidia`, `dcgm`, `gpu`, `prometheus`

### 추천 대시보드

| Dashboard | ID | 설명 |
|-----------|-----|------|
| Node Exporter Full | 1860 | 완전한 시스템 메트릭 |
| NVIDIA DCGM Exporter | 12239 | GPU 메트릭 |
| Prometheus Stats | 2 | Prometheus 자체 모니터링 |
| Node Exporter for Prometheus | 11074 | 간단한 시스템 메트릭 |
| NVIDIA GPU Metrics | 14574 | 대체 GPU 대시보드 |

### 문서

- [Grafana 문서](https://grafana.com/docs/grafana/latest/)
- [PromQL 가이드](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [DCGM Exporter 메트릭](https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html)
- [Node Exporter 메트릭](https://github.com/prometheus/node_exporter#enabled-by-default)

---

## 요약

**빠른 시작 (3분):**
1. Grafana 접속
2. Dashboard ID `1860` Import (시스템 메트릭)
3. Dashboard ID `12239` Import (GPU 메트릭)
4. 완료! 🎉

**데이터가 없다면:**
- 클러스터가 생성되었는지 확인
- HeadNode + ComputeNode가 실행 중인지 확인
- 5-10분 정도 기다리면 데이터 수집 시작
