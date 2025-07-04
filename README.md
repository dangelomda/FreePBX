# FreePBX
FreePBX 17 - Baremetal in Rasperry PI 5

Guia Definitivo v2 (Completo): FreePBX 17 Bare Metal no Raspberry Pi 5 (Debian 12)
Este guia consolida o passo a passo completo e assertivo para instalar o FreePBX 17 em um Raspberry Pi 5 rodando o sistema operacional Debian 12 ("Bookworm") 64-bit. Todos os passos foram testados e incluem as correções necessárias para dependências e permissões.

Pré-requisito
Um Raspberry Pi 5 com uma instalação limpa do Raspberry Pi OS Lite (64-bit).

Acesso à internet configurado e funcionando no dispositivo.

Fase 1: Preparação do Sistema
Esta fase prepara o sistema operacional com todas as ferramentas e pacotes necessários para o processo.

Atualize completamente o sistema:

Bash

sudo apt update && sudo apt upgrade -y
Instale todas as dependências de uma só vez:
Isso inclui ferramentas de compilação, bibliotecas essenciais, git, nodejs e npm.

Bash

sudo apt install -y build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev libsrtp2-dev uuid-dev sox nodejs npm
Fase 2: Configuração dos Serviços Base (Apache, MariaDB, PHP)
Aqui, instalamos e configuramos o ambiente do servidor web e do banco de dados.

Instale o Apache e o MariaDB:

Bash

sudo apt install -y apache2 mariadb-server
Instale o PHP 8.2 (padrão do Debian 12) e todas as extensões exigidas:

Bash

sudo apt install -y php php-mysql php-cli php-common php-gd php-mbstring php-xml php-curl php-json php-pear php-bcmath php-zip
Proteja a instalação do banco de dados:

Bash

sudo mysql_secure_installation
IMPORTANTE: Siga as instruções na tela. Crie uma senha de root forte para o MariaDB e anote-a. É seguro responder Sim (Y) para todas as outras perguntas para remover padrões inseguros.

Configure o Apache e o PHP:

Bash

# Habilita o módulo 'rewrite' do Apache
sudo a2enmod rewrite

# Altera o AllowOverride para que os arquivos .htaccess do FreePBX funcionem
sudo sed -i 's/\(AllowOverride \).*/\1All/' /etc/apache2/apache2.conf

# Ajusta os limites de upload e memória do PHP
sudo sed -i 's/\(^upload_max_filesize = \).*/\1120M/' /etc/php/8.2/apache2/php.ini
sudo sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/8.2/apache2/php.ini
Crie a regra de permissão do sudo para o FreePBX:
Isso evita problemas de permissão da interface web no futuro.

Bash

echo "www-data ALL=(asterisk) NOPASSWD: /usr/sbin/fwconsole" | sudo tee /etc/sudoers.d/freepbx-security
sudo chmod 0440 /etc/sudoers.d/freepbx-security
Reinicie o Apache para aplicar todas as configurações:

Bash

sudo systemctl restart apache2
Fase 3: Compilação e Instalação do Asterisk
Esta fase compila o motor de telefonia Asterisk a partir do código-fonte.

Baixe e extraia o código-fonte do Asterisk 20:

Bash

cd /usr/src/
sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar -zxvf asterisk-20-current.tar.gz
cd asterisk-20.*/
Prepare as dependências e configure a compilação:

Bash

sudo contrib/scripts/get_mp3_source.sh
sudo contrib/scripts/install_prereq install
sudo ./configure
Selecione os módulos do Asterisk:

Bash

sudo make menuselect
Use as setas do teclado. É altamente recomendado habilitar ([*]) o item format_mp3 na seção Add-ons. Explore as outras seções como Core Sound Packages para adicionar pacotes de áudio. Pressione F12 para Salvar e Sair.

Compile e instale o Asterisk:
Aviso: O comando make pode levar muitos minutos.

Bash

sudo make
sudo make install
sudo make samples
sudo make config
Crie o usuário asterisk e defina as permissões:

Bash

sudo groupadd asterisk
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
sudo chown -R asterisk:asterisk /etc/asterisk /var/lib/asterisk /var/log/asterisk /var/spool/asterisk /usr/lib/asterisk
sudo sed -i 's/\(AST_USER=\).*/\1"asterisk"/' /etc/default/asterisk
sudo sed -i 's/\(AST_GROUP=\).*/\1"asterisk"/' /etc/default/asterisk
Fase 4: Instalação do FreePBX 17
Agora, com toda a base pronta, instalamos a interface do FreePBX.

Baixe o código-fonte do FreePBX 17 usando git:

Bash

cd /usr/src/
sudo rm -rf freepbx
sudo git clone -b release/17.0 https://github.com/FreePBX/framework.git freepbx
Execute o script de instalação interativo:

Bash

cd freepbx
sudo ./start_asterisk start
sudo ./install
Siga o instalador, aceitando os padrões (pressionando Enter), e quando for solicitado Database password:, digite a senha de root do MariaDB que você criou na Fase 2.

Fase 5: Finalização e Ajustes Pós-Instalação (A "Lição Aprendida")
Estes são os ajustes finos que descobrimos serem necessários para garantir que tudo funcione perfeitamente.

Instale o IonCube Loader (necessário para módulos comerciais):

Bash

cd /tmp
sudo wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_aarch64.tar.gz
sudo tar -xzvf ioncube_loaders_lin_aarch64.tar.gz
sudo cp /tmp/ioncube/ioncube_loader_lin_8.2.so /usr/lib/php/20220829/
echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/apache2/conf.d/00-ioncube.ini
echo "zend_extension = /usr/lib/php/20220829/ioncube_loader_lin_8.2.so" | sudo tee /etc/php/8.2/cli/conf.d/00-ioncube.ini
sudo systemctl restart apache2
Instale os módulos base do FreePBX e corrija as permissões:

Bash

cd /usr/src/freepbx
sudo fwconsole ma downloadinstall sysadmin
sudo fwconsole ma downloadinstall pm2
sudo fwconsole ma installall
Execute a correção de permissões final (resolve o loop de sessão):

Bash

sudo chown -R www-data:www-data /var/www/html/
Recarregue as configurações e reinicie os serviços:

Bash

sudo fwconsole reload
sudo systemctl restart asterisk
sudo systemctl restart apache2
Fase 6: Acesso Final
Abra seu navegador e acesse o endereço IP do seu Raspberry Pi: http://<IP_DO_SEU_PI>.

Você será recebido pela tela de criação de usuário administrador. Após isso, seu sistema PABX estará 100% funcional.
