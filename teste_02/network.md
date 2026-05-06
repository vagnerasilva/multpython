# SageMaker Studio Network Auditor

Este repositório contém a solução completa para monitoramento e auditoria de conexões externas e cross-account dentro do ambiente SageMaker Studio (Ubuntu).

## 📋 Resumo da Solução
- **Monitoramento em Background:** Captura DNS e conexões TCP (SYN) via `tcpdump`.
- **Enriquecimento de Dados:** Vincula cada conexão ao e-mail e profile do usuário via `governance.env`.
- **Filtros Inteligentes:** Separa o que é Internet, APIs AWS e conexões Cross-Account, ignorando o ruído da VPC local.
- **Governança de Logs:** Rotação automática para manter 90 dias de logs localmente.

---

## 🛠️ Arquivo: network_monitor.py

```python
import subprocess
import re
import datetime
import os
import logging
from logging.handlers import TimedRotatingFileHandler

# --- CONFIGURAÇÕES ---
LOG_FILE = "/home/sagemaker-user/audit_network.log"
ENV_FILE = "/home/sagemaker-user/.local/governance.env"
MY_VPC_CIDR = r"^(10\.0\.)"            # Ajuste para o range da sua VPC local
CROSS_ACCOUNT_NETWORKS = r"^(10\.50\.|192\.168\.)" # Ranges de outras contas (Peering/TGW)

# --- CONFIGURAÇÃO DE ROTAÇÃO (90 DIAS) ---
# Rotaciona diariamente ("D"), a cada 1 dia, mantém os últimos 90 arquivos.
handler = TimedRotatingFileHandler(LOG_FILE, when="D", interval=1, backupCount=90)
logger = logging.getLogger("AuditLogger")
logger.setLevel(logging.INFO)
logger.addHandler(handler)

def get_user_info():
    """Extrai identidade do arquivo governance.env"""
    info = {"email": "unknown", "profile": "unknown"}
    if os.path.exists(ENV_FILE):
        try:
            with open(ENV_FILE, "r") as e:
                content = e.read()
                email = re.search(r"EMAIL=(.*)", content)
                profile = re.search(r"USER_PROFILE=(.*)", content)
                if email: info["email"] = email.group(1).strip().replace('"', '')
                if profile: info["profile"] = profile.group(1).strip().replace('"', '')
        except Exception: pass
    return info

# Inicialização de contexto do usuário
user = get_user_info()
user_tag = f"USER: {user['email']} | PROFILE: {user['profile']}"

# Comando tcpdump: Captura queries DNS e pacotes de início de conexão (SYN)
cmd = ["sudo", "tcpdump", "-i", "any", "-n", "-l", "port 53 or (tcp[tcpflags] & tcp-syn != 0)"]

print(f"Iniciando auditoria para {user['email']}...")

# Execução em subprocesso capturando STDOUT de forma line-buffered
process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)

for line in iter(process.stdout.readline, ""):
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    # 1. PROCESSAMENTO DE DNS (Nome do site ou API)
    if "A?" in line:
        match = re.search(r"A\? ([\w\.\-]+)", line)
        if match:
            domain = match.group(1)
            # Diferencia APIs AWS de sites externos
            tag = "AWS_API_CALL" if "amazonaws.com" in domain else "EXTERNAL_DOMAIN"
            logger.info(f"{now} | {user_tag} | {tag} | DOMAIN: {domain}")
    
    # 2. PROCESSAMENTO DE CONEXÕES TCP (IP e Porta de saída)
    elif "Flags [S]" in line:
        match = re.search(r"IP ([\d\.]+) > ([\d\.]+)\.(\d+): Flags \[S\]", line)
        if match:
            dst_ip = match.group(2)
            dst_port = match.group(3)
            
            # Lógica de classificação de destino
            if re.match(CROSS_ACCOUNT_NETWORKS, dst_ip):
                event = "CROSS_ACCOUNT_CONN"
            elif re.match(MY_VPC_CIDR, dst_ip):
                continue # Ignora tráfego interno da própria conta
            else:
                event = "EXTERNAL_CONN"
            
            logger.info(f"{now} | {user_tag} | {event} | DST_IP: {dst_ip} | PORT: {dst_port}")
```

