
# Projeto DevSecOps 2025 – Servidor Web com Monitoramento

Este projeto configura uma infraestrutura simples baseada em uma instância EC2 rodando Linux (Ubuntu), com:

- Servidor web (Nginx)
- Site HTML interativo (imagem + som)
- Script de monitoramento automático com alertas via Discord
- Agendamento via `cron`

---

## 🛠️ a. Como configurar o ambiente

### 1. Criar uma VPC personalizada na AWS

- CIDR: `10.0.0.0/16`
- Sub-redes públicas:
  - `public-subnet-1` (ex: `10.0.1.0/24`)
- Criar um **Internet Gateway** e associar à VPC
- Criar uma **Route Table pública** com rota `0.0.0.0/0 → IGW` e associar às sub-redes públicas
- Criar um **Security Group** com:
  - Porta 22 liberada (SSH)
  - Porta 80 liberada (HTTP)

### 2. Criar uma instância EC2

- AMI: Ubuntu Server 22.04 ou superior
- Tipo: `t2.micro` (elegível ao Free Tier)
- Sub-rede: pública
- IP público: habilitado (Auto-assign public IP: Enable)
- Par de chaves: baixar e salvar a `.pem`

---

## 🌐 b. Como instalar e configurar o servidor web

### 1. Conectar na EC2 via SSH

```bash
ssh -i chave.pem ubuntu@<ip-público>
```

### 2. Instalar o Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### 3. Enviar arquivos HTML, imagem e som

No seu PC local:

```bash
scp -i chave.pem index.html imagem.jpg som.mp3 ubuntu@<ip-público>:/tmp/
```

Na EC2:

```bash
sudo mv /tmp/*.html /tmp/*.jpg /tmp/*.mp3 /var/www/html/
```

Acesse em `http://<ip-público>` para visualizar o site.

---

## 📟 c. Como funciona o script de monitoramento

### Script: `/usr/local/bin/monitoramento.sh`

Este script:

- Verifica se o site responde com código HTTP 200
- Registra o resultado em `/var/log/monitoramento.log`
- Envia alerta via **Discord Webhook** se detectar falha

#### Exemplo de funcionamento:

```bash
#!/bin/bash

SITE="http://localhost"
LOG="/var/log/monitoramento.log"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$SITE")
DATA=$(date '+%Y-%m-%d %H:%M:%S')

WEBHOOK_URL="https://discordapp.com/api/webhooks/..."

if [ "$STATUS" -eq 200 ]; then
    echo "$DATA - OK - $SITE está online" >> "$LOG"
else
    echo "$DATA - ERRO - $SITE está offline (status: $STATUS)" >> "$LOG"
    curl -H "Content-Type: application/json" -X POST -d "{
      \"content\": \"🚨 *Alerta:* O site $SITE está fora do ar!\\nStatus: $STATUS\\nData: $DATA\"
    }" "$WEBHOOK_URL"
fi
```

---

## ⏱️ d. Como testar e validar a solução

### 1. Testar o script manualmente:

```bash
sudo /usr/local/bin/monitoramento.sh
cat /var/log/monitoramento.log
```

### 2. Validar execução automática com `cron`

Adicionar no crontab do root:

```bash
sudo crontab -e
```

E incluir:

```bash
* * * * * /usr/local/bin/monitoramento.sh >> /var/log/monitoramento.log 2>&1
```

Isso executará o monitoramento a **cada minuto**.

### 3. Simular falha

- Pare o Nginx:

```bash
sudo systemctl stop nginx
```

- Aguarde 1 minuto
- Veja a notificação no Discord

---

## 📦 Estrutura de diretórios

```
/var/www/html/
├── index.html
├── imagem.jpg
└── som.mp3

/usr/local/bin/
└── monitoramento.sh

/var/log/
└── monitoramento.log
```

---

## 📬 Notificações

As notificações são enviadas para o canal Discord via Webhook com o seguinte formato:

```
🚨 *Alerta:* O site http://localhost está fora do ar!
Status: 000
Data: 2025-06-27 14:35:00
```

---

## 📌 Requisitos

- AWS EC2 (Ubuntu)
- Nginx
- Bash
- `cron`
- Webhook do Discord configurado

---

## ✨ Autor

Ruan Marra — 2025  
Projeto acadêmico DevSecOps com foco em práticas de automação, infraestrutura e monitoramento.
