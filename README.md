# Proyecto de self hosting

## Introducción

¡Bienvenido al proyecto de Servidor Web con Nginx y Ansible en Vagrant!
Este proyecto despliega un servidor web autogestionado utilizando tecnologías como Vagrant, Ansible, y Nginx. Incluye funcionalidades como:

-Autenticación básica para páginas protegidas.
-Generación de certificado proporcionado por Let's Encrypt.
-Túnel público con Ngrok para acceder desde cualquier lugar.
-Monitorización en tiempo real con Netdata.
-Pruebas de rendimiento.

Además, el servidor incluye páginas personalizadas, como una página principal, una de administración, y una página de error 404.

## Tabla de contenidos:

1. [Requisitos Previos](#1-requisitos-previos)
2. [Estructura del Proyecto](#2-estructura-del-proyecto)
3. [Tecnologías Utilizadas](#3-tecnologías-utilizadas)
4. [Certificado proporcionado por Let's Encrypt](#4-certificado-proporcionado-por-let's-encrypt)
   - [Instalar Certbot](#instalar-certbot)
   - [Configurar el certificado SSL](#configurar-el-certificado-ssl)
   - [Permitir HTTPS a través del firewall](#permitir-https-a-través-del-firewall)
   - [Obtener el certificado SSL](#obtener-el-certificado-ssl)
   - [Comprobación de Certificado](#comprobación-de-certificado)
5. [Autenticación básica](#5-autenticación-básica)
6. [Página de Error Personalizada](#6-página-de-error-personalizada)
7. [Status del servidor](#7-status-del-servidor)
8. [Pruebas de rendimiento](#8-pruebas-de-rendimiento)

## 1. Requisitos previos

Antes de comenzar, asegúrate de tener instalado y configurado lo siguiente en tu máquina:

### VirtualBox:

-Versión recomendada: 7.0 o superior.

### Vagrant:

-Herramienta para gestionar máquinas virtuales.

### Conexión a Internet:

-Necesaria para descargar dependencias y realizar configuraciones automáticas.

## 2. Estructura del Proyecto

```bash
vagrant-ansible-nginx/
├── Vagrantfile
├── ansible/
│   ├── playbook.yml                # Playbook principal para Ansible.
│   ├── roles/
│   │   ├── nginx/
│   │   │   ├── handlers/
│   │   │   │   └── main.yml  
│   │   │   ├── tasks/
│   │   │   │   └── main.yml        # Tareas de configuración de Nginx.
│   │   │   ├── templates/
│   │   │   │   └── mi_web.conf.j2  # Configuración personalizada de Nginx.
│   │   │   └── files/              # Archivos HTML, CSS y recursos.              
└── README.md
```

## 3. Tecnologías Utilizadas

Este proyecto combina varias herramientas modernas para la automatización y el despliegue eficiente de un servidor web. A continuación, se enumeran:

-Vagrant
    -Provisión de entornos virtuales portátiles.

-Ansible
    -Herramienta de automatización para la configuración y gestión de servidores.

-Nginx
    -Servidor web y proxy inverso de alto rendimiento.

-Ngrok
    -Permite exponer el servidor local a través de un túnel seguro accesible desde cualquier parte del mundo.

-Let's Encrypt
    -Generación de certificados SSL gratuitos para asegurar la conexión HTTPS.

-Netdata
    -Sistema de monitorización en tiempo real para recursos del servidor.

## 4. Certificado proporcionado por Let's Encrypt

### Instalar Certbot

``` bash
sudo apt install snapd
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### Configurar el certificado SSL

``` bash
sudo nano /etc/nginx/sites-available/mi_web
```

``` /etc/nginx/sites-available/mi_web
...
ServerName mi_web;
...
```

``` bash
sudo nginx -t
sudo systemctl reload nginx
```

### Permitir HTTPS a través del firewall

``` bash
sudo ufw allow 'WWW Full'
sudo ufw delete allow 'WWW'
```

### Obtener el certificado SSL

``` bash
sudo certbot --apache -d mi_web -d www.mi_web
```

### Comprobación de Certificado

En el navegador, debemos comprobar el emisor del certificado:

![Certificado Let's Encrypt](./img/lets-encrypt.png)

## 5. Autenticación Básica

La autenticación básica en este servidor está configurada a través de archivos .htpasswd y se utiliza para proteger áreas específicas de la web, como el panel de administración y el estado del servidor.

En este caso, se ha configurado para las siguientes rutas:

-/admin: Un área de administración protegida.
-/status: Una página de estado del servidor.

### Configuración en Ansible

``` main.yml
- name: Create .htpasswd files
  shell: |
    echo -n 'admin:' >> /etc/nginx/.htpasswd_admin
    echo "$(openssl passwd -apr1 'asir')" >> /etc/nginx/.htpasswd_admin
    echo -n 'sysadmin:' >> /etc/nginx/.htpasswd_status
    echo "$(openssl passwd -apr1 'risa')" >> /etc/nginx/.htpasswd_status
  args:
    creates: /etc/nginx/.htpasswd_admin
```

### Archivo de configuración Nginx

La autenticación básica en Nginx se configura en los archivos de configuración de los sitios. La directiva auth_basic se utiliza para activar la protección, y la directiva auth_basic_user_file señala el archivo .htpasswd correspondiente.

En el archivo de configuración de Nginx, /etc/nginx/sites-available/mi_web, se incluye lo siguiente para proteger las rutas /admin y /status:

``` 
  location /admin {
    auth_basic "Área restringida";
    auth_basic_user_file /etc/nginx/.htpasswd_admin;
    try_files /admin.html =404; 
  }

  location /status {
    proxy_pass http://localhost:19999;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    auth_basic "Área restringida";
    auth_basic_user_file /etc/nginx/.htpasswd_status;

    location /status/api {
        proxy_pass http://localhost:19999/api;
    }
  }
```

## 6. Página de Error Personalizada

Una página de error personalizada es una forma de ofrecer una experiencia de usuario más amigable y controlada cuando algo no sale como se esperaba. En este proyecto, hemos configurado una página de error personalizada para el código HTTP 404 (página no encontrada). De esta manera, si un usuario intenta acceder a una página que no existe en el servidor, en lugar de mostrar una página de error estándar del servidor, se le mostrará una página de error diseñada especialmente para ello.

![Error 404](./img/error404.png)

## 7. Status del servidor

En un entorno de servidores web, es útil tener una página que permita visualizar el estado del servidor en tiempo real. Esto puede ser importante para monitorear el rendimiento, la carga del servidor, y otros indicadores clave de salud del sistema. En este proyecto, se ha implementado una página de status del servidor que muestra información relevante sobre el estado del servidor, como estadísticas de recursos y métricas del sistema.

### Visualización de Netdata en /status

![Status](./img/status1.png)

## 8. Pruebas de Rendimiento

Una de las herramientas más sencillas y eficaces para realizar pruebas de rendimiento en un servidor web es ApacheBench (ab). Esta herramienta permite simular múltiples usuarios concurrentes y medir el rendimiento de la respuesta del servidor bajo diferentes cargas.

En este caso, realizaremos las pruebas de rendimiento desde un ordenador distinto al servidor web, utilizando dos configuraciones de carga diferentes:

-100 usuarios concurrentes con 1000 peticiones.
-1000 usuarios concurrentes con 10000 peticiones.

### Objetivos de las Pruebas

Se realizarán las pruebas sobre los siguientes recursos:

-Página principal: https://example.com/ (con SSL/TLS habilitado).
-Recurso estático: https://example.com/logo.png.
-Página de administración: https://example.com/admin/ (acceso con autenticación básica).

### Parámetros de la Prueba

Los parámetros que se utilizarán son los siguientes:

-k: Mantiene la conexión viva entre el cliente y el servidor, lo que permite reutilizar la misma conexión para múltiples solicitudes HTTP en lugar de abrir una nueva para cada una, mejorando la eficiencia.

-H "Accept-Encoding: gzip, deflate": Indica al servidor que se comprima la salida de datos. Esta cabecera permite que el servidor envíe los datos comprimidos, lo que puede reducir el tamaño de la respuesta y mejorar los tiempos de carga. Se estima que esta compresión puede reducir el tamaño de los datos de un 25% a un 75%.

### Realización de las Pruebas

#### Para 100 usuarios concurrentes y 1000 peticiones

1. Página principal:

``` bash
ab -n 1000 -c 100 -k https://albacore-select-marmoset.ngrok-free.app/
```

2. Recurso estático (logo.png):

```bash
ab -n 1000 -c 100 -k https://albacore-select-marmoset.ngrok-free.app/logo.png
```

3. Página de administración (/admin):
```bash
ab -n 1000 -c 100 -k -A sysadmin:risa https://albacore-select-marmoset.ngrok-free.app/admin/
```

#### Para 1000 usuarios concurrentes y 10000 peticiones

1. Página principal:

``` bash
ab -n 10000 -c 1000 -k https://albacore-select-marmoset.ngrok-free.app/
```

2. Recurso estático (logo.png):

```bash
ab -n 10000 -c 1000 -k https://albacore-select-marmoset.ngrok-free.app/logo.png
```

3. Página de administración (/admin):
```bash
ab -n 10000 -c 1000 -k -A sysadmin:risa https://albacore-select-marmoset.ngrok-free.app/admin/
```

En los comandos anteriores:
    -n especifica el número total de peticiones que se realizarán (1000 o 10000).
    -c establece el número de conexiones concurrentes (100 o 1000).
    -k habilita la opción Keep-Alive, que mantiene abiertas las conexiones entre el cliente y el servidor.
    -H "Accept-Encoding: gzip, deflate" agrega la cabecera que solicita la compresión de la respuesta del servidor.