---

## ⚙️ Instruções de Setup

### 1. Dockerfile
Adicione estas linhas à sua imagem customizada para garantir permissões:
```dockerfile
USER root
RUN apt-get update && apt-get install -y tcpdump python3-pip
# SUID bit permite que o script rode o tcpdump sem pedir senha de sudo
RUN chmod +s /usr/sbin/tcpdump
```

### 2. Lifecycle Configuration (LCC)
Crie um script de LCC do tipo "Start Notebook" para disparar o monitor:
```bash
#!/bin/bash
# Garante que o script python seja iniciado de forma independente
nohup python3 /home/sagemaker-user/network_monitor.py > /dev/null 2>&1 &
```

### 3. Exemplo de Log Gerado
O arquivo `/home/sagemaker-user/audit_network.log` terá este padrão:
```text
2024-05-10 15:00:01 | USER: joao@empresa.com | PROFILE: prf-01 | EXTERNAL_DOMAIN | DOMAIN: ://openai.com
2024-05-10 15:00:02 | USER: joao@empresa.com | PROFILE: prf-01 | EXTERNAL_CONN | DST_IP: 104.18.7.192 | PORT: 443
2024-05-10 15:05:00 | USER: joao@empresa.com | PROFILE: prf-01 | CROSS_ACCOUNT_CONN | DST_IP: 10.50.5.11 | PORT: 5432
```
---
# SageMaker Studio Network Auditor (v2 - Connection Timing)

Este script monitora o início e o fim de conexões TCP, calculando a duração total e registrando no log de auditoria.

---

## 🛠️ Arquivo: network_monitor.py

```python
import subprocess
import re
import datetime
import os
import logging
from logging.handlers import TimedRotatingFileHandler

# --- CONFIGURAÇÕES ---
LOG_FILE = "/home/sagemaker-user/audit_network.log"
ENV_FILE = "/home/sagemaker-user/.local/governance.env"
MY_VPC_CIDR = r"^(10\.0\.)"            
CROSS_ACCOUNT_NETWORKS = r"^(10\.50\.|192\.168\.)"

# Dicionário para rastrear conexões ativas: {(src_ip, src_port, dst_ip, dst_port): start_time}
active_connections = {}

# --- CONFIGURAÇÃO DE ROTAÇÃO (90 DIAS) ---
handler = TimedRotatingFileHandler(LOG_FILE, when="D", interval=1, backupCount=90)
logger = logging.getLogger("AuditLogger")
logger.setLevel(logging.INFO)
logger.addHandler(handler)

def get_user_info():
    info = {"email": "unknown", "profile": "unknown"}
    if os.path.exists(ENV_FILE):
        try:
            with open(ENV_FILE, "r") as e:
                content = e.read()
                email = re.search(r"EMAIL=(.*)", content)
                profile = re.search(r"USER_PROFILE=(.*)", content)
                if email: info["email"] = email.group(1).strip().replace('"', '')
                if profile: info["profile"] = profile.group(1).strip().replace('"', '')
        except Exception: pass
    return info

user = get_user_info()
user_tag = f"USER: {user['email']} | PROFILE: {user['profile']}"

# tcpdump captura: DNS (53), SYN (Início), FIN/RST (Fim)
cmd = ["sudo", "tcpdump", "-i", "any", "-n", "-l", "port 53 or (tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0)"]

process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)

for line in iter(process.stdout.readline, ""):
    now = datetime.datetime.now()
    timestamp = now.strftime("%Y-%m-%d %H:%M:%S")
    
    # 1. PROCESSAMENTO DE DNS
    if "A?" in line:
        match = re.search(r"A\? ([\w\.\-]+)", line)
        if match:
            domain = match.group(1)
            tag = "AWS_API_CALL" if "amazonaws.com" in domain else "EXTERNAL_DOMAIN"
            logger.info(f"{timestamp} | {user_tag} | {tag} | DOMAIN: {domain}")
    
    # 2. PROCESSAMENTO DE TCP (START / END)
    elif "Flags [" in line:
        # Extrai IPs e Portas: IP 10.0.1.5.12345 > 1.2.3.4.443
        match = re.search(r"IP ([\d\.]+)\.(\d+) > ([\d\.]+)\.(\d+): Flags \[([SFP\.]+)\]", line)
        if match:
            src_ip, src_port, dst_ip, dst_port, flag = match.groups()
            conn_key = (src_ip, src_port, dst_ip, dst_port)

            # FILTRO: Ignorar tráfego interno da VPC
            if re.match(MY_VPC_CIDR, dst_ip) and not re.match(CROSS_ACCOUNT_NETWORKS, dst_ip):
                continue

            # INÍCIO DA CONEXÃO (Flag [S] = SYN)
            if "S" in flag:
                active_connections[conn_key] = now
                event = "CROSS_ACCOUNT_CONN" if re.match(CROSS_ACCOUNT_NETWORKS, dst_ip) else "EXTERNAL_CONN"
                logger.info(f"{timestamp} | {user_tag} | {event}_START | DST_IP: {dst_ip} | PORT: {dst_port}")

            # FIM DA CONEXÃO (Flag [F] = FIN ou [R] = RST)
            elif any(f in flag for f in ["F", "R"]):
                if conn_key in active_connections:
                    start_time = active_connections.pop(conn_key)
                    duration = (now - start_time).total_seconds()
                    event = "CROSS_ACCOUNT_CONN" if re.match(CROSS_ACCOUNT_NETWORKS, dst_ip) else "EXTERNAL_CONN"
                    logger.info(f"{timestamp} | {user_tag} | {event}_END | DST_IP: {dst_ip} | DURATION: {duration}s")
```

