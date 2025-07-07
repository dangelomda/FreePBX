Guia Definitivo: FreePBX 17 Bare Metal no Raspberry Pi 5 (Debian 12)
Definitive Guide: FreePBX 17 Bare Metal on Raspberry Pi 5 (Debian 12)
Este guia consolida o passo a passo completo e assertivo para instalar o FreePBX 17 em um Raspberry Pi 5 rodando o sistema operacional Debian 12 ("Bookworm") 64-bit. A estratégia central deste guia é unificar as permissões, fazendo com que o Asterisk e o servidor web Apache rodem sob o mesmo usuário (asterisk) para eliminar todos os conflitos de permissão e inicialização.

This guide consolidates the complete and assertive step-by-step process for installing FreePBX 17 on a Raspberry Pi 5 running the Debian 12 ("Bookworm") 64-bit operating system. The core strategy is to unify permissions by running both Asterisk and the Apache web server under the same user (asterisk) to eliminate all permission and startup conflicts.

Pré-requisito / Prerequisite
Um Raspberry Pi 5 com uma instalação limpa do Raspberry Pi OS Lite (64-bit).

A Raspberry Pi 5 with a clean installation of Raspberry Pi OS Lite (64-bit).

Acesso à internet configurado e funcionando no dispositivo.

Internet access configured and working on the device.

Fase 1: Preparação do Sistema / Phase 1: System Preparation
Prepara o sistema operacional com todas as ferramentas e pacotes necessários para o processo.
This phase prepares the operating system with all the necessary tools and packages.

Atualize completamente o sistema / Fully update the system:

Bash

sudo apt update && sudo apt upgrade -y
Instale todas as dependências de uma só vez / Install all dependencies at once:
Isso inclui ferramentas de compilação, bibliotecas essenciais, git, nodejs, npm, sox e incron.
This includes build tools, essential libraries, git, nodejs, npm, and sox.

Bash

sudo apt install -y build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev libsrtp2-dev uuid-dev sox nodejs npm incron
Fase 2: Configuração dos Serviços Base (Apache, MariaDB, PHP) / Phase 2: Base Services Configuration
Instala e configura o ambiente do servidor. A mudança crítica de fazer o Apache rodar como usuário asterisk é feita aqui para prevenir erros.
This phase installs and configures the server environment. The critical change of making Apache run as the asterisk user is done here to prevent errors.

Instale Apache e MariaDB / Install Apache and MariaDB:

Bash

sudo apt install -y apache2 mariadb-server
Instale o PHP 8.2 (padrão do Debian 12) e suas extensões / Install PHP 8.2 (Debian 12 default) and its extensions:

Bash

sudo apt install -y php php-mysql php-cli php-common php-gd php-mbstring php-xml php-curl php-json php-pear php-bcmath php-zip
Proteja a instalação do banco de dados / Secure the database installation:

Bash

sudo mysql_secure_installation
IMPORTANTE: Crie uma senha de root forte para o MariaDB e anote-a. Responda Sim (Y) para todas as outras perguntas.

IMPORTANT: Create a strong root password for MariaDB and write it down. It is safe to answer Yes (Y) to all other questions.

Configure o Apache para rodar como o usuário asterisk / Configure Apache to run as the asterisk user:

Bash

sudo nano /etc/apache2/envvars
Altere as duas linhas a seguir: / Change the following two lines:
export APACHE_RUN_USER=www-data -> export APACHE_RUN_USER=asterisk
export APACHE_RUN_GROUP=www-data -> export APACHE_RUN_GROUP=asterisk

Ajuste outras configurações do Apache e PHP / Adjust other Apache and PHP settings:

Bash

sudo a2enmod rewrite
sudo sed -i 's/\(AllowOverride \).*/\1All/' /etc/apache2/apache2.conf
sudo sed -i 's/\(^upload_max_filesize = \).*/\1120M/' /etc/php/8.2/apache2/php.ini
sudo sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/8.2/apache2/php.ini
sudo systemctl restart apache2
Fase 3: Compilação e Instalação do Asterisk / Phase 3: Asterisk Compilation & Installation
Compila o motor de telefonia. O usuário asterisk que o Apache agora usa será criado aqui.
This phase compiles the telephony engine. The asterisk user that Apache now uses will be created here.

Baixe e extraia o código-fonte do Asterisk 20 / Download and extract the Asterisk 20 source code:

Bash

cd /usr/src/
sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar -zxvf asterisk-20-current.tar.gz
cd asterisk-20.*/
Prepare, configure, compile e instale / Prepare, configure, compile, and install:

Bash

