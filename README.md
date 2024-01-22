# Ansible_wordpress

En esta práctica se realizará la adaptación de la instalación del Wordpress en 2 niveles que se llevó a cabo en la práctica 9, en Ansible.
Para ello se tendrá una serie de playbooks para ejecutarlos desde un mismo nodo de control sin tener que estar accediendo por ssh a cada una de las máquinas salvo
para realizar alguna que otra prueba inicial de conexión. Nuevamente tendremos los **Deploys** para el **backend** y **frontend**, asi como los **installLamp** para cada uno de ellos, y un config_https para el **cerbot**. Al final se tendrá un archivo **main.yml** con las rutas de **cada playbook** para ejecutarlos **todos a la vez** cuando se sepa que **individualmente** están **correctamente configurados**.

## 1. Configuración del Frontend:

En esta parte de aquí da comienzo el playbook donde se le establece el nombre del grupo en **hosts** para indicar a que grupo de instancias queremos ejecutarlo,
y se le pone a **become** el valor **yes** para permitir que escale privilegios.
```
---
- name: Playbook para hacer el deploy de la aplicación web PrestaShop
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

 ```
    - name: Movemos la utlidad y de esta forma lo podemos usar sin poner la ruta completa
      command: mv /tmp/wp-cli.phar /usr/local/bin/wp
```

```  
    - name: Eliminamos instalaciones previas del Wordpress
      shell: rm -rf /var/www/html/*
 ```

 ```
    - name: Descargamos codigo fuente Wordpress en /var/www/html
      command: wp core download --locale=es_ES --path=/var/www/html --allow-root
  ```
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
  ```
  ```
    - name: Actualización
      command: wp core update --path=/var/www/html --allow-root
```
    - name: Instalación de un tema
      command: wp theme install sydney --activate --path=/var/www/html --allow-root
```
    - name: Actualización plugins
      command: wp plugin update --path=/var/www/html --all --allow-root 
```
    - name: Instalamos el plugin bbpress
      command: wp plugin install bbpress --activate --path=/var/www/html --allow-root
```
```
    - name: Instalamos el plugin  wps-hide-login
      command: wp plugin install wps-hide-login --activate --path=/var/www/html --allow-root
```
```
    - name: Configuración del nombre de la entrada
      command: 
        wp rewrite structure '/%postname%/' \
       --path=/var/www/html \
       --allow-root
```
```
    - name: Le configuro un nombre personalizado al nombre oculto
      command: wp option update whl_page "candado" --path=/var/www/html --allow-root
```
```
    - name: Reescritura
      apache2_module:
        name: rewrite
        state: present
```
```
    - name: Copiar el htaccess al /var/www/html
      copy:
        src: /home/ubuntu/Ansible_wordpress/practica3/htaccess/.htaccess
        dest: /var/www/html
        mode: 0755
```
```
    - name: Cambiar el propietario del directorio /var/www/html
      file:
        path: /var/www/html
        owner: www-data
        group: www-data
        recurse: yes
```
