
# Projeto DevSecOps 2025 – Servidor Web com Monitoramento

Este projeto se baseia na configuração de uma infraestrutura simples baseada em uma instância EC2 rodando Linux (Ubuntu), com:

---
1. Criar uma VPC na AWS com:

   - 2 sub-redes públicas (para acesso externo).
   - 2 sub-redes privadas (para futuras expansões).
   - Uma Internet Gateway conectada às sub-redes públicas.


---

2. Criar uma instância EC2 na AWS:
   
   - Escolher uma AMI baseada em Linux (Ubuntu/Debian/Amazon/Linux).
   - Instalar na sub-rede pública criada anteriormente.
   - Associar um Security Group que permita tráfego nas seguintes portas:
   HTTP - `(porta 80)`
   SSH  - `(porta 22)`

---

3. Acessar a instância via SSH para realizar configurações futuras.

   - Servidor web (Nginx)
   - Site HTML interativo
   - Script de monitoramento automático com alertas via Discord
   - Agendamento via `cron`

---

#  Requisitos

- AWS EC2 (Ubuntu)
- Nginx
- Bash
- Cron
- Webhook do Discord Configurado
---

#  A. Configurando o Ambiente

## 1. Criar uma VPC personalizada na AWS

- CIDR: `10.0.0.0/16`
- Sub-redes públicas:
  - `public-subnet-1` (ex: `10.0.1.0/24`)
- Criar um **Internet Gateway** e associar à VPC
- Criar uma **Route Table pública** com rota `0.0.0.0/0 → IGW` e associar às sub-redes públicas
- Criar um **Security Group** com:
  - Porta 22 liberada (SSH)
  - Porta 80 liberada (HTTP)

### Passo a Passo

   - Para iniciar o projeto, o primeiro passo é criar uma VPC personalizada na AWS com estrutura adequada para hospedar servidores web e permitir futuras expansões. A seguir está o procedimento completo.

   - Acesse o Console de Gerenciamento da AWS e vá para o serviço “VPC”.

   - Clique em “Create VPC” e selecione a opção “VPC and more” para usar o assistente de criação.

   - Defina um nome para sua VPC, como “devsecops-vpc”, e configure o bloco CIDR como 10.0.0.0/16. Deixe o restante das opções com os valores padrão e prossiga.

   - Em seguida, crie quatro sub-redes: duas públicas e duas privadas. As sub-redes públicas devem ter a opção de “Auto-assign public IPv4 address” ativada. Escolha zonas de disponibilidade diferentes para cada uma, como us-east-2a e us-east-2b,       para distribuir os recursos.

   - Crie um Internet Gateway e associe-o à VPC. Isso permitirá que as sub-redes públicas tenham acesso à internet.

   - Configure uma tabela de rotas pública, crie uma rota apontando para 0.0.0.0/0 com destino ao Internet Gateway e associe essa tabela apenas às sub-redes públicas.


## 2. Criar uma instância EC2

- AMI: Ubuntu Server 22.04 ou superior
- Tipo: `t2.micro` (elegível ao Free Tier)
- Sub-rede: pública
- IP público: habilitado (Auto-assign public IP: Enable)
- Par de chaves: baixar e salvar a `.pem`

---

#  B. Configurando o Servidor Web

### 1. Conectar na EC2 via SSH
   - É importante ressaltar que será gerado um arquivo ".pem", este será sua chave para se conectar na EC2 via terminal
```bash
ssh -i "suachave.pem" ubuntu@<ip-público>
```

### 2. Instalar o Nginx
   - Para subir o site, será necessário a instalação do Nginx na sua EC2, execute os seguintes comandos:
```bash
sudo apt update
sudo apt install nginx -y
```

### 3. Enviar arquivo HTML para EC2
   - Você pode criar um arquivo html com seu site diretamente na EC2 pelo terminal ou mover da sua máquina para ela da seguinte forma:

No seu PC local:

```bash
scp -i chave.pem "seusite.html" ubuntu@<ip-público>:/tmp/
```

Na EC2:

```bash
sudo mv /tmp/*.html /var/www/html/
```

Acesse em `http://<ip-público>` para visualizar o site.

---

#  C. Script de Monitoramento

### Script: `/usr/local/bin/monitoramento.sh`

Este script:

- Verifica se o site responde com código HTTP 200
- Registra o resultado em `/var/log/monitoramento.log`
- Envia alerta via **Discord Webhook** se detectar falha

- Você precisará seguir o seguinte passo a passo para conseguir seu link webhook do discord:
     - Crie um servidor, e vá em: Configurações do Servidor>Integrações>Webhooks>Novo Webhook
       

<p align="center">
  <img src="https://github.com/user-attachments/assets/ebcb0391-1461-4a9a-90a9-9c9e60c587e6" alt="Imagem" width="400">
</p>

   - Copie o URL do WebHook

- Além disso, é necessário mudar o fuso horário da sua EC2, execute o seguite comando:

```bash
sudo timedatectl set-timezone America/Sao_Paulo
```



### Exemplo de funcionamento:

```bash
#!/bin/bash                                                                                              
                                                                                                         
# Configurações                                                                                          
SITE="http://localhost"                                                                                  
LOG="/var/log/monitoramento.log"                                                                         
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$SITE")                                                 
DATA=$(date '+%Y-%m-%d %H:%M:%S')                                                                        

# Webhook do Discord
WEBHOOK_URL="seuwebhook"

if [ "$STATUS" -eq 200 ]; then
    echo "$DATA - OK - $SITE está online" >> "$LOG"
else
    echo "$DATA - ERRO - $SITE está offline (status: $STATUS)" >> "$LOG"
    curl -H "Content-Type: application/json" -X POST -d "{
      \"content\": \" *Alerta:* O site está fora do ar!\nStatus: offline\"
    }" "$WEBHOOK_URL"
fi

```

### Descrição do funcionamento do script:
#### Define o site a ser monitorado
  
   - O endereço do site que será verificado é armazenado em uma variável. No exemplo, é o endereço http://localhost, que representa o próprio servidor onde o script está rodando.


#### Define o caminho do arquivo de log

   - As informações de funcionamento do script (se o site está online ou offline) serão registradas em um arquivo localizado em /var/log/monitoramento.log.


#### Verifica se o site está respondendo corretamente

   - O script faz uma requisição ao site e verifica apenas o código de resposta, que é aquele número padrão de HTTP (como 200 para OK, 404 para não encontrado, etc.). Esse código é armazenado em uma variável.


#### Pega a data e hora atual

   - O script obtém a data e o horário em que a verificação está sendo feita. Isso será usado tanto no log quanto na mensagem de alerta.


#### Define o endereço do webhook do Discord

   - Um link de webhook do Discord é usado para enviar alertas. Essa URL é configurada pelo usuário.


#### Verifica se o site está online ou não

   - O script compara o código de status da requisição. Se for 200 (OK), ele registra que o site está online. Caso contrário, registra um erro no log e envia uma mensagem para o Discord informando que o site está fora do ar.


---

# ⏱️ D. Como Testar

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

#  Estrutura de Diretórios

```
/var/www/html/
├── seusite.html

/usr/local/bin/
└── monitoramento.sh

/var/log/
└── monitoramento.log
```

---

#  Notificações

As notificações são enviadas para o canal Discord via Webhook com o seguinte formato:


<p align="center">
  <img src="https://github.com/user-attachments/assets/f0a95645-ade3-44a2-b102-e2014e49f76a" alt="image" width="500">
</p>


---

##  Autor

Ruan Marra — 2025  
Projeto acadêmico DevSecOps com foco em práticas de automação, infraestrutura e monitoramento.