sudo contrib/scripts/get_mp3_source.sh
sudo contrib/scripts/install_prereq install
sudo ./configure
sudo make menuselect
Dica: No menu, habilite pelo menos o format_mp3 na seção Add-ons. Pressione F12 para Salvar e Sair.

Tip: In the menu, enable at least format_mp3 in the Add-ons section. Press F12 to Save & Exit.

Bash

sudo make # Aviso: Este comando pode demorar muitos minutos. / Warning: This command can take many minutes.
sudo make install
sudo make samples
sudo make config
Crie o usuário asterisk e defina as permissões base / Create the asterisk user and set base permissions:

Bash

sudo groupadd asterisk
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
sudo chown -R asterisk:asterisk /etc/asterisk /var/lib/asterisk /var/log/asterisk /var/spool/asterisk /usr/lib/asterisk
sudo sed -i 's/\(AST_USER=\).*/\1"asterisk"/' /etc/default/asterisk
sudo sed -i 's/\(AST_GROUP=\).*/\1"asterisk"/' /etc/default/asterisk
Fase 4: Instalação do FreePBX 17 / Phase 4: FreePBX 17 Installation
Instala a interface do FreePBX usando o método git, que é o mais confiável.
Installs the FreePBX interface using the git method, which is the most reliable.

Baixe o código-fonte do FreePBX 17 / Download the FreePBX 17 source code:

Bash

cd /usr/src/
sudo rm -rf freepbx
sudo git clone -b release/17.0 https://github.com/FreePBX/framework.git freepbx
Execute o script de instalação interativo / Run the interactive install script:

Bash

cd freepbx
sudo ./start_asterisk start
sudo ./install
Siga o instalador, aceitando os padrões (pressionando Enter), e digite a sua senha de root do MariaDB quando solicitado.

Follow the installer, accepting the defaults (by pressing Enter), and enter your MariaDB root password when prompted.

Fase 5: Finalização e Ajustes Pós-Instalação ("Lições Aprendidas") / Phase 5: Finalization & Post-Install Adjustments ("Lessons Learned")
Estes são os ajustes finos que descobrimos serem necessários para garantir que tudo funcione perfeitamente.
These are the fine-tuning adjustments we discovered are necessary to ensure everything works perfectly.

Instale o IonCube Loader / Install the IonCube Loader:

Bash

cd /tmp
sudo wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_aarch64.tar.gz
sudo tar -xzvf ioncube_loaders_lin_aarch64.tar.gz
sudo cp /tmp/ioncube/ioncube_loader_lin_8.2.so /usr/lib/php/20220829/
echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/apache2/conf.d/00-ioncube.ini
echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/cli/conf.d/00-ioncube.ini
sudo systemctl restart apache2
Instale os Módulos do FreePBX e Corrija Permissões / Install FreePBX Modules and Fix Permissions:

Bash

cd /usr/src/freepbx
sudo fwconsole ma downloadinstall sysadmin
sudo fwconsole ma downloadinstall firewall
sudo fwconsole ma installall
sudo fwconsole chown
Crie as Regras de Inicialização Automática Permanentes / Create Permanent Auto-Start Service Rules:

Bash

# Para corrigir o erro do "Apply Config" após reiniciar / To fix the "Apply Config" error after rebooting
sudo systemctl edit asterisk.service
Cole o seguinte conteúdo, salve e saia: / Paste the following content, then save and exit:

Ini, TOML

[Service]
RuntimeDirectory=asterisk
RuntimeDirectoryMode=0775
Bash

# Para iniciar o UCP Daemon automaticamente com atraso / To start the UCP Daemon automatically with a delay
sudo nano /etc/systemd/system/freepbx-ucp-delayed-start.service
Cole o seguinte conteúdo, salve e saia: / Paste the following content, then save and exit:

Ini, TOML

[Unit]
Description=Delayed and Robust Start for FreePBX UCP Daemon
After=network-online.target mariadb.service asterisk.service freepbx.service
[Service]
Type=oneshot
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStartPre=/bin/sleep 45
ExecStart=/usr/sbin/fwconsole start ucp

[Install]
WantedBy=multi-user.target

Aplique todas as mudanças de serviço e reinicie / Apply all service changes and restart:

Bash

sudo systemctl daemon-reload
sudo systemctl enable freepbx-ucp-delayed-start.service
sudo systemctl restart asterisk
sudo systemctl restart freepbx
sudo systemctl restart apache2
Fase 6: Acesso Final / Phase 6: Final Access
Abra seu navegador e acesse o endereço IP do seu Raspberry Pi: http://<IP_DO_SEU_PI>.

O sistema estará 100% funcional e robusto.
Your PBX system will be 100% functional and robust.












Vídeo

Deep Research

Canvas

