# Ansible_wordpress

En esta práctica se realizará la adaptación de la instalación del Wordpress en 2 niveles que se llevó a cabo en la práctica 9, en Ansible.
Para ello se tendrá una serie de playbooks para ejecutarlos desde un mismo nodo de control sin tener que estar accediendo por ssh a cada una de las máquinas salvo
para realizar alguna que otra prueba inicial de conexión. Nuevamente tendremos los **Deploys** para el **backend** y **frontend**, asi como los **installLamp** para cada uno de ellos, y un config_https para el **cerbot**. Al final se tendrá un archivo **main.yml** con las rutas de **cada playbook** para ejecutarlos **todos a la vez** cuando se sepa que **individualmente** están **correctamente configurados**.

## 1. Configuración del Frontend (InstallLampFrontend.yml):

En esta parte de aquí da comienzo el playbook donde se le establece el nombre del grupo en **hosts** para indicar a que grupo de instancias queremos ejecutarlo,
y se le pone a **become** el valor **yes** para permitir que escale privilegios.

```
---
- name: Playbook para instalar la pila LAMP
  hosts: frontend
  become: yes
```
Mediante las siguientees sentencias procedo a **actualizar** los repositorios:
```
  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes
```
Aqui procedo con la **instalación** del servidor web **Apache**
```
    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present
```
Instalar **PHP** y los **módulos necesarios**:
```
    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
        state: present
```
**Reiniciar** el servidor web **Apache**:

```
    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```
**Copiar** el **archivo de configuración** de apache hacia el **/var/www/html**
```
    - name: Copiar el archivo de conf de apache
      copy:
        src: /home/ubuntu/Ansible_wordpress/practica3/templates/000-default.conf
        dest: /var/www/html/
        mode: 0755
```
**Copiar** el archivo **phpinfo.php**
```   
    - name: Copiar el archivo phpinfo.php
      copy:
        src: /home/ubuntu/Ansible_wordpress/practica3/php/phpinfo.php
        dest: /var/www/html/
        mode: 0755
```
## 1.1 Configuración del Frontend (DeployFrontend.yml):

En esta parte de aquí da comienzo el playbook donde se le establece el nombre del grupo en **hosts** para indicar a que grupo de instancias queremos ejecutarlo,
y se le pone a **become** el valor **yes** para permitir que escale privilegios.
```
---
- name: Playbook para hacer el deploy de la aplicación web Wordpress
  hosts: frontend
  become: yes
```
En esta parte de aqui se realiza la **importación de las variables**, equivalente al **source**

```
  vars_files:
    - ../vars/variables.yml
```

Aquí empezamos definiendo el **tasks**, que hace referencia a la **lista principal** de los pasos que serán ejecutados por un rol específico:

Empezamos eliminando descargar previas mediante el módulo **file** con el estado **absent**

```
  tasks:

    - name: Eliminar descargas previas
      file:
        path: /tmp/wp-cli.phar 
        state: absent
 ```
Mediante el módulo **get_url** descargamos la utilidad **wp-cli** para la instalación sin interfaz gráfica, en la **carpeta temporal** y procedo a darles los permisos de ejecución:

```
    - name: Descargamos utilidad wp-cli
      get_url:
        url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
        dest: /tmp
        mode: "+x"
 ```
Movemos la utilidad y de esta forma lo podemos usar **sin poner la ruta completa**
 ```
    - name: Movemos la utlidad y de esta forma lo podemos usar sin poner la ruta completa
      command: mv /tmp/wp-cli.phar /usr/local/bin/wp
```
Eliminamos **instalaciones previas** del **Wordpress** (mediante **shell** porque con el modulo **file** no las borraba)
```  
    - name: Eliminamos instalaciones previas del Wordpress
      shell: rm -rf /var/www/html/*
 ```
Descargamos **codigo fuente** Wordpress en **/var/www/html**
 ```
    - name: Descargamos codigo fuente Wordpress en /var/www/html
      command: wp core download --locale=es_ES --path=/var/www/html --allow-root
  ```
Creamos el **archivo de configuración** de la base de datos de **Wordpress** y procedemos más abajo con su instalación, haciendo uso de **command**
```
    - name: Creamos el archivo de configuracion
      command: 
        wp config create \
        --dbname={{ Wordpress.WORDPRESS_DB_NAME }} \
        --dbuser={{ Wordpress.WORDPRESS_DB_USER }} \
        --dbpass={{ Wordpress.WORDPRESS_DB_PASSWORD }} \
        --dbhost={{ Wordpress.WORDPRESS_DB_HOST }} \
        --path=/var/www/html \
        --allow-root
```

    - name: Instalación de Wordpress
      command: wp core install 
        --url="{{ Wordpress_conf.dominio }}" 
        --title="{{ Wordpress_conf.WORDPRESS_TITLE }}" 
        --admin_user="{{ Wordpress_conf.WORDPRESS_ADMIN }}"
        --admin_password="{{ Wordpress_conf.WORDPRESS_PASS }}" 
        --admin_email="{{ Wordpress_conf.WORDPRESS_email }}"
        --path=/var/www/html 
        --allow-root
 
 Actualización:
```
    - name: Actualización
      command: wp core update --path=/var/www/html --allow-root

```
Instalación del tema **sydney**:
```
    - name: Instalación de un tema
      command: wp theme install sydney --activate --path=/var/www/html --allow-root
```
Se procede a realizar la actualización de los **plugins**