---

## 📈 Exemplo de Log com Tempo de Conexão
```text
2024-05-10 15:00:01 | USER: joao@... | PROFILE: prf-01 | EXTERNAL_DOMAIN | DOMAIN: ://openai.com
2024-05-10 15:00:02 | USER: joao@... | PROFILE: prf-01 | EXTERNAL_CONN_START | DST_IP: 104.18.7.192 | PORT: 443
2024-05-10 15:00:15 | USER: joao@... | PROFILE: prf-01 | EXTERNAL_CONN_END | DST_IP: 104.18.7.192 | DURATION: 13.5s
```

### Notas sobre esta versão:
1.  **Chave de Conexão:** Usamos o par IP/Porta de origem e destino para garantir que o tempo de fim corresponda exatamente ao início daquela sessão específica.
2.  **Duração:** O tempo é calculado em segundos. Se a conexão durar menos de 1 segundo, aparecerá como `0.x s`.
3.  **Memória:** O dicionário `active_connections` limpa as entradas automaticamente ao detectar o fim da conexão para não consumir memória desnecessária.

Deseja que eu adicione algum **limite de tempo (timeout)** para limpar conexões que nunca enviam o pacote de fim (ex: conexões "zumbis")?

# SageMaker Studio Network Auditor (v3 - Domain Link & Timing)

Este script monitora o ciclo de vida completo: Resolve o DNS, inicia a conexão e, no fim, reporta a duração vinculada ao domínio original.

---

## 🛠️ Arquivo: network_monitor.py

