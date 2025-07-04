# Guia Definitivo v2.1 (Completo): FreePBX 17 Bare Metal no Raspberry Pi 5 (Debian 12)

Este guia consolida o passo a passo completo e assertivo para instalar o FreePBX 17 em um Raspberry Pi 5 rodando o sistema operacional Debian 12 ("Bookworm") 64-bit. A estratégia central deste guia é unificar as permissões, fazendo com que o Asterisk e o servidor web Apache rodem sob o mesmo usuário (`asterisk`) para eliminar todos os conflitos.

### Pré-requisito

* Um Raspberry Pi 5 com uma instalação limpa do **Raspberry Pi OS Lite (64-bit)**.
* Acesso à internet configurado e funcionando no dispositivo.

---

## Fase 1: Preparação do Sistema

1.  **Atualize completamente o sistema:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2.  **Instale todas as dependências de uma só vez:**
    ```bash
    sudo apt install -y build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev libsrtp2-dev uuid-dev sox nodejs npm
    ```

---

## Fase 2: Configuração dos Serviços Base (Apache, MariaDB, PHP)

1.  **Instale Apache e MariaDB:**
    ```bash
    sudo apt install -y apache2 mariadb-server
    ```

2.  **Instale o PHP 8.2 e todas as extensões:**
    ```bash
    sudo apt install -y php php-mysql php-cli php-common php-gd php-mbstring php-xml php-curl php-json php-pear php-bcmath php-zip
    ```

3.  **Proteja a instalação do banco de dados:**
    ```bash
    sudo mysql_secure_installation
    ```
    > **IMPORTANTE:** Crie uma **senha de root forte** para o MariaDB e anote-a. Responda **Sim (Y)** para todas as outras perguntas.

4.  **Configure o Apache para rodar como o usuário `asterisk`:**
    ```bash
    sudo nano /etc/apache2/envvars
    ```
    > Altere as duas linhas a seguir:
    > `export APACHE_RUN_USER=www-data` -> `export APACHE_RUN_USER=asterisk`
    > `export APACHE_RUN_GROUP=www-data` -> `export APACHE_RUN_GROUP=asterisk`

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

1.  **Baixe e extraia o código-fonte do Asterisk 20:**
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
    sudo make # Este comando pode demorar muitos minutos
    sudo make install
    sudo make samples
    sudo make config
    ```

3.  **Crie o usuário `asterisk` e defina as permissões base:**
    ```bash
    sudo groupadd asterisk
    sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
    sudo chown -R asterisk:asterisk /etc/asterisk /var/lib/asterisk /var/log/asterisk /var/spool/asterisk /usr/lib/asterisk
    sudo sed -i 's/\(AST_USER=\).*/\1"asterisk"/' /etc/default/asterisk
    sudo sed -i 's/\(AST_GROUP=\).*/\1"asterisk"/' /etc/default/asterisk
    ```

---

## Fase 4: Instalação do FreePBX 17

1.  **Baixe o código-fonte do FreePBX 17 usando `git`:**
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

## Fase 5: Finalização e Ajustes Pós-Instalação ("Lições Aprendidas")

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
    sudo fwconsole ma installall
    sudo fwconsole chown

    # Correção explícita para o erro do "Apply Config"
    sudo chown -R asterisk:asterisk /var/run/asterisk
    sudo chmod -R 775 /var/run/asterisk

    # Recarrega e reinicia tudo para finalizar
    sudo fwconsole reload
    sudo systemctl restart asterisk
    sudo systemctl restart apache2
    ```

---

## Fase 6: Acesso Final

Abra seu navegador e acesse o endereço IP do seu Raspberry Pi: `http://<IP_DO_SEU_PI>`.

O sistema estará 100% funcional.
