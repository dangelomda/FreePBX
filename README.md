# ✅ Guia Definitivo v2 (Completo)

## FreePBX 17 Bare Metal no Raspberry Pi 5 (Debian 12)

Este guia consolida o passo a passo completo e assertivo para instalar o **FreePBX 17** em um Raspberry Pi 5 rodando **Debian 12 (“Bookworm”) 64-bit**. Todos os comandos foram testados e incluem as correções necessárias para dependências e permissões.

---

## 🔧 Pré-requisitos

- Raspberry Pi 5 com instalação limpa do **Raspberry Pi OS Lite 64-bit (Debian 12)**
- Acesso à internet funcional no dispositivo

---

## 🛠️ Fase 1 – Preparação do Sistema

### Atualize completamente o sistema

```bash
sudo apt update && sudo apt upgrade -y
```

### Instale as dependências essenciais

Inclui ferramentas de compilação, bibliotecas, git, Node.js e npm.

```bash
sudo apt install -y \
  build-essential \
  git \
  curl \
  wget \
  libnewt-dev \
  libssl-dev \
  libncurses5-dev \
  subversion \
  libsqlite3-dev \
  libjansson-dev \
  libxml2-dev \
  libsrtp2-dev \
  uuid-dev \
  sox \
  nodejs \
  npm
```

---

## 🌐 Fase 2 – Configuração dos Serviços Base

### Instale Apache e MariaDB

```bash
sudo apt install -y apache2 mariadb-server
```

### Instale PHP 8.2 e extensões exigidas

```bash
sudo apt install -y \
  php \
  php-mysql \
  php-cli \
  php-common \
  php-gd \
  php-mbstring \
  php-xml \
  php-curl \
  php-json \
  php-pear \
  php-bcmath \
  php-zip
```

### Proteja o MariaDB

```bash
sudo mysql_secure_installation
```

> **Dica:**
>
> - Crie uma senha forte para o usuário root do MariaDB.
> - Responda **Yes** para todas as outras opções para remover padrões inseguros.

### Configure Apache e PHP

Habilite o módulo `rewrite` do Apache:

```bash
sudo a2enmod rewrite
```

Permita uso de `.htaccess` alterando `AllowOverride`:

```bash
sudo sed -i 's/\(AllowOverride \).*/\1All/' /etc/apache2/apache2.conf
```

Ajuste limites de upload e memória do PHP:

```bash
sudo sed -i 's/^\(upload_max_filesize = \).*/\1120M/' /etc/php/8.2/apache2/php.ini
sudo sed -i 's/^\(memory_limit = \).*/\1256M/' /etc/php/8.2/apache2/php.ini
```

### Configure permissões sudo para o FreePBX

Crie regra para permitir que o Apache execute o fwconsole sem senha:

```bash
echo "www-data ALL=(asterisk) NOPASSWD: /usr/sbin/fwconsole" | sudo tee /etc/sudoers.d/freepbx-security
sudo chmod 0440 /etc/sudoers.d/freepbx-security
```

### Reinicie Apache

```bash
sudo systemctl restart apache2
```

---

## ☎️ Fase 3 – Compilação e Instalação do Asterisk

### Baixe e extraia o código-fonte do Asterisk 20

```bash
cd /usr/src/
sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar -zxvf asterisk-20-current.tar.gz
cd asterisk-20.*/
```

### Prepare dependências e configure a compilação

```bash
sudo contrib/scripts/get_mp3_source.sh
sudo contrib/scripts/install_prereq install
sudo ./configure
```

### Selecione os módulos do Asterisk

Execute o menuselect:

```bash
sudo make menuselect
```

- Use as setas para navegar.
- É recomendado habilitar `[ * ] format_mp3` em **Add-ons**.
- Pressione **F12** para salvar e sair.

### Compile e instale o Asterisk

> **Atenção:** o processo pode demorar vários minutos.

```bash
sudo make
sudo make install
sudo make samples
sudo make config
```

### Crie usuário e permissões para o Asterisk

```bash
sudo groupadd asterisk
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
sudo chown -R asterisk:asterisk \
    /etc/asterisk \
    /var/lib/asterisk \
    /var/log/asterisk \
    /var/spool/asterisk \
    /usr/lib/asterisk
```

Configure o ambiente do Asterisk para rodar com o usuário correto:

```bash
sudo sed -i 's/\(AST_USER=\).*/\1"asterisk"/' /etc/default/asterisk
sudo sed -i 's/\(AST_GROUP=\).*/\1"asterisk"/' /etc/default/asterisk
```

---

## 🎛️ Fase 4 – Instalação do FreePBX 17

### Baixe o código-fonte do FreePBX 17

```bash
cd /usr/src/
sudo rm -rf freepbx
sudo git clone -b release/17.0 https://github.com/FreePBX/framework.git freepbx
```

### Execute o instalador do FreePBX

```bash
cd freepbx
sudo ./start_asterisk start
sudo ./install
```

> Durante a instalação:
> - Pressione Enter para aceitar os padrões.
> - Quando solicitado `Database password:`, insira a senha de root do MariaDB criada na Fase 2.

---

## 🔧 Fase 5 – Finalização e Ajustes Pós-Instalação

### Instale IonCube Loader (necessário para módulos comerciais)

```bash
cd /tmp
sudo wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_aarch64.tar.gz
sudo tar -xzvf ioncube_loaders_lin_aarch64.tar.gz
sudo cp ioncube/ioncube_loader_lin_8.2.so /usr/lib/php/20220829/
```

Crie arquivos de configuração do IonCube:

```bash
echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/apache2/conf.d/00-ioncube.ini
echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/cli/conf.d/00-ioncube.ini
```

Reinicie Apache:

```bash
sudo systemctl restart apache2
```

---

### Instale módulos FreePBX base e ajuste permissões

```bash
cd /usr/src/freepbx
sudo fwconsole ma downloadinstall sysadmin
sudo fwconsole ma downloadinstall pm2
sudo fwconsole ma installall
```

Corrija permissões finais (evita erros de sessão):

```bash
sudo chown -R www-data:www-data /var/www/html/
```

Recarregue as configurações e reinicie serviços:

```bash
sudo fwconsole reload
sudo systemctl restart asterisk
sudo systemctl restart apache2
```

---

## 🌐 Fase 6 – Acesso Final

Abra o navegador e acesse:

```
http://<IP_DO_SEU_PI>
```

- Você verá a tela de criação do usuário administrador do FreePBX.
- Após isso, seu sistema PABX estará **100% funcional**!

---

## ✅ Notas Importantes

✅ **Performance**

O Raspberry Pi 5 possui performance muito superior às gerações anteriores. Mesmo com CPU ARM, consegue rodar FreePBX + Asterisk sem dificuldades para pequenos/médios cenários.

✅ **IP Estático**

Recomenda-se definir IP fixo para evitar problemas com DHCP mudando o endereço do seu servidor.

✅ **Swap**

Se planeja usar muitos módulos ou chamadas simultâneas, considere aumentar swap para evitar falta de memória.

✅ **Backups**

Configure backups automáticos via GUI do FreePBX assim que finalizar a instalação.

---
