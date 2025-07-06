# Guia Final e Vitorioso: FreePBX 17 Bare Metal no Raspberry Pi 5 (Debian 12)

Este guia consolida o passo a passo completo e assertivo para instalar o FreePBX 17 em um Raspberry Pi 5 rodando Debian 12 ("Bookworm") 64-bit. Inclui todas as correções de dependências e permissões descobertas para garantir uma instalação estável e persistente.

### Pré-requisito

* Um Raspberry Pi 5 com uma instalação limpa do **Raspberry Pi OS Lite (64-bit)**.
* Acesso à internet configurado e funcionando no dispositivo.

---

## Fase 1: Preparação do Sistema

1.  **Atualize o sistema:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2.  **Instale todas as dependências de uma só vez:**
    ```bash
    sudo apt install -y build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev libsrtp2-dev uuid-dev sox nodejs npm incron
    ```

---

## Fase 2: Configuração dos Serviços Base (Apache, MariaDB, PHP)

1.  **Instale os serviços web e de banco de dados:**
    ```bash
    sudo apt install -y apache2 mariadb-server
    ```

2.  **Instale o PHP 8.2 e suas extensões:**
    ```bash
    sudo apt install -y php php-mysql php-cli php-common php-gd php-mbstring php-xml php-curl php-json php-pear php-bcmath php-zip
    ```

3.  **Proteja a instalação do MariaDB:**
    ```bash
    sudo mysql_secure_installation
    ```
    > **IMPORTANTE:** Crie uma **senha de root forte** para o MariaDB e anote-a. Responda **Sim (Y)** para as outras perguntas.

4.  **Configure o Apache para rodar como o usuário `asterisk`:**
    ```bash
    sudo nano /etc/apache2/envvars
    ```
    > Altere `APACHE_RUN_USER=www-data` para `APACHE_RUN_USER=asterisk`
    > Altere `APACHE_RUN_GROUP=www-data` para `APACHE_RUN_GROUP=asterisk`

5.  **Ajuste outras configurações do Apache e PHP:**
    ```bash
    sudo a2enmod rewrite
    sudo sed -i 's/\(AllowOverride \).*/\1All/' /etc/apache2/apache2.conf
    sudo sed -i 's/\(^upload_max_filesize = \).*/\1120M/' /etc/php/8.2/apache2/php.ini
    sudo sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/8.2/apache2/php.ini
    sudo systemctl restart apache2
    ```

---

## Fase 3: Compilação e Instalação do Asterisk

1.  **Baixe e extraia o Asterisk 20:**
    ```bash
    cd /usr/src/
    sudo wget [http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz](http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz)
    sudo tar -zxvf asterisk-20-current.tar.gz
    cd asterisk-20.*/
    ```

2.  **Prepare, configure, compile e instale:**
    ```bash
    sudo contrib/scripts/get_mp3_source.sh
    sudo contrib/scripts/install_prereq install
    sudo ./configure
    sudo make menuselect # Pressione F12 para Salvar e Sair
    sudo make # Aviso: Este comando demora
    sudo make install
    sudo make samples
    sudo make config
    ```

3.  **Crie o usuário e grupo `asterisk`:**
    ```bash
    sudo groupadd asterisk
    sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
    sudo chown -R asterisk:asterisk /etc/asterisk /var/lib/asterisk /var/log/asterisk /var/spool/asterisk /usr/lib/asterisk
    sudo sed -i 's/\(AST_USER=\).*/\1"asterisk"/' /etc/default/asterisk
    sudo sed -i 's/\(AST_GROUP=\).*/\1"asterisk"/' /etc/default/asterisk
    ```

---

## Fase 4: Instalação do FreePBX 17

1.  **Baixe o código-fonte do FreePBX 17 via `git`:**
    ```bash
    cd /usr/src/
    sudo rm -rf freepbx
    sudo git clone -b release/17.0 [https://github.com/FreePBX/framework.git](https://github.com/FreePBX/framework.git) freepbx
    ```

2.  **Execute o script de instalação interativo:**
    ```bash
    cd freepbx
    sudo ./start_asterisk start
    sudo ./install
    ```
    > Siga o instalador, aceitando os padrões (pressionando **Enter**), e digite a sua senha de root do MariaDB quando solicitado.

---

## Fase 5: Finalização e Ajustes Pós-Instalação

1.  **Instale o IonCube Loader:**
    ```bash
    cd /tmp
    sudo wget [https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_aarch64.tar.gz](https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_aarch64.tar.gz)
    sudo tar -xzvf ioncube_loaders_lin_aarch64.tar.gz
    sudo cp /tmp/ioncube/ioncube_loader_lin_8.2.so /usr/lib/php/20220829/
    echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/apache2/conf.d/00-ioncube.ini
    echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/cli/conf.d/00-ioncube.ini
    sudo systemctl restart apache2
    ```

2.  **Instale os Módulos e Corrija as Permissões Finais:**
    ```bash
    cd /usr/src/freepbx
    sudo fwconsole ma downloadinstall sysadmin
    sudo fwconsole ma downloadinstall firewall
    sudo fwconsole ma installall
    sudo fwconsole chown
    sudo fwconsole reload
    ```
    
3.  **Crie os Scripts de Inicialização Automática Permanentes:**
    ```bash
    # Para o erro "Apply Config"
    sudo systemctl edit asterisk.service
    ```
    > Cole o seguinte conteúdo, salve e saia:
    > ```ini
    > [Service]
    > User=asterisk
    > Group=asterisk
    > RuntimeDirectory=asterisk
    > RuntimeDirectoryMode=0775
    > ```

    ```bash
    # Para o UCP Daemon teimoso
    sudo systemctl edit freepbx.service
    ```
    > Cole o seguinte conteúdo, salve e saia. *Nota: `freepbx.service` é o nome correto do serviço principal.*
    > ```ini
    > [Service]
    > ExecStartPost=/bin/bash -c "sleep 20 && /usr/sbin/fwconsole start ucp"
    > ```

4.  **Aplique todas as mudanças de serviço e reinicie:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart asterisk
    sudo systemctl restart freepbx
    sudo systemctl restart apache2
    ```

---

## Fase 6: Acesso Final

Abra seu navegador e acesse o endereço IP do seu Raspberry Pi: `http://<IP_DO_SEU_PI>`. O sistema estará 100% funcional e robusto.
