# Ansible_wordpress

En esta práctica se realizará la adaptación de la instalación del Wordpress en 2 niveles que se llevó a cabo en la práctica 9, en Ansible.
Para ello se tendrá una serie de playbooks para ejecutarlos desde un mismo nodo de control sin tener que estar accediendo por ssh a cada una de las máquinas salvo
para realizar alguna que otra prueba inicial de conexión. Nuevamente tendremos los **Deploys** para el **backend** y **frontend**, asi como los **installLamp** para cada uno de ellos, y un config_https para el **cerbot**. Al final se tendrá un archivo **main.yml** con las rutas de **cada playbook** para ejecutarlos **todos a la vez** cuando se sepa que **individualmente** están **correctamente configurados**.
