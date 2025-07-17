# Guia Definitivo e Realista: FreePBX 17 no Raspberry Pi 5 (Debian 12)  
## The Definitive & Realistic Guide: FreePBX 17 on Raspberry Pi 5 (Debian 12)

Este guia consolida o passo a passo completo e funcional para instalar o FreePBX 17 em um Raspberry Pi 5 com Debian 12.  
Ele inclui todas as dependências e correções de permissão para se obter um sistema operacional estável.  
Ao final, é documentado um bug conhecido de instabilidade com o UCP Daemon e a solução de contorno para garantir um sistema estável.

*This guide consolidates the complete and functional step-by-step process for installing FreePBX 17 on a Raspberry Pi 5 with Debian 12. It includes all necessary dependency and permission fixes. Finally, it documents a known stability bug with the UCP Daemon and the workaround to ensure a stable system.*

---

## ✅ Pré-requisitos / Prerequisites

- Um Raspberry Pi 5 com uma instalação limpa do **Raspberry Pi OS Lite (64-bit)**  
  *A Raspberry Pi 5 with a clean installation of **Raspberry Pi OS Lite (64-bit)**.*

- Acesso à internet configurado e funcionando  
  *Internet access configured and working on the device*

---

## 🛠️ Fase 1: Preparação do Sistema / Phase 1: System Preparation

1. **Atualize completamente o sistema / Fully update the system:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Instale todas as dependências de uma só vez / Install all dependencies at once:**
   ```bash
   sudo apt install -y build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev libsrtp2-dev uuid-dev sox nodejs npm incron
   ```

---

## ⚙️ Fase 2: Configuração dos Serviços Base / Phase 2: Base Services Configuration

1. **Instale Apache e MariaDB / Install Apache and MariaDB:**
   ```bash
   sudo apt install -y apache2 mariadb-server
   ```

2. **Instale o PHP 8.2 e suas extensões / Install PHP 8.2 and its extensions:**
   ```bash
   sudo apt install -y php php-mysql php-cli php-common php-gd php-mbstring php-xml php-curl php-json php-pear php-bcmath php-zip
   ```

3. **Proteja a instalação do banco de dados / Secure the database installation:**
   ```bash
   sudo mysql_secure_installation
   ```
   > Crie uma **senha de root forte** e responda **Sim (Y)** para todas as perguntas.  
   > *Create a **strong root password** and answer **Yes (Y)** to all questions.*

4. **Corrija o bug de crash do UCP comentando o `bind-address`:**
   ```bash
   sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```
   > Comente a linha:
   > ```text
   > #bind-address = 127.0.0.1
   > ```

5. **Configure o Apache para rodar como o usuário `asterisk`:**
   ```bash
   sudo nano /etc/apache2/envvars
   ```
   > Altere as seguintes linhas:
   > ```text
   > export APACHE_RUN_USER=asterisk
   > export APACHE_RUN_GROUP=asterisk
   > ```

6. **Ajuste outras configurações do Apache e PHP:**
   ```bash
   sudo a2enmod rewrite
   sudo sed -i 's/\(AllowOverride \).*/\1All/' /etc/apache2/apache2.conf
   sudo sed -i 's/\(^upload_max_filesize = \).*/\1120M/' /etc/php/8.2/apache2/php.ini
   sudo sed -i 's/\(^post_max_size = \).*/\1128M/' /etc/php/8.2/apache2/php.ini
   sudo systemctl restart apache2
   sudo systemctl restart mariadb.service
   ```

---

## 🔧 Fase 3 e 4: Asterisk e FreePBX / Phases 3 & 4: Asterisk and FreePBX

> As instruções para compilar o Asterisk e clonar o FreePBX via Git permanecem idênticas ao guia oficial.  
> *The steps to compile Asterisk and clone FreePBX remain the same as in the official guide.*

---

## ✅ Fase 5: Ajustes Pós-Instalação / Post-Install Adjustments

1. **Instale o IonCube Loader:**
   ```bash
   cd /tmp
   sudo wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_aarch64.tar.gz
   sudo tar -xzvf ioncube_loaders_lin_aarch64.tar.gz
   sudo cp /tmp/ioncube/ioncube_loader_lin_8.2.so /usr/lib/php/20220829/
   echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/apache2/conf.d/00-ioncube.ini
   echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/cli/conf.d/00-ioncube.ini
   sudo systemctl restart apache2
   ```

2. **Instale módulos adicionais do FreePBX:**
   ```bash
   cd /usr/src/freepbx
   sudo fwconsole ma downloadinstall sysadmin
   sudo fwconsole ma downloadinstall firewall
   sudo fwconsole ma installall
   sudo fwconsole chown
   sudo fwconsole reload
   ```

---

## ⚠️ Fase 6: Bug Conhecido do UCP Daemon / Known UCP Daemon Bug

Após a instalação, um **bug de instabilidade** com o **UCP Daemon** pode afetar o funcionamento do sistema no ARM64 (Debian 12), provocando falhas no `fwconsole start` automático.

> The UCP Daemon may enter a **crash loop** (seen via `fwconsole pm2 --list` showing high restarts).  
> This appears to be an unresolved issue with ARM64 on Debian 12.

**Solução Recomendável / Recommended Workaround:**

Após cada reboot:

```bash
# Inicie apenas os serviços essenciais
sudo fwconsole start
```

Para garantir que o UCP não cause instabilidade, desative-o manualmente:

```bash
# Caso o UCP seja iniciado acidentalmente
sudo fwconsole stop ucp
```

Isso garantirá que o sistema permaneça estável com os serviços como `core-calltransfer-monitor` e `core-fastagi` ativos.

---

## 🧠 Observação Final / Final Note

> Para um ambiente de produção mais robusto e sem esse bug, o FreePBX ainda é **mais estável em arquiteturas x86_64**.  
> *For production environments, x86_64 remains the most stable architecture for FreePBX.*
