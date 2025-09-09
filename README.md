# LogScope

## 📌 프로젝트 개요

운영 환경에서는 수십 개의 컨테이너와 다양한 시스템 서비스가 동시에 실행됩니다.
이 과정에서 자원 과부하, 서비스 중단, 에러 로그 누락과 같은 문제가 발생할 수 있으며, 이를 신속히 탐지하지 못하면 장애 대응이 지연됩니다.

**LogScope**는 이러한 문제를 해결하기 위해 제작된 **경량 실시간 모니터링 툴**입니다.

* Docker 컨테이너와 주요 시스템 서비스 상태를 **한 화면에서 직관적으로 확인**
* CPU 및 메모리 사용량을 **실시간 bar 그래프**로 시각화
* 최근 에러 로그를 자동 수집하여 **운영자가 즉각적으로 대응 가능**

---

## 🔎 문제 상황

1. **명령어의 단편성**

   * `docker stats`, `systemctl status`, `docker logs` 등 각각의 명령어를 따로 실행해야 함
   * 모니터링 항목이 분산되어 있어 실시간 파악이 어려움

2. **로그 추적의 어려움**

   * 에러 메시지를 찾기 위해 컨테이너별 로그 파일을 수동으로 열람해야 함
   * 서비스/컨테이너 전체에 걸친 에러 패턴을 통합적으로 보기 어려움

3. **운영 효율성 저하**

   * 다수의 컨테이너가 실행되는 환경에서는 수동 확인 방식이 비효율적
   * 장애 대응 속도가 늦어지고, 문제 재현/분석이 지연됨

---

## 🏗️ 설계 개요

### 1. 자원 사용량 수집 계층

* **컨테이너**: `docker stats --no-stream` 을 활용하여 CPU/MEM 사용량 수집
* **서비스**: `systemctl` 과 `psutil` 모듈로 서비스 활성 상태 및 PID 기반 CPU/MEM 사용량 추적

### 2. 로그 필터링 계층

* `docker logs --tail=20` 명령어 실행
* `"error"`, `"fail"`, `"exception"` 키워드 포함된 메시지만 추출
* 노이즈 제거 후 **최신 10개 로그**만 출력

### 3. 시각화 계층

* `curses` 기반 터미널 UI
* CPU/MEM 사용량을 bar 형식으로 출력 → 직관적인 자원 상태 파악 가능
* 색상 규칙 설정정

---

## ⚙️ 주요 기능 및 코드 설명

### 1. 컨테이너 모니터링 (`get_docker_stats`)

```python
["docker", "stats", "--no-stream", "--format", "{{.Name}} {{.CPUPerc}} {{.MemPerc}}"]
```

* 실행 중 컨테이너 이름, CPU%, 메모리% 출력
* CPU 사용량이 70% 이상일 경우 경고, 90% 이상이면 위험 상태 표시

### 2. 서비스 모니터링 (`get_service_stats`)

```python
["systemctl", "is-active", svc]
psutil.Process(pid).cpu_percent()
```

* `systemctl` 로 서비스 상태(활성/비활성/실패) 확인
* PID 기반으로 프로세스 CPU/MEM 사용량을 `psutil` 로 추적
* 서비스 장애를 빠르게 식별 가능

### 3. 에러 로그 수집 (`get_error_logs`)

```python
["docker", "logs", "--tail", "20", name]
```

* 컨테이너 로그 중 최근 20줄만 확인
* `"error"`, `"fail"`, `"exception"` 키워드 포함된 로그만 필터링
* **최근 10개 로그**를 출력하여 UI 하단에 표시

### 4. 실시간 터미널 UI (`monitor`)

* 1초 주기로 새로고침
* `curses` 라이브러리를 사용하여 컬러풀한 대시보드 제공
* 자원 사용량 bar 그래프 출력
* 서비스/컨테이너 상태에 따라 색상 다르게 표시

---

## 🚀 설치 및 실행

### 1. 저장소 클론

```bash
git clone https://github.com/username/logscope.git
cd logscope
```

### 2. 설치 (Makefile 기반)

```bash
make install
```

👉 이후 `logscope` 명령어 실행 가능

### 3. 실행

```bash
logscope
```

<img width="980" height="517" alt="image" src="https://github.com/user-attachments/assets/80c02979-0e15-4da4-b608-209a9d29adeb" />

---

## 📂 프로젝트 구조

```
logscope/
 ├── logscope.py      # 핵심 모니터링 코드
 ├── Makefile         # 빌드 및 설치 스크립트
 ├── README.md        # 프로젝트 문서
```

---

## 🔧 Makefile

```makefile
PREFIX=/usr/local/bin

install:
	cp logscope.py $(PREFIX)/logscope
	chmod +x $(PREFIX)/logscope
	@echo "✅ logscope installed. Run 'logscope' to start."

uninstall:
	rm -f $(PREFIX)/logscope
	@echo "❌ logscope removed."
```
---
# ⚙️ 주요 기능 (코드 설명 포함)

