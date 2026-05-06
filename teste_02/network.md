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
