### 0. PostgreSQL (EDB EPAS)

psql -p 5444 -U enterprisedb
```psql
CREATE USER openwebui WITH PASSWORD 'change_me_strong_password';
CREATE DATABASE openwebui OWNER openwebui;
GRANT ALL PRIVILEGES ON DATABASE openwebui TO openwebui;
```

psql -d openwebui -p 5444 -U enterprisedb
```
create extension vector;
ALTER DATABASE openwebui SET datestyle TO 'ISO, MDY';
```
### 1. uv 설치
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Open WebUI 실행 (설치+실행 한번에, 3.11 고정)
```
export DATA_DIR=/home/ec2-user/OpenWebUI/.open-webui
export VECTOR_DB=pgvector
export DATABASE_URL="postgresql://openwebui:change_me_strong_password@localhost:5444/openwebui"
export WEBUI_SECRET_KEY="dd27cd9747e3526233ca607f70ae470a2b3e5fddc9e4e7d87e9d33552b2b7343"
export OLLAMA_BASE_URL="http://localhost:11434"

export HOST=0.0.0.0
export PORT=8080
uvx --python 3.11 --from "open-webui[postgres]" open-webui serve
```

### 참고. WEBUI_SECRET_KEY. 안전한 랜덤 문자열 생성
```
openssl rand -hex 32
```
예시 출력: dd27cd9747e3526233ca607f70ae470a2b3e5fddc9e4e7d87e9d33552b2b7343

```
export WEBUI_SECRET_KEY="dd27cd9747e3526233ca607f70ae470a2b3e5fddc9e4e7d87e9d33552b2b7343"
uvx --python 3.11 --from "open-webui[postgres]" open-webui serve
```

### 확인 : 로그에 아래와 비슷한 메시지가 뜨면 정상
"Connected to PostgreSQL database" <br>

또는 직접 테이블 생성 여부 확인 <br>
```
psql -h localhost -U openwebui -d openwebui -c "\dt" -p 5444
```
user, chat, auth, config 등의 테이블이 보이면 정상적으로 Postgres를 백엔드로 쓰고 있는 겁니다.

### System Service 등록
```
sudo tee /etc/systemd/system/open-webui.service <<EOF
[Unit]
Description=Open WebUI (uv, PostgreSQL backend)
After=network.target postgresql.service

[Service]
Type=simple
User=openwebui
WorkingDirectory=/home/ec2-user/OpenWebUI
Environment="DATA_DIR=/home/ec2-user/OpenWebUI/.open-webui"
Environment="VECTOR_DB=pgvector"
Environment="DATABASE_URL=postgresql://openwebui:change_me_strong_password@localhost:5432/openwebui"
Environment="WEBUI_SECRET_KEY=dd27cd9747e3526233ca607f70ae470a2b3e5fddc9e4e7d87e9d33552b2b7343"
Environment="OLLAMA_BASE_URL=http://localhost:11434"
Environment="PATH=/home/ec2-user/.local/bin:/usr/bin:/bin"
Environment="HOST=0.0.0.0"
Environment="PORT=8080"

ExecStart=/home/ec2-user/.local/bin/uvx --python 3.11 --from "open-webui[postgres]" open-webui serve
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now open-webui
sudo journalctl -u open-webui -f     # 로그 확인
```

### 서비스 접속.
http://Public-IP:8080

