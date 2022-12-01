# Instalación RHEL8.6


[![Interconectando®](/documentos/instalacion/assets/images/logo_interconectando_cont.png)](https://interconectando.com) <h2> Plataforma de cálculo y valuación actuarial **Interconectando® AVC Server** </h2>

## Instrucciones de Instalación, actualización y recuperación, en una instancia de servidor VM Google Cloud - Linux RHEL 8.7


### Requerimientos

#### Instancia

| **Objeto** | **Valor** |
|---:|---|
| CPU: | 4 núcleos |
| RAM: | 8GB |
| Almacenamiento: | 30GB SSD |
| Nombre de dominio<br>ó sub-dominio: | **interval.intercam.com.mx** |
| Correo electrónico <br>transaccional (opcional) | sistema@interval.intercam.com.mx |
| Reglas de DNS de dominio: | Tipo A |
| Reglas de Firewall: | - Permiso: Allow, Origen: :::/, Destino: 127.0.0.1 ó localhost, Puertos: 22, 80, 433.<br>|
| Llaves ssh: | - Acceso remoto a VM  <br>- Acceso de VM a repositorio Github |
| **Conexión a Internet:** | **Durante instalación y actualizaciones.** |


#### Variables 

| **Nombre** | **Variable** | **Valor** | **Nota** |
|---:|---|---|---|
| Usuario Linux Root | [usuario_root] | root |  |
| Contraseña usuario Linux root | [pass_root] | "           "  | (opcional con ssh) |
| Usuario Linux sistema | [usuario_wheel] | avc-server | Usuario a configurar en el Framework. |
| Contraseña usuario Linux sistema | [pass_usuario_wheel] | "           " |  |
| Usuario MariaDB | [mariadb_user] | " root "  |   |
| Contraseña usuario MariaDB | [mariadb_user_pass] | " [pass_root] " |    |
| Correo de sistema | [correo_sistema] | sistema@interval.intercam.com.mx |  |
| Nombre de dominio | [sitio] | interval.intercam.com.mx | |
| IP4 privada | [ip4_prv] |"10.100.1.120"|  |
| IP4 pública | [ip4_pub] |"     .    .    .     "| (opcional) |
| Gateway | [ip4_prv] |"10.100.1. "|  |

**NOTA: En estas instrucciones de instalación, se asume que las configuraciones de "Suscripción del sistema operativo RHEL", configuraciones de red interna y externa y configuraciones de dominio, serán configuradas previamente en la nube para la operación de la instancia de servidor.** 
  
  
---
  

## Pasos de instalación
    
  
### 0. Creación manual de instancia en Cloud y acceso remoto.
  - Crea una instancia VM en [Google Cloud - Compute engine, Instancias de VM](https://console.cloud.google.com/compute/instances?hl=es-419)
  - Crea y asigna las reglas de firewall de la instancia [Google Cloud - Red de VPC, Firewall](https://console.cloud.google.com/networking/firewalls/list?hl=es-419)
    - Nombre: avc-in, Tipo: entrada, Destino: [ip4_prv], Protocolos/Puertos: tcp: 443. Acción: Permitir  
    - Nombre: avc-servicio, Tipo: entrada, Destino: [ip4_prv], Protocolos/Puertos: tcp: 80, 8000, 22. Acción: Permitir
  - Si administrarás el dominio desde Google Cloud, crea una zona [Google Cloud - Servicios de red, Cloud DNS](https://console.cloud.google.com/net-services/dns/zones?hl=es-419), [Google Cloud - Servicios de red, Cloud DNS | crear zona](https://console.cloud.google.com/net-services/dns/zones/new/create?hl=es-419)
  - Agrega un conjunto de registros **Tipo A** en la zona creada del dominio, ó, sub-dominio, apuntando a la IP del servidor.
  - Accesa a tu instancia via consola. Mediante cliente SSH.
    ```SH
    ssh -i ~/.ssh/llave_ssh.pem root@[ip4_prv]
    ```
  
  
### 1. Revisión de versiones, usuario, configuración básica y actualización de sistema.
  
  - 1.1 Revisión de versiones de Kernel, Linux, y otros datos.
    ```sh
    whoami                        # (respuesta: root)
    sudo su -l root
    # cd ..
    
    cat /etc/os-release
    cat /etc/redhat-release       # (respuesta: Red Hat Enterprise Linux release 8.6 (Ootpa)) 
    
    uname -a                      # (respuesta: Linux interval 4.18.0-372.26.1.el8_6.x86_64 1 SMP Sat Aug 27 02:44:20 EDT 2022 x86_64 x86_64 x86_64 GNU/Linux)
        
    # cat /proc/cpuinfo 
    # cat /proc/cpuinfo | grep processor.
    ```
    
  - 1.2 Consulta y asigna la **Zona horaria** de la Ciudad de México.
    ```sh
    timedatectl
    timedatectl set-timezone America/Mexico_City
    ```
    
  - 1.3 Elimina servidor WEB Apache, en caso de existir.
    ```sh
    sudo systemctl stop httpd       # Failed to stop httpd.service: Unit httpd.service not loaded.
    sudo yum remove -y httpd httpd-tools apr apr-util    
    ```
    
  - 1.4 Revisa la versión de MariaDB actual, y eliminala, en caso de existir.
    ```sh
    mariadb --version                       # (Debe ser >=: mariadb Ver 15.1 Distrib 10.6.7-MariaDB, )
    sudo systemctl stop mariadb
    sudo yum remove "MariaDB-*"
    sudo yum remove galera-4
    sudo yum remove galera
    rpm --query --all | grep -i -E "mariadb|galera"
    ```
    
  - 1.5 **Firewall**
  
    Agrega estas reglas al firewall INVESTIGAR LA DE PUERTO 22
    ```sh
    sudo firewall-cmd --zone=public --add-port=22/tcp
    sudo firewall-cmd --zone=public --add-port=80/tcp
    sudo firewall-cmd --zone=public --add-port=443/tcp
    sudo firewall-cmd --zone=public --add-port=8000/tcp      <--- Durante instalación
    sudo firewall-cmd --runtime-to-permanent
    ```
      
  - 1.6 **Actualiza** y reinicia la instancia.
    ```sh
    sudo yum update -y
    reboot
    ```
    
  - 1.7 Crea usuario linux con contraseña, con directorio HOME en **/bin/bash**. 
    ```sh
    useradd -m -s /bin/bash avc-server

    passwd avc-server
    
      New password:         intercamsf
      Retype new password:  intercamsf
      >>>passwd: all authentication tokens updated successfully.
    
    # Asigna al grupo de usuarios "wheel".
      usermod -aG wheel avc-server
    
    # Comando Stream editor para evitar introducir contraseña para los usuarios wheel.
      sed -i 's/^#\s*\(%wheel\s\+ALL=(ALL)\s\+NOPASSWD:\s\+ALL\)/\1/' /etc/sudoers
    ```
      
  - 1.8 Configura el Path del usuario
    ```sh
    su - avc-server
      
    # Edita le archivo **.bashrc** 
    vi ~/.bashrc
    
        # Agrega estas líneas. Puede ser al final. 
          PATH=$PATH:~/.local/bin/
          PATH=$PATH:/usr/local/bin/

    # Asigna la fuente.
    source ~/.bashrc
    ```
    
  - 1.9 **Localización**
    ```sh
    # Selecciona distribución de teclado y lenguaje y codificación.
      dnf install glibc-langpack-en -y
      # dnf install langpacks-en glibc-all-langpacks -y
      localectl set-keymap us && sudo localectl set-locale LANG=en_US.utf8
        
    # Edita el archivo environment 
      sudo vi /etc/environment
      
      # Y agrega los valores:
        LC_ALL=C.UTF-8
        LC_CTYPE=en_US.UTF-8
        LANG=en_US.UTF-8
        
    # Edita el archivo sudoers 
      sudo vi /etc/sudoers
      
      # Y agrega ruta apl PATH:
        # Antes
        Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
        
        # Después
        Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
    
    
    # Reinicia
    sudo reboot
    ```
----
    
    
### 2. Instalación de dependencias, módulos, paquetes, bibliotecas y herramientas.   Tiempo aprox a ese punto 13 mins.
    
  - 2.1 Bibliotecas
    ```sh
    sudo su -l root
    # cd ..
       
    ```
        
  - 2.2 Instala el repositorio de paquetes firmados **EPEL** y actívalo.
    ```sh
    sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    
    
    # Activa el CodeReady Builder (CRB) repository
    /usr/bin/crb enable
        # Respuesta
        Enabling CRB repo
        /usr/bin/crb: line 47: subscription-manager: command not found
        CRB repo is enabled and named: rhui-codeready-builder-for-rhel-8-x86_64-rhui-rpms
    ```
    
  - 2.3 Bibliotecas y herramientas.
    
    ```sh
    sudo yum groupinstall -y "Development tools"
    sudo yum -y install redhat-lsb-core git openssl openssl-devel libffi-devel zlib-devel bzip2-devel libffi libffi-devel xz-devel nano tar wget gcc make yum-utils gcc-c++ git libxslt-devel libxml2-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel libpcap-devel expat-devel uuid libnsl2-devel
    
    sudo yum -y install git openssl openssl-devel libffi-devel zlib-devel bzip2-devel libffi libffi-devel xz-devel nano tar wget gcc make yum-utils gcc-c++ git libxslt-devel libxml2-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel libpcap-devel expat-devel uuid libnsl2-devel
    
    sudo yum install libuv-devel --nobest -y
    sudo reboot
    ```
    
  - 2.4 Instala **Python 3.10**                      # Tiempo aprox a ese punto 19 mins.
    ```sh
    sudo su -l root
    # cd ..
    dnf remove python3 -y
    sudo yum-builddep -y
    sudo yum-builddep python3 -y
    sudo reboot

    
    cd /usr/local/src
    rm -rf Python-*
    wget https://www.python.org/ftp/python/3.10.8/Python-3.10.8.tgz
    tar xvf Python-3.10.8.tgz
    cd Python-3.10.8
    
    # sudo ./configure --prefix=/usr/bin --enable-optimizations --with-system-ffi --with-computed-gotos --enable-loadable-sqlite-extensions
    sudo ./configure --enable-optimizations --with-system-ffi --with-computed-gotos --enable-loadable-sqlite-extensions

    sudo make -j "$(nproc)"
    sudo make altinstall

    sudo rm /usr/local/src/Python-3.10.8.tgz
            
    # Agregar symlinks
      sudo ln -s /usr/local/bin/python3.10 /usr/bin/python
      sudo ln -s /usr/local/bin/python3.10 /usr/local/bin/python3
      sudo ln -s /usr/local/bin/python3.10 /usr/bin/python310
      sudo rm /usr/bin/python3
      sudo ln -s /usr/local/bin/python3.10 /usr/bin/python3

      sudo ln -s /usr/local/bin/python3.10 /usr/local/bin/python
      sudo ln -s /usr/local/bin/python3.10-config /usr/bin/python-config
      sudo ln -s /usr/local/bin/pydoc3.10 /usr/bin/pydoc
      sudo ln -s /usr/local/bin/idle3.10 /usr/bin/idle
      sudo ln -s /usr/local/bin/2to3-3.10 /usr/bin/2to3

      sudo ln -s /usr/local/bin/python3.10-config /usr/local/bin/python-config
      sudo ln -s /usr/local/bin/pydoc3.10 /usr/local/bin/pydoc
      sudo ln -s /usr/local/bin/idle3.10 /usr/local/bin/idle

      sudo rm /usr/bin/pip3
      sudo ln -s /usr/local/bin/pip3.10 /usr/bin/pip3
      sudo ln -s /usr/local/bin/pip3.10 /usr/bin/pip

    sudo reboot
    ```   
    
  - 2.5 Instala **Nginx y Redis**                   # Tiempo aprox a ese punto 30 mins.
    ```sh
    sudo su -l root
    cd..    
    ```
     - Inicia y activa **Redis 5.0.3**
      ```sh
      sudo yum install -y redis nginx
      sudo systemctl enable redis
      sudo systemctl start redis
      sudo systemctl status redis
      ```
    
    - Inicia y activa **NGINX**
      ```sh
      ### chkconfig nginx on
      sudo systemctl enable nginx
      sudo systemctl start nginx
      sudo systemctl status nginx      
      ```
      
  - 2.6 Instala **NodeJS**. 
    ```sh
    curl -fsSL https://rpm.nodesource.com/setup_16.x | bash -
    ## sudo yum remove -y nodejs npm
    sudo yum install -y nodejs
    ```
    
  - 2.7 Instala **Yarn**.
    ```sh
    curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
    sudo yum install -y yarn
    sudo yarn add latest
    sudo yarn add node-sass
    ```
    
  - 2.8 Instala **NPM, Node Sass, Sass**. 
    ```sh
    # npm install -g npm@8.19.2
    npm install -g npm
    npm install node-sass
    npm install -g sass
    ```
    
  - 2.9 Instala y configura **Supervisor**
    ```sh
    sudo yum -y install supervisor
    ```
    ```sh
    sudo vi /etc/supervisord.conf 
    ```
      - Quita el comentario, ó, agrega la línea dentro de las secciones Under **[unix_http_server]** y **[supervisord]**
        ```sh
        chown=avc-server:avc-server
        ```

      - Dentro de **[unix_http_server]** 
        ```sh
        # Comenta:
        ;file=/run/supervisor/supervisor.sock   ; (the path to the socket file)

        # Agrega:
        file=/tmp/supervisor.sock   ; (the path to the socket file)
        ```
        
     - Dentro de **[supervisorctl]** 
        ```sh
        # Comenta:
        ;serverurl=unix:///run/supervisor/supervisor.sock ; use a unix:// URL  for a unix socket
        ```
    
    
  - 2.10 Instala **MariaDB v10.5** desde root
    ```sh
    curl -LsS -O https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
    bash mariadb_repo_setup --mariadb-server-version=10.5
    
    yum install boost-program-options -y
    yum module reset mariadb -y
    
    yum -y install MariaDB-server MariaDB-client MariaDB-backup
            
    systemctl enable --now mariadb
    systemctl start mariadb.service
    systemctl status mariadb.service
    
    rpm -qi MariaDB-server
        
    ### En caso de encontrar el siquiente error durante, o posterior a la instalación de MariaDB.
        ## ERROR 1146 (42S02) at line 1: Table 'mysql.global_priv' doesn't exist
        sudo mysql_upgrade
    
    # Configura los formatos de la base de datos Mariadb.
      vi /etc/my.cnf      
              
        # Agrega estas líneas.  
          [mysqld]
          character-set-client-handshake = FALSE
          character-set-server = utf8mb4
          collation-server = utf8mb4_unicode_ci
          # innodb-read-only-compressed = OFF

          [mysql]
          default-character-set = utf8mb4
        
    ### Realiza una instalación segura
    
      mariadb-secure-installation
            
        # Ingresa las opciones que recomendamos, ó, personalizalo a discreción. 
      
          Enter current password for root (enter for none): 
          Press your [Enter] key, there is no password set by default  
          Switch to unix_socket authentication            N
          You already have your root account protected, so you can safely answer 'n'.  
          Change the root password? [Y/n]                 n
          Set root password? [Y/n]                        Y  
          New password:                                   intercamsf
          Re-enter new password:                          intercamsf  
          Remove anonymous users? [Y/n]                   Y  
          Disallow root login remotely? [Y/n]             Y  
          Remove test database and access to it? [Y/n]    Y  
          Reload privilege tables now? [Y/n]              Y  
                      
      systemctl restart mysql
      systemctl status mariadb
    ```
  
  - 2.11 Instala WkHTMLtoPDF
    
    - Obten e instala WkHTMLtoPDF
      ```sh
      yum -y install libXrender libXext xorg-x11-fonts-75dpi xorg-x11-fonts-Type1
      wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.4/wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
      tar -xf wkhtmltox-0.12.4_linux-generic-amd64.tar.xz -C /opt
      
      ln -s /opt/wkhtmltox/bin/wkhtmltopdf /usr/bin/wkhtmltopdf
      ln -s /opt/wkhtmltox/bin/wkhtmltoimage /usr/bin/wkhtmltoimage
      
      wkhtmltopdf -V
      >>> wkhtmltopdf 0.12.4 (with patched qt)
      ```
      
  - 2.12 Una comprobación final de los requisitos y versiones.
      ```sh
      python3.10 -V                   // (Debe ser >=: Python 3.10.8 )
      node --version                  // (Debe ser >=: v16.18.0 )
      npm --version                   // (Debe ser >=: 8.19.2 )
      mariadb --version               // (Debe ser >=: mariadb  Ver 15.1 Distrib 10.6.10-MariaDB, for Linux (x86_64) using readline 5.1 )
      yarn --version                  // (Debe ser >=: 1.22.19 )
      npm node-sass -v                // (Debe ser >=: 8.19.2 )
      pip3 --version                  // (Debe ser >=: pip 22.2.2 from /usr/local/lib/python3.10/site-packages/pip (python 3.10))
      pip --version                   // (Debe ser >=: pip pip 22.2.2 from /usr/local/lib/python3.10/site-packages/pip (python 3.10))
      redis-server --version          // (Debe ser >=: Redis server v=5.0.3 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=d0e2dcdeba8c0cee) 
      wkhtmltopdf —version            // (Debe ser >=: wkhtmltopdf 0.12.4 )
      git —-version                   // (Debe ser >=: git version 2.34.1)
      ```
      python3.10 -V && node --version &&  npm --version && mariadb --version && yarn --version && npm node-sass -v && pip3 -V && pip3.10 -V && redis-server --version && git —-version && wkhtmltopdf —V
---


### 3. Otras configuraciones
  

    - 3.3 Reinicia
      ```sh
      sudo reboot
      ```

    - 3.4 Revisa la versión de node en este punto, para confirmar que la versión instalada correspinda a Node-16
      ```sh
      node --version    # ( Debe ser >=: v16.18 )
      npm --version     # ( Debe ser >=: 8.19.2 )
      ```
    
    
### 4. Instalación de framework e Interconectando® AVC Server
  
  - 4.1 Crea el directorio de instalación del framework
    ```sh
    sudo mkdir /home/avc-server/bench
    sudo chown -R avc-server:avc-server /home/avc-server/bench
    sudo chmod -R o+rx /home/avc-server
    
    sudo mkdir /home/avc-server/bench/fuentes
    sudo chown -R avc-server:avc-server /home/avc-server/bench/fuentes
    sudo chmod -R o+rx /home/avc-server/fuentes
    ```
    
  - 4.2 Agrega node-saas al bench
    ```sh
    cd /home/avc-server/bench/
    npm install node-sass
    # npm install node-sass@6.0.1
    
    sudo yarn add node-sass
        
    # Revisa las versiones de yarn y node-sass en este punto
      
      yarn --version                        # (Debe ser >=: 1.22.19 )
      
      ./node_modules/.bin/node-sass -v      # (Debe ser >=: node-sass	7.0.3	(Wrapper)	[JavaScript]
                                              libsass 3.5.5 (Sass Compiler)	[C/C++] )
      
      npm node-sass -v                      # (Debe ser >=: 8.19.2 )
      ```
  
  - 4.3  Actualiza PIP, instala el ambiente e instala el framework
    
    PIP 
    
    | **Pack** | **Ver.** | **Fuente** | **Checksums \| SHA256 - MD5 - Blake-256** |
    |:---:|:---:|:---:|:---:|
    | pip | 22.3.1 | pip-22.3.1.tar.gz<br>https://pypi.org/project/pip/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/pip | 65fd48317359f3af8e593943e6ae1506b66325085ea64b706a998c6e83eeaf38<br>996f58a94fe0b8b82b6795c42bd171ba<br>a350c4d2727b99052780aad92c7297465af5fe6eec2dbae490aa9763273ffdc1 |
    | setuptools | 65.6.0 | setuptools-65.6.0.tar.gz<br>https://pypi.org/project/setuptools/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/setuptools | d1eebf881c6114e51df1664bc2c9133d022f78d12d5f4f665b9191f084e2862d<br>c2f54b9d5282984416a9ca08a4f338ba<br>09b633512596fb92ba68f7c45e9bbc5e1bb9b24fbd941f9aece250fb420c2f5c |
    | wheel | 0.38.4 | wheel-0.38.4.tar.gz<br>https://pypi.org/project/wheel/#files<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/wheel | 965f5259b566725405b05e7cf774052044b1ed30119b5d586b2703aafe8719ac<br>83bb4e7bd4d687d398733f341a64ab91<br>a2b86a06ff0f13a00fc3c3e7d222a995526cbca26c1ad107691b6b1badbbabf1 |
    | pyopenssl | 22.1.0 | pyOpenSSL-22.1.0.tar.gz<br>https://pypi.org/project/pyOpenSSL/python3.10<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/pyOpenSSL | 7a83b7b272dd595222d672f5ce29aa030f1fb837630ef229f62e72e395ce8968<br>6834da75e33d3c8dcd891b723bfcec9e<br>e72fc6d89edac75482f11e231b644e365d31d5479b7b727734e6a8f3d00decd5 |
    | virtualenv | 20.16.7 | virtualenv-20.16.7.tar.gz<br>https://pypi.org/project/virtualenv/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/virtualenv | 8691e3ff9387f743e00f6bb20f70121f5e4f596cae754531f2b3b3a1b1ac696e<br>6afef08830926a345f673dadc93f979c<br>2bdcbe4da7a7fea4e8c3612a4f1901efc694b4f5f1c30179518ffef88c5f8dde |
    | numpy | 1.23.5 | numpy-1.23.5.tar.gz<br>https://pypi.org/project/numpy/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/numpy | 1b1766d6f397c18153d40015ddfc79ddb715cabadc04d2d228d4e5a8bc4ded1a<br>8b2692a511a3795f3af8af2cd7566a15<br>4238775b43da55fa7473015eddc9a819571517d9a271a9f8134f68fb9be2f212 |
    | pandas | 1.5.2 | pandas-1.5.2.tar.gz<br>https://pypi.org/project/pandas/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/pandas | 220b98d15cee0b2cd839a6358bd1f273d0356bf964c1a1aeb32d47db0215488b<br>6da04c30fa35af20fe4d866e0f05caae<br>4d07c4d69e1acb7723ca49d24fc60a89aa07a914dfb8e7a07fdbb9d8646630cd |
    | cryptography  | 38.0.3 | cryptography-38.0.3.tar.gz<br>https://pypi.org/project/cryptography/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/cryptography | bfbe6ee19615b07a98b1d2287d6a6073f734735b49ee45b11324d85efc4d5cbd<br>2148f1283f22df0677e204e46bccaf06<br>13dda9608b7aebe5d2dc0c98a4b2090a6b815628efa46cc1c046b89d8cd25f4c |
    | ansible | 7.0.0 | ansible-7.0.0.tar.gz<br>https://pypi.org/project/ansible/#files<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/ansible | 73144e7e602715fab623005d2e71e503dddae86185e061fed861b2449c5618ea<br>4250dd7c6b1f35f2a1f6039b2c83f85a<br>18fd963a3328d6f72e54d17dfbb40af6e9d6827333a5f1745c79c6d81b30dc85 |
    | hatchling | 1.11.1 | hatchling-1.11.1.tar.gz<br>https://pypi.org/project/hatchling/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/hatchling | 9f84361f70cf3a7ab9543b0c3ecc64211ed2ba8a606a71eb6a473c1c9b08e1d0<br>e06cc65ac646f9b01df5406aa1f97022<br>24203e21d2bc57229822ac9fb9b314d7892c16f829f34a0eb247c55fc11e09a8 |
    | Jinja2 | 3.1.2 | Jinja2-3.1.2.tar.gz<br>https://pypi.org/project/Jinja2/<br><br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/Jinja2 | 31351a702a408a9e7595a8fc6150fc3f43bb6bf7e319770cbc0db9df9437e852<br>d31148abd89c1df1cdb077a55db27d02<br>7aff75c28576a1d900e87eb6335b063fab47a8ef3c8b4d88524c4bf78f670cce |
    | frappe-bench | 5.14.4 | frappe_bench-5.14.4.tar.gz<br>https://pypi.org/project/frappe-bench/<br>python3.10 -m pip install --no-index --find-links=/home/avc-server/bench/fuentes/frappe-bench | 99466d905986248df1975627f2a75b603e46f123168bd324e99e0cf75d25fb2e<br>504f8713c1803736d7d5277f652152bc<br>10027e7ba37859058527c70958b5abf8607100d4e6fea08d75be27e8242739f5 |

    ```sh      
    cd /home/avc-server/bench
    virtualenv --python python3.10 env

    /home/avc-server/bench/env/bin/python3.10 -m pip install frappe-bench
    bench --version
    
    bench version
    sudo reboot
    ```
    
  - 4.4  Inicia el framework
    ```sh
    su - avc-server
           
    bench init --python /home/avc-server/bench/env/bin/python3.10 --ignore-exist --frappe-branch version-14 /home/avc-server/bench --without-site
    
    cd /home/avc-server/bench
    bench --version
    /home/avc-server/bench/env/bin/gunicorn --version        // (Debe ser >=: gunicorn (version 20.1.0)
    ```
    
  - 4.5 Ingresa con el usuario avc-server
    ```sh
    su - avc-server
    cd /home/avc-server/bench
    ```
    

  - 4.7 Descarga la versión 14 desde GitHub ó del filesystem
      ```sh
      
      bench get-app git@github.com:Interconectando/interconectando_avc_server.git --branch intercam-version-14
      bench get-app /local/avc-server/fuentes/intercam_avc_server.zip --branch intercam-version-14
     
      ./env/bin/pip install -e apps/frappe/
      ./env/bin/pip install -e apps/interconectando_avc_server/
      ./env/bin/pip install wheel                                         # Successfully installed numpy-1.23.4
      ./env/bin/pip install numpy                                         # Successfully installed numpy-1.23.4
      ./env/bin/pip install pandas                                        # Successfully installed pandas-1.5.1
      ./env/bin/pip install statistics                                    # Successfully installed docutils-0.19 statistics-1.0.3.5
      ```
      
 - 4.9 Habilita DNS multitenant
        ```sh
        bench config dns_multitenant on
        ```

 - 4.8 Creación de sitio, y configuraciones básicas
      ```sh
      bench new-site interval.intercam.com.mx --db-root-password intercamsf --admin-password IntercamSF --verbose --set-default
      bench use interval.intercam.com.mx
      bench --site interval.intercam.com.mx add-to-hosts
      bench --site interval.intercam.com.mx set-config developer_mode 0
      bench --site interval.intercam.com.mx enable-scheduler
      bench setup add-domain --site interval.intercam.com.mx interval.intercam.com.mx
      bench --site interval.interconectando.me install-app interconectando_avc_server
      bench config set-common-config -c scheduler_tick_interval 1
      bench config http_timeout 7200

      bench --site interval.intercam.com.mx add-user --first-name León --last-name Sedano --password Interval-123 lsedano@intercam.com.mx
      ```

    - Borra sólo el contenido del archivo:    <----- Solo para Sandbox
        ```sh
        # vi sites/currentsite.txt            <----- Solo para Sandbox
        ```

  
      
      
      ```sh
      
      sudo systemctl enable supervisord
      sudo systemctl start supervisord
      sudo systemctl status supervisord
    
      
      sudo systemctl enable redis
      sudo systemctl start redis
      sudo systemctl status redis
      
      sudo systemctl enable --now mariadb
      sudo systemctl start mariadb.service
      sudo systemctl status mariadb.service
      
      ### chkconfig nginx on
      sudo systemctl enable nginx
      sudo systemctl start nginx
      sudo systemctl status nginx
      
      bench setup redis
      bench setup supervisor --yes
      bench setup nginx
    
      
      ```
      
  - 4.10 Configuración de **SELinux**
      ```sh
      chmod 755 /home/avc-server
      chcon -t httpd_config_t config/nginx.conf
      sudo setsebool -P httpd_can_network_connect=1
      sudo setsebool -P httpd_enable_homedirs=1
      sudo setsebool -P httpd_read_user_content=1
      ```
      
  - 4.11 Configuración de **Supervsord**
      ```sh
      sudo rm /etc/supervisord.d/frappe-bench.ini
      sudo ln -s `pwd`/config/supervisor.conf /etc/supervisord.d/frappe-bench.ini
      ```
      
  - 4.12 Configura NGINX con las configuraciones del sitio creado
      ```sh
      sudo rm -f /etc/nginx/sites-enabled/*
      sudo rm -f /etc/nginx/sites-available/*
      
      sudo rm -f /etc/nginx/conf.d/frappe-bench.conf
      sudo ln -s `pwd`/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf
      ```
      
  - 4.13 Reinicia los servicios
      ```sh
      sudo systemctl reload supervisord
      # sudo systemctl restart supervisord
      
      sudo systemctl reload nginx
      # sudo systemctl restart nginx
           
      sudo systemctl status supervisord
      sudo systemctl status nginx
      sudo systemctl status redis    
      sudo systemctl status mariadb    
      
         
      
          
          # sudo systemctl status nginx
          # sudo systemctl enable nginx
          # sudo systemctl start nginx
      ```
      ```
      # Accede al sitio. Debe aparecer la página de NGINX
      http://[sitio]
      ```

  - 4.14 Actualización de framework
      ```sh
      cd /home/avc-server/bench/
      bench update --reset --no-backup
      ```


  - 4.15 Compara el archivo de configuración del framework.
    ```sh
      su - avc-server
      cd /home/avc-server/bench/
      cat common_site_config.json
    ```
    
    ```sh
      {
       "background_workers": 1,
       "dns_multitenant": true,
       "file_watcher_port": 6787,
       "frappe_user": "avc-server",
       "gunicorn_workers": 9,
       "live_reload": true,
       "maintenance_mode": 0,
       "pause_scheduler": 0,
       "rebase_on_pull": false,
       "redis_cache": "redis://localhost:13000",
       "redis_queue": "redis://localhost:11000",
       "redis_socketio": "redis://localhost:12000",
       "restart_supervisor_on_update": true,
       "restart_systemd_on_update": false,
       "scheduler_tick_interval": 1,
       "serve_default_site": true,
       "shallow_clone": true,
       "socketio_port": 9000,
       "use_redis_auth": false,
       "webserver_port": 8000
      }
    ```

-------


### 5. Solución de Problemas comunes de instalación.
    
      - if js and css file is not loading on login window run the following command
        ```sh
        sudo chmod o+x /home/avc-server
        ```

      - Sí, aparece el mensaje caniuse-lite en actualización.
       ```sh
       cd /home/avc-server/bench/apps/frappe/ 
       npx browserslist@latest --update-db yarn upgrade caniuse-lite browserslist
       ```

      - Sí, aparece el mensaje /transparent_hugepage.
        ```sh
        sudo echo never > /sys/kernel/mm/transparent_hugepage/enabled
        ```
      - Algunos parametros del Kernel
        ```sh
        echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf
        echo "echo never > /sys/kernel/mm/transparent_hugepage/enabled" | sudo tee -a /etc/rc.d/rc.local
        sudo chmod 755 /etc/rc.d/rc.local
        ```
      
-------

    
### 6. Configuraciónes finales
  
  - 6.1 Instala certificado con [**Certbot**](https://certbot.eff.org/instructions?ws=nginx&os=centosrhel8) y [SNAP](https://snapcraft.io/docs/installing-snap-on-red-hat) 
      ```sh
      cd bench/
      sudo yum install snapd -y
      sudo ln -s /var/lib/snapd/snap /snap
      sudo systemctl enable --now snapd.socket
      # Espera unos 10 segundos antes del siguiente comando.  Sino error: too early for operation, device not yet seeded or device model not acknowledged
      sudo snap install core
      sudo snap refresh core
      
      sudo yum remove certbot
      sudo snap install --classic certbot
      sudo ln -s /snap/bin/certbot /usr/bin/certbot
      sudo certbot --nginx
      
      bench set-url-root interval.interconectando.me https://interval.interconectando.me
      ```
  
  - 6.2 **Failtoban**
    
    ```sh
    # No configurado or el momento
    ```
  
  - 6.3 Archivo de intercambio  (Opcional).     **<--- No usar**. Solo para ambiente Sandbox
    ```sh
    # Crea un SwapFile de 4GB
    free -m
    sudo fallocate -l 4G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab
    ```

  
  
#### 7. Puesta en operación, Revisión y monitoreo de operación.
  - Revisa el scheduler.
    ```sh
    bench doctor
    ```
    
  - Ingresa al sitio https://interval.intercam.com.mx
  

  
---

Revisión: 2.28  Fecha: 2022-11-23
