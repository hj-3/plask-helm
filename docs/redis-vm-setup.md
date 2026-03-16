# Redis VM 설정 가이드

VM: 10.84.101.160 — Redis 2개 인스턴스 운영

## 1. Redis 설치

```bash
sudo apt update && sudo apt install -y redis-server
sudo systemctl stop redis-server
sudo systemctl disable redis-server   # 기본 서비스 비활성화
```

## 2. 인스턴스별 데이터 디렉토리 생성

```bash
sudo mkdir -p /var/lib/redis/session
sudo mkdir -p /var/lib/redis/queue
sudo chown -R redis:redis /var/lib/redis
```

## 3. Redis Session 설정 (포트 6379)

```bash
sudo cp /etc/redis/redis.conf /etc/redis/redis-session.conf
sudo vi /etc/redis/redis-session.conf
```

변경 항목:
```conf
port 6379
bind 0.0.0.0
daemonize no
supervised systemd
dir /var/lib/redis/session
logfile /var/log/redis/redis-session.log
pidfile /var/run/redis/redis-session.pid
# 클러스터 내부망에서만 접근 — 필요 시 requirepass 설정
# requirepass your-session-redis-password
save 900 1
save 300 10
save 60 10000
```

## 4. Redis Queue 설정 (포트 6380)

```bash
sudo cp /etc/redis/redis.conf /etc/redis/redis-queue.conf
sudo vi /etc/redis/redis-queue.conf
```

변경 항목:
```conf
port 6380
bind 0.0.0.0
daemonize no
supervised systemd
dir /var/lib/redis/queue
logfile /var/log/redis/redis-queue.log
pidfile /var/run/redis/redis-queue.pid
# 큐는 영속성 불필요 (주문 처리 후 소멸)
save ""
appendonly no
```

## 5. systemd 서비스 등록

### Redis Session 서비스

```bash
sudo cat > /etc/systemd/system/redis-session.service << 'EOF'
[Unit]
Description=Redis Session Store (port 6379)
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/redis-session.conf
ExecStop=/usr/bin/redis-cli -p 6379 shutdown
Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### Redis Queue 서비스

```bash
sudo cat > /etc/systemd/system/redis-queue.service << 'EOF'
[Unit]
Description=Redis Order Queue (port 6380)
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/redis-queue.conf
ExecStop=/usr/bin/redis-cli -p 6380 shutdown
Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable redis-session redis-queue
sudo systemctl start redis-session redis-queue
sudo systemctl status redis-session redis-queue
```

## 6. 방화벽 설정

K8s 워커 노드 IP 대역(10.84.101.110~130)에서만 접근 허용:

```bash
# ufw 사용 시
sudo ufw allow from 10.84.101.0/24 to any port 6379
sudo ufw allow from 10.84.101.0/24 to any port 6380
sudo ufw reload

# iptables 사용 시
sudo iptables -A INPUT -s 10.84.101.0/24 -p tcp --dport 6379 -j ACCEPT
sudo iptables -A INPUT -s 10.84.101.0/24 -p tcp --dport 6380 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP
sudo iptables -A INPUT -p tcp --dport 6380 -j DROP
```

## 7. 연결 확인 (K8s 워커 노드에서)

```bash
redis-cli -h 10.84.101.160 -p 6379 ping   # PONG
redis-cli -h 10.84.101.160 -p 6380 ping   # PONG
```

## 8. 구조 요약

```
VM 10.84.101.160
├── redis-session (포트 6379) — JWT 세션 캐시
│   └── K8s Service: redis-session-svc:6379 → Endpoints → 10.84.101.160:6379
└── redis-queue (포트 6380) — FIFO 주문 큐
    └── K8s Service: redis-queue-svc:6380 → Endpoints → 10.84.101.160:6380
```
