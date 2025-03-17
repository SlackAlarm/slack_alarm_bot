# 📀 **Slack_Alarm_Bot**

**nginx** 및 **MySQL** 프로세스의 리소스를 모니터링하고, 임계치를 초과할 경우 **Slack**으로 자동 알람을 보내는 시스템을 구성하였습니다.

---
## 👨‍👨‍👦‍👦 팀원 소개  
| <img src="https://github.com/wns5120.png" width="200px"> | <img src="https://github.com/Aunsxm.png" width="200px"> | <img src="https://github.com/andytjdqls.png" width="200px"> |
| :---: | :---: | :---: |
| [유호준](https://github.com/wns5120) | [장수현](https://github.com/Aunsxm) | [이성빈](https://github.com/andytjdqls) |

---

## 🎯 **프로젝트 목표**
이 프로젝트는 **자동화된 서버 모니터링**을 통해 **실시간 대응력을 높이고**, IT 인프라를 효과적으로 관리하며, 기록된 데이터를 기반으로 추후 활용할 수 있도록 하는 것을 목표로 합니다.

🚀 **목표**
- 서버 프로세스 리소스(**Nginx, MySQL**)의 **CPU 및 메모리 사용량**을 모니터링
- **임계값 초과 시 Slack 알람** 전송하여 실시간 대응 가능
- 주기적인 **로그 기록**을 통해 데이터 활용
- IT 인프라 관점에서 **자동화 및 유지보수 효율성** 향상

---

## 🔑 **핵심 키워드**
- `crontab` (주기적 실행)
- `grep` (정규표현식 사용)
- `uptime` (서버 가동 시간 체크)

---

## 🚨 **알람**
| CPU 알람 | 메모리 알람 |Mysql|
|:------:|:------:|:------:|
|<img src="https://github.com/user-attachments/assets/a6bd628c-a6fc-4ce2-90ba-e85d42ecb1ed" width="250"> | <img src="https://github.com/user-attachments/assets/a4605223-c362-4812-bb06-5e7e22a37f4a" width="250"> | <img src="https://github.com/user-attachments/assets/fa824902-a506-439c-b446-35c47d2f7d42" width="250">|




## 📌 **구성 과정**
1. `manager` 새로운 유저 계정 생성 및 로그 디렉토리 생성
2. `process_monitor.sh` (Nginx) & `process_monitor_mysql.sh` (MySQL) 스크립트 작성 및 실행 권한 부여
3. `crontab`을 활용하여 자동 실행 설정 (1분마다 실행)
4. `wrk`을 이용한 부하 테스트 및 성능 점검

---

## 🔹 **1. 사용자 및 로그 디렉토리 설정**
먼저, 모니터링 스크립트를 실행할 관리 계정(`manager`)을 생성하고, 로그를 저장할 디렉토리를 생성

```bash
# manager 사용자 추가 및 sudo 권한 부여
sudo adduser manager
id manager
sudo usermod -aG sudo manager

# nginx 설치 및 확인
sudo apt update
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl status nginx

# mysql 설치 및 확인
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql
sudo systemctl status mysql

# 로그 저장을 위한 디렉토리 및 파일 생성
sudo mkdir -p /var/log/manager_logs
sudo touch /var/log/manager_logs/process_monitor.log
sudo touch /var/log/manager_logs/process_monitor_mysql.log

# 권한 설정
sudo chmod 777 /var/log/manager_logs
sudo chmod 777 /var/log/manager_logs/process_monitor.log
sudo chmod 777 /var/log/manager_logs/process_monitor_mysql.log
sudo chown manager:manager /var/log/manager_logs/process_monitor.log
sudo chown manager:manager /var/log/manager_logs/process_monitor_mysql.log
```

---

## 🔹 **2. 리소스 모니터링 스크립트 작성**

### ✅ **Nginx 모니터링 스크립트**
```bash
nano /home/manager/process_monitor.sh
```

📌 **`process_monitor.sh` (Nginx 리소스 모니터링)**
```bash
#!/bin/bash
LOG_FILE="/var/log/manager_logs/process_monitor.log"
PROCESS="nginx: worker process"
CPU_THRESHOLD=10.0  
MEM_THRESHOLD=10700  

SLACK_WEBHOOK_URL="https://hooks.slack.com/services/XXXXXXXXX/XXXXXXXXX/XXXXXXXXXXXX"

echo "$(date '+%Y-%m-%d %H:%M:%S') - Checking $PROCESS resource usage..." >> $LOG_FILE
PIDS=$(pgrep -f "$PROCESS")

if [ -z "$PIDS" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Process $PROCESS not running" >> $LOG_FILE
    exit 1
fi

TOTAL_CPU=0
TOTAL_MEM=0

for PID in $PIDS; do
    CPU_USAGE=$(ps -p "$PID" -o %cpu --no-headers | awk '{print $1}')
    MEM_USAGE=$(ps -p "$PID" -o rss --no-headers | awk '{print $1}')
    TOTAL_CPU=$(echo "$TOTAL_CPU + $CPU_USAGE" | bc)
    TOTAL_MEM=$(echo "$TOTAL_MEM + $MEM_USAGE" | bc)
done

echo "$(date '+%Y-%m-%d %H:%M:%S') - Total CPU: $TOTAL_CPU% | Total Memory: $TOTAL_MEM KB" >> $LOG_FILE

send_slack_alert() {
    curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"$1\"}" $SLACK_WEBHOOK_URL
}

if (( $(echo "$TOTAL_CPU > $CPU_THRESHOLD" | bc -l) )); then
    send_slack_alert "🔥 *High CPU Usage Alert!* Process: $PROCESS | CPU: $TOTAL_CPU%"
fi

if [ "$TOTAL_MEM" -ge "$MEM_THRESHOLD" ]; then
    send_slack_alert "🔥 *High Memory Usage Alert!* Process: $PROCESS | Memory: $TOTAL_MEM KB"
fi
```

### ✅ **MySQL 모니터링 스크립트**
```bash
nano /home/manager/process_monitor_mysql.sh
```

📌 **`process_monitor_mysql.sh` (MySQL 리소스 모니터링)**
```bash
#!/bin/bash
LOG_FILE="/var/log/manager_logs/process_monitor_mysql.log"
PROCESS="mysqld"
CPU_THRESHOLD=10.0  
MEM_THRESHOLD=10700  

SLACK_WEBHOOK_URL="https://hooks.slack.com/services/XXXXXXXXX/XXXXXXXXX/XXXXXXXXXXXX"

echo "$(date '+%Y-%m-%d %H:%M:%S') - Checking $PROCESS resource usage..." >> $LOG_FILE
PIDS=$(pgrep -f "$PROCESS")

if [ -z "$PIDS" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Process $PROCESS not running" >> $LOG_FILE
    exit 1
fi

TOTAL_CPU=0
TOTAL_MEM=0

for PID in $PIDS; do
    CPU_USAGE=$(ps -p "$PID" -o %cpu --no-headers | awk '{print $1}')
    MEM_USAGE=$(ps -p "$PID" -o rss --no-headers | awk '{print $1}')
    TOTAL_CPU=$(echo "$TOTAL_CPU + $CPU_USAGE" | bc)
    TOTAL_MEM=$(echo "$TOTAL_MEM + $MEM_USAGE" | bc)
done

echo "$(date '+%Y-%m-%d %H:%M:%S') - Total CPU: $TOTAL_CPU% | Total Memory: $TOTAL_MEM KB" >> $LOG_FILE

send_slack_alert() {
    curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"$1\"}" $SLACK_WEBHOOK_URL
}

if (( $(echo "$TOTAL_CPU > $CPU_THRESHOLD" | bc -l) )); then
    send_slack_alert "🔥 *High CPU Usage Alert!* Process: $PROCESS | CPU: $TOTAL_CPU%"
fi

if [ "$TOTAL_MEM" -ge "$MEM_THRESHOLD" ]; then
    send_slack_alert "🔥 *High Memory Usage Alert!* Process: $PROCESS | Memory: $TOTAL_MEM KB"
fi
```

---

## 🔹 **3. Crontab 자동 실행 설정**
```bash
crontab -e
```
다음 내용을 추가:
```
* * * * * /bin/bash /home/manager/process_monitor.sh
* * * * * /bin/bash /home/manager/process_monitor_mysql.sh
```

---

## 🔹 **4. 부하 테스트 (wrk 활용)**
```bash
sudo apt install wrk -y
wrk -t4 -c1000 -d100s http://localhost/
```

---

## 🛠 **트러블슈팅**

| 문제 | 원인 | 해결 방법 |
|:------:|:------:|:----------:|
| **Slack 알람이 전송되지 않음** | `SLACK_WEBHOOK_URL`이 올바르지 않거나, 네트워크 문제 발생 | `SLACK_WEBHOOK_URL`이 정확한지 확인 |
| **`crontab`이 실행되지 않음** | `crontab`이 제대로 등록되지 않았거나, 실행 권한 문제 발생 | `crontab -l` 명령어로 등록된 작업 확인, `/var/log/syslog`에서 cron 실행 여부 확인 `sudo grep process_monitor.sh /var/log/syslog` , `chmod +x`로 실행 권한 부여 후 재등록   |
| **MySQL 프로세스가 감지되지 않음** | `mysqld` 프로세스가 예상과 다른 위치에서 실행 중이거나, 서비스가 중지됨 | `pgrep -a mysqld` 또는 `systemctl status mysql` 실행 후 MySQL 서비스가 실행 중인지 확인하고, `pgrep -f "mysqld"`로 프로세스 탐색 |
| **로그 파일이 생성되지 않음** | 로그 디렉토리 또는 파일에 대한 권한 문제 | `/var/log/manager_logs/`와 `/var/log/manager_logs/process_monitor.log`의 권한 확인 (`chmod 777` 및 `chown manager:manager` 설정) |
| **Nginx의 PID가 여러 개 발견됨 (Master, Worker 존재)** | Nginx는 **Master 프로세스 1개** + **Worker 프로세스 여러 개**로 구성됨 | `pgrep -f "nginx: worker process"`를 사용하여 **Worker 프로세스만 모니터링**하도록 설정 |
| **Nginx의 모든 Worker 프로세스의 CPU 및 메모리 사용량 합산 필요** | `pgrep`로 찾은 각각의 PID 사용량을 개별적으로 체크해야 함 | `for PID in $(pgrep -f "nginx: worker process"); do ... done` 루프를 사용하여 CPU 및 메모리 사용량을 모두 합산 |
| **Nginx 또는 MySQL이 재시작될 때 PID 변경됨** | 프로세스가 재시작될 때마다 새로운 PID가 할당됨 | 매번 `pgrep`을 실행하여 현재 실행 중인 프로세스를 다시 감지하도록 스크립트 수정 |

---

## 🪄 앞으로 추가할 기능
| 기능 | 설명 | 기대 효과 |
|------|------|----------|
| ELK Stack 연동 | Elasticsearch, Logstash, Kibana를 이용한 로그 시각화 | 장애 발생 원인 분석 및 실시간 대시보드 제공 |
| MySQL 쿼리 성능 모니터링 | mysqld_exporter와 Slow Query Log 분석 | 비효율적인 쿼리 탐색 및 최적화 |
| Nginx 상태 점검 API 연동 | stub_status 모듈을 활용하여 Nginx 상태 조회 | Active Connections, Request 처리량 등 모니터링 가능 |
| Slack 알람 세분화 | 경고 수준별 알람 (INFO, WARNING, CRITICAL) | 운영자의 불필요한 알람 피로도를 줄이고, 중요한 알람만 즉각 대응 가능 |
| Failover 시스템 구축 | 장애 발생 시 백업 서버로 트래픽 자동 전환 | 서비스 연속성 유지 |
| 백업 및 복구 자동화 | MySQL 데이터 백업 + 주기적인 스냅샷 저장 | 장애 발생 시 신속한 복구 가능 |