```
    - name: Actualización plugins
      command: wp plugin update --path=/var/www/html --all --allow-root 
```
Instalación del plugin **bbpress**:
```
    - name: Instalamos el plugin bbpress
      command: wp plugin install bbpress --activate --path=/var/www/html --allow-root
```
Instalamos el plugin  **wps-hide-login**

```
    - name: Instalamos el plugin  wps-hide-login
      command: wp plugin install wps-hide-login --activate --path=/var/www/html --allow-root
```
Configuración del **nombre de la entrada**

```
    - name: Configuración del nombre de la entrada
      command: 
        wp rewrite structure '/%postname%/' \
       --path=/var/www/html \
       --allow-root
```
Le configuro un **nombre personalizado** al **nombre oculto**, en mi caso "candado"
```
    - name: Le configuro un nombre personalizado al nombre oculto
      command: wp option update whl_page "candado" --path=/var/www/html --allow-root
```
Mediante las siguientes sentencias se habilita la **reescritura**:

```
    - name: Reescritura
      apache2_module:
        name: rewrite
        state: present
```
Copiar el htaccess al **/var/www/html**

```
    - name: Copiar el htaccess al /var/www/html
      copy:
        src: /home/ubuntu/Ansible_wordpress/practica3/htaccess/.htaccess
        dest: /var/www/html
        mode: 0755
```
Cambiar el propietario del directorio **/var/www/html** por **www-data**
 
 ```
    - name: Cambiar el propietario del directorio /var/www/html
      file:
        path: /var/www/html
        owner: www-data
        group: www-data
        recurse: yes
 ```
### 2 Configuración del Backend (InstallLampbackend.yml):

Comienzo del **playbook**:
```
---
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes
```
En esta parte de aqui se realiza la **importación de las variables**, equivalente al **source**
```
  vars_files:
    - ../vars/variables.yml
```
Mediante las siguientees sentencias procedo a **actualizar** los repositorios:
```
  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes
```
Instalar el sistema gestor de bases de datos **MySQL**
```
    - name: Instalar el sistema gestor de bases de datos MySQL
      apt:
        name: mysql-server
        state: present
```
Configuramos **MySQL** para **permitir conexiones** desde **cualquier interfaz**
(mediante el siguiente comando hacemos algo equivalente al comando **sed**)
```    
    - name: Configuramos MySQL para permitir conexiones desde cualquier interfaz
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: 127.0.0.1
        replace: "{{ Wordpress.MYSQL_PRIVATE }}"
 ```
Se procede a **reiniciar** el servidor web Apache:
```   
    - name: Reiniciar el servidor web Apache
      service:
        name: mysql
        state: restarted

```
### 2.1 Configuración del Backend (DeployBackend.yml):

En esta parte de aquí da comienzo el playbook donde se le establece el nombre del grupo en **hosts** para indicar a que grupo de instancias queremos ejecutarlo,
y se le pone a **become** el valor **yes** para permitir que escale privilegios.

```
---
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes
```
Importación de las **variables**:

```
  vars_files:
    - ../vars/variables.yml
```
Procedo con la instalación del gestor de paquetes de **Python pip3**

```
  tasks:

    - name: Instalamos el gestor de paquetes de Python pip3
      apt: 
        name: python3-pip
        state: present
```
Instalamos el módulo de **pymysql**

```
    - name: Instalamos el módulo de pymysql
      pip:
        name: pymysql
        state: present
```
Procedo a crear la base de datos (si quisiera borrar la base de datos previamente sería el mismo comando con el estado **absent**)
```
    - name: Crear una base de datos
      mysql_db:
        name: "{{ Wordpress.WORDPRESS_DB_NAME }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 
```
Creación del **usuario** de la **base de datos**:
```
    - name: Crear el usuario de la base de datos
      no_log: true
      mysql_user:         
        name: "{{ Wordpress.WORDPRESS_DB_USER }}"
        password: "{{ Wordpress.WORDPRESS_DB_PASSWORD }}"
        priv: "{{ Wordpress.WORDPRESS_DB_NAME }}.*:ALL"
        host: "{{ Wordpress.IP_CLIENTE_MYSQL }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 
```
### 2.2 Despliegue del cerbot (config_https.yml):

En esta parte de aquí da comienzo el playbook donde se le establece el nombre del grupo en **hosts** para indicar a que grupo de instancias queremos ejecutarlo,
y se le pone a **become** el valor **yes** para permitir que escale privilegios.

```
---
- name: Playbook para configurar HTTPS
  hosts: frontend
  become: yes
```

Importación de las variables:

```
  vars_files:
    - ../vars/variables.yml
```
Desinstalar **instalaciones previas** de **Certbot**
```
  tasks:

    - name: Desinstalar instalaciones previas de Certbot
      apt:
        name: certbot
        state: absent
```
Instalar **Certbot** con **snap**
```
    - name: Instalar Certbot con snap
      snap:                                    
        name: certbot
        classic: yes
        state: present
```
Solicitar y configurar **certificado SSL/TLS** a **Let's Encrypt** con **certbot**
```
    - name: Solicitar y configurar certificado SSL/TLS a Let's Encrypt con certbot
      command:
        certbot --apache \
        -m {{ certbot.email }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ certbot.domain }}
```