```python
import subprocess
import re
import datetime
import os
import logging
from logging.handlers import TimedRotatingFileHandler

# --- CONFIGURAÇÕES ---
LOG_FILE = "/home/sagemaker-user/audit_network.log"
ENV_FILE = "/home/sagemaker-user/.local/governance.env"
MY_VPC_CIDR = r"^(10\.0\.)"            
CROSS_ACCOUNT_NETWORKS = r"^(10\.50\.|192\.168\.)"

# Rastreio de Conexões e DNS
# active_conns: {(src_port, dst_ip, dst_port): {"start": time, "domain": str}}
active_conns = {}
dns_cache = {} # Mapeia IP -> Domínio (para saber quem é o IP no fim da conexão)

# --- CONFIGURAÇÃO DE ROTAÇÃO (90 DIAS) ---
handler = TimedRotatingFileHandler(LOG_FILE, when="D", interval=1, backupCount=90)
logger = logging.getLogger("AuditLogger")
logger.setLevel(logging.INFO)
logger.addHandler(handler)

def get_user_info():
    info = {"email": "unknown", "profile": "unknown"}
    if os.path.exists(ENV_FILE):
        try:
            with open(ENV_FILE, "r") as e:
                content = e.read()
                email = re.search(r"EMAIL=(.*)", content)
                profile = re.search(r"USER_PROFILE=(.*)", content)
                if email: info["email"] = email.group(1).strip().replace('"', '')
                if profile: info["profile"] = profile.group(1).strip().replace('"', '')
        except Exception: pass
    return info

user = get_user_info()
u_tag = f"USER: {user['email']} | PROFILE: {user['profile']}"

# Captura DNS (53), SYN, FIN e RST
cmd = ["sudo", "tcpdump", "-i", "any", "-n", "-l", "port 53 or (tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0)"]

process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)

for line in iter(process.stdout.readline, ""):
    now = datetime.datetime.now()
    ts = now.strftime("%Y-%m-%d %H:%M:%S")
    
    # 1. CAPTURA DNS E MAPEIA IP
    if " A? " in line or " A " in line:
        # Tenta capturar a resposta DNS (IP que o domínio resolveu)
        dns_match = re.search(r"A\? ([\w\.\-]+)", line)
        ip_match = re.search(r"A ([\d\.]+)$", line.strip())
        
        if dns_match:
            current_query = dns_match.group(1)
        if ip_match and 'current_query' in locals():
            dns_cache[ip_match.group(1)] = current_query

    # 2. PROCESSAMENTO TCP (START/END)
    elif "Flags [" in line:
        m = re.search(r"IP ([\d\.]+)\.(\d+) > ([\d\.]+)\.(\d+): Flags \[([SFP\.]+)\]", line)
        if m:
            src_ip, src_p, dst_ip, dst_p, flag = m.groups()
            conn_key = (src_p, dst_ip, dst_p) # Porta origem + Destino identifica a sessão
            domain = dns_cache.get(dst_ip, "unknown-domain")

            if re.match(MY_VPC_CIDR, dst_ip) and not re.match(CROSS_ACCOUNT_NETWORKS, dst_ip):
                continue

            # START
            if "S" in flag:
                active_conns[conn_key] = {"start": now, "domain": domain}
                logger.info(f"{ts} | {u_tag} | START | DOMAIN: {domain} | DST: {dst_ip}:{dst_p}")

            # END
            elif any(f in flag for f in ["F", "R"]):
                if conn_key in active_conns:
                    c = active_conns.pop(conn_key)
                    dur = (now - c["start"]).total_seconds()
                    logger.info(f"{ts} | {u_tag} | END   | DOMAIN: {c['domain']} | DST: {dst_ip}:{dst_p} | DURATION: {dur}s")
```

---

### Como ler o novo log:
```text
2024-05-10 15:00:01 | USER: joao@... | START | DOMAIN: api.openai.com | DST: 104.18.7.192:443
2024-05-10 15:00:15 | USER: joao@... | END   | DOMAIN: api.openai.com | DST: 104.18.7.192:443 | DURATION: 14.2s
```

### O que mudou:
1.  **Vínculo Inteligente:** O script observa a resposta do DNS e guarda que o IP `104.18.7.192` pertence à `openai.com`.
2.  **Consolidação:** Você não tem mais uma linha solta de DNS e outra de IP. O log de `START` e `END` já traz o domínio amarrado à conexão.
3.  **Precisão:** Se o usuário baixar um dataset gigante do Kaggle, o log de `END` só aparecerá quando o download terminar, mostrando a duração total (ex: `DURATION: 320.5s`).

Deseja que eu adicione uma **limpeza periódica** no cache de domínios para evitar que ele cresça muito se o usuário ficar dias com o ambiente ligado?