## 1. 컨테이너 모니터링 (`get_docker_stats`)
```python
["docker", "stats", "--no-stream", "--format", "{{.Name}} {{.CPUPerc}} {{.MemPerc}}"]
````

* 모든 실행 중 컨테이너의 이름, CPU%, MEM% 수집
* CPU 70% 이상일 경우 **경고**, 90% 이상일 경우 **위험** 표시

---

## 2. Pod 모니터링 (`get_pod_stats`)

```python
["kubectl", "get", "pods", "-o", "jsonpath={.items[*].metadata.name}"]
```

* Pod 이름과 실행 상태(Running, Pending, Failed) 표시
* CPU/MEM 사용량은 확장 시 **metrics-server 연동**을 통해 추가 가능

---

## 3. 서비스 모니터링 (`get_service_stats`)

```python
["systemctl", "is-active", svc]
psutil.Process(pid).cpu_percent()
```

* `systemctl` 을 사용해 서비스 상태(활성/비활성/실패) 조회
* `psutil` 모듈을 활용하여 **PID 단위 CPU/MEM 사용량** 정밀 측정

---

## 4. 에러 로그 수집 (`get_error_logs`)

```bash
docker logs --tail=20
kubectl logs --tail=20
```

* 각 컨테이너/Pod의 최근 20개 로그 확인
* `"error"`, `"fail"`, `"exception"` 키워드가 포함된 로그만 추출
* 최종적으로 **최근 10개 에러 로그**만 출력

---

## 5. 터미널 UI (`monitor`)

* `curses` 기반 실시간 UI
* CPU/MEM 사용량을 **bar 그래프**로 시각화
* 색상 코드:

  * 🟢 초록: 정상
  * 🟡 노랑: 경고 (CPU > 70%)
  * 🔴 빨강: 위험 (CPU > 90% 또는 서비스 실패)
  * 🔵 파랑: 헤더 및 구분선
* 1초 주기로 화면 자동 갱신

---

## 📌 실행 예시

<img width="980" height="517" alt="image" src="https://github.com/user-attachments/assets/69b8f9b2-c2d2-40de-9989-939ca72cebb4" />

---

# 📌 향후 개선 사항

## 1. 시각화 강화
- 컨테이너, 서비스, 로그의 **상태/종류별 글자 색상** 세분화  
  - 컨테이너:  
    - `running` → 🟢 초록  
    - `exited` → 🔴 빨강  
  - 서비스:  
    - `active` → 🟢 초록  
    - `inactive` → ⚪ 회색  
    - `failed` → 🔴 빨강  
  - 로그 메시지:  
    - `"error"` → 🔴 빨강  
    - `"warning"` → 🟡 노랑  
    - `"info"` → 🔵 파랑  

## 2. 로그 처리 개선
- 에러 로그 **키워드별 하이라이팅** 적용  
- Docker / 시스템 로그 **필터링 규칙** 추가  
- 에러 발생 시 즉시 화면 최상단에 알림 표시  


## 5. 확장성
- 웹 기반 대시보드 제공 (Flask, FastAPI 연동)  
- Prometheus / Grafana 등과 연계하여 **시각화 대시보드 확장**  
- 멀티 노드 환경 지원 (여러 서버의 상태를 한 화면에서 모니터링)  

---

# 📝 프로젝트 회고

이번 프로젝트는 단순한 로그 수집/출력 기능에서 출발했지만, 점차 발전시켜 **실시간 모니터링 툴**로서의 성격을 강화하는 방향으로 개발을 이어갔다.  

## 🔹 발전 과정
- **초기 단계**에서는 단순히 `docker logs`를 모아 보여주는 수준이었다.  
- 이후 컨테이너 CPU/MEM 사용률을 실시간으로 확인하기 위해 **`docker stats` 연동**을 추가했다.  
- 시스템 서비스(`systemctl`) 상태와 자원 사용량을 확인할 수 있도록 **서비스 모니터링 기능**을 개발했다.  
- 단순 출력만으로는 직관성이 떨어져, `curses` 기반 **터미널 UI**를 도입해 시각적인 가독성을 확보했다.  
- 에러 로그는 단순 출력에서 벗어나, **에러 키워드 기반 필터링**과 최근 로그 10개만 보여주는 방식으로 개선했다.  

## 🔹 느낀 점
이전 버전에서는 단순한 로그 모음에 불과했지만, 이번 개발 과정을 통해 **실제 운영 환경에서 즉시 활용할 수 있는 수준**으로 발전시킬 수 있었다.  
특히, 사용자의 시각적 편의성과 직관성을 높이는 방향으로 점진적으로 개선하면서 **"툴을 만든다" → "운영에 도움이 되는 툴로 발전시킨다"**는 경험을 할 수 있었다.  

앞으로는 단순 모니터링을 넘어서, 알림 전송이나 장기적인 성능 분석까지 연결되는 **운영 자동화 툴**로 확장해보고 싶다.  

---


