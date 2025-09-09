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
* 색상 규칙:

  * ✅ 정상(녹색)
  * ⚠️ 경고(노란색, CPU > 70%)
  * 🔴 위험(빨강, CPU > 90% 또는 서비스 실패)

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

## 📌 기대 효과

* **단일 명령어**로 컨테이너 및 서비스 상태 확인 가능
* 로그 수집 및 분석을 자동화하여 **문제 탐지 시간 단축**
* 운영자가 **실시간 UI 기반 모니터링**으로 장애 상황을 직관적으로 파악
