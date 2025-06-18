# ☁️ CloudInventario - Sistema de Gestión en la Nube

[![Python](https://img.shields.io/badge/Python-3.12-blue.svg)](https://www.python.org/)
[![Django](https://img.shields.io/badge/Django-4.2-green.svg)](https://www.djangoproject.com/)
[![AWS](https://img.shields.io/badge/AWS-EC2%2C%20ASG%2C%20ALB-orange.svg)](https://aws.amazon.com/)

Aplicación empresarial con despliegue en la nube, alta disponibilidad y autoescalado.

## 📦 Requisitos Previos
- Ubuntu 22.04+
- Python 3.12
- Docker (para despliegue en contenedores)
- Cuenta AWS (para despliegue en la nube)

Ejecutar los siguientes comandos para instalar dependencias:
```bash
sudo apt-get update && sudo apt-get install -y \
    build-essential \
    libpango1.0-0 \
    libgdk-pixbuf2.0-0 \
    libffi-dev \
    libcairo2 \
    weasyprint \
    python3.12 \
    python3.12-venv && sudo apt clean
```

En caso de errores: 
```bash
sudo apt install libpango-1.0-0
```
## Clonación del repositorio

```bash
git clone https://github.com/wdavilav/pos-store.git
cd pos-store
```

## Crear y activar entorno virtual

```bash
python3 -m venv env
source env/bin/activate
```


## Instalar dependencias Python

```bash
pip install -r deploy/txt/requirements.txt
```
🎯 Resumen del error:
```bash
Error: pg_config executable not found.
```
Esto quiere decir que pg_config (una herramienta necesaria para compilar psycopg2 desde el código fuente) no está instalada. Es parte de los paquetes de desarrollo de PostgreSQL.
Solución rápida:
```bash
sudo apt install libpq-dev
sudo apt install python3-dev build-essential
```

🎯 Resumen del error:
```bash
ERROR: Failed building wheel for Pillow
ERROR: Could not build wheels for cffi, Pillow, which is required to install pyproject.toml-based projects
```
Solución rápida:
```bash
sudo apt install -y libjpeg-dev zlib1g-dev libfreetype-dev liblcms2-dev \
                    libopenjp2-7-dev libtiff-dev libwebp-dev \
                    python3-dev build-essential libffi-dev

```
Ejecutar denuevo:
```bash
pip install -r deploy/txt/requirements.txt
```

## Ejecutar migraciones

```bash
python manage.py makemigrations
python manage.py migrate
```
🧨 El error:
```bash
ModuleNotFoundError: No module named 'pkg_resources'
```
✅ Solución rápida:
```bash
pip install setuptools
pip install --upgrade pip setuptools wheel
```
💥 El error
```bash
OSError: cannot load library 'pangoft2-1.0-0': No such file or directory.
```
✅ Solución rápida:
```bash
sudo apt update
sudo apt install -y libpangoft2-1.0-0
```
```bash
python manage.py makemigrations
python manage.py migrate
```
## Cargar datos iniciales

```bash
python manage.py shell --command='from core.init import *'
python manage.py shell --command='from core.utils import *'
```

## Automatizar con servicio systemd

> **Nota:** Asegúrate de que tu aplicación esté modificada para ejecutarse con su propia configuración de base de datos y otros ajustes necesarios. Este ejemplo asume que la aplicación está configurada para ejecutarse.

### Permisos para ejecutar en el puerto 80

Para permitir que manage.py se ejecute por el puerto 80, es necesario permitir que tu usuario tenga acceso a manejar puertos menores a 1024.

Para ello, puedes usar `setcap` para otorgar permisos al ejecutable de Python:

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.12
```

### Crear el servicio systemd

Cree el archivo `/etc/systemd/system/pos-store.service` con el siguiente comando: **(ASEGURATE DE QUE EL USUARIO Y LAS RUTAS COINCIDAN CON TU CONFIGURACIÓN)**

> Si no sabes que usuario usar, puedes verificarlo con el comando `whoami`. Para saber la ruta de tu Proyecto, puedes usar `pwd` dentro del directorio del proyecto.

```bash
APP_USER="ubuntu" # Cambia esto al usuario que ejecutará la aplicación
APP_DIR="/home/$APP_USER/pos-store" # Cambia esto, si usaste una ruta diferente al clonar el repositorio

sudo bash -c "cat > /etc/systemd/system/pos-store.service <<EOF
[Unit]
Description=pos-store Django App
After=network.target

[Service]
User=$APP_USER
WorkingDirectory=$APP_DIR
Environment=PATH=$APP_DIR/env/bin
ExecStart=$APP_DIR/env/bin/python $APP_DIR/manage.py runserver 0.0.0.0:8000
Restart=always

[Install]
WantedBy=multi-user.target
EOF"


```

## Habilitar y levantar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable pos-store.service
sudo systemctl start pos-store.service
```

## Verificar estado del servicio

```bash
sudo systemctl status pos-store.service
```
#¡La aplicación debería estar corriendo!



# 🐳 Despliegue de la aplicacion en Docker con EC2

## 🖥️ Requisitos del sistema

- Ubuntu Server 22.04 
- Acceso a usuario con privilegios `sudo`
- Puerto 80 disponible
---

## ⚙️ Instalación de Docker en Ubuntu 22.04

### 1. Agrega el repositorio oficial de Docker

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### 2. Instala Docker y Docker Compose

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

---

## 🐳 Contenerización Local con Docker

### 1. ir al proyecto

```bash
cd pos-store
```

### 2. Crea un `Dockerfile`

Crea un archivo llamado `Dockerfile` en la raíz del proyecto usando `nano Dockerfile` con el siguiente contenido:

```dockerfile
FROM python:3.10-slim

RUN apt-get update && apt-get install -y \
        build-essential \
        libpango1.0-0 \
        libgdk-pixbuf2.0-0 \
        libffi-dev \
        libcairo2 \
        && apt-get clean

WORKDIR /app

COPY deploy/txt/requirements.txt /app/requirements.txt
RUN pip install --upgrade pip && pip install -r requirements.txt

COPY . .

COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

EXPOSE 8080

ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["python", "manage.py", "runserver", "0.0.0.0:8080"]
```

### 3. Crea y configura `entrypoint.sh`

Crea un archivo llamado `entrypoint.sh` en la raíz del proyecto con `nano entrypoint.sh`

```bash
#!/bin/bash

# Migraciones
echo "📦 Ejecutando makemigrations y migrate"
python manage.py makemigrations
python manage.py migrate 

# Carga de datos iniciales
echo "📥 Cargando datos iniciales"
python manage.py shell --command='from core.init import *'
echo "📦 Cargando datos de ejemplo"
python manage.py shell --command='from core.utils import *'

# Arrancar servidor
echo "🚀 Iniciando servidor Django"
exec "$@"
```

Hazlo ejecutable:

```bash
chmod +x entrypoint.sh
```

### 4. Construye la imagen Docker

```bash
docker build -t pos-store:latest .
```

### 5. Ejecuta el contenedor con SQLite
Antes de ejecutar el contenedor 
hay que cerrar el servicio de systemd:
```bash
sudo systemctl stop pos-store.service
```
Ahora ejecutar:
```bash
docker run -it --rm -p 80:8080 pos-store:latest
```

Accede a la app en [http://localhost:80](http://localhost:80).
---
## ☁️ Despliegue Automático con Docker Compose en AWS EC2

### 1. Instala Docker en tu instancia EC2

Conéctate por SSH y ejecuta los pasos de instalación de Docker descritos arriba.

### 2. Instala Docker Compose

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. Crea el archivo `docker-compose.yml`

En la raíz del proyecto, crea un archivo `docker-compose.yml` con el siguiente contenido:

```yaml
version: '3.8'

services:
    web:
        image: pos-store:latest
        ports:
            - "80:8080"
        restart: always
```

### 4. Ejecuta Docker Compose

```bash
sudo apt install docker-compose
sudo docker-compose up -d
```

### 5. Accede a la app

Abre tu navegador y accede a la aplicación Django en la IP pública de tu instancia EC2:  
`http://<IP-PUBLICA-EC2>`



# Despliegue HA + Auto Scaling para pos-store en AWS

---

## 🖥 Crear AMI personalizada

1. **Crear la imagen:**
    - Ir a **Consola EC2 > Instancias**.
    - Seleccionar la instancia y elegir **Acciones > Imagen y plantillas > Crear imagen**.
    - Nombre: `certamen-3-imagen`.
    - Crear la imagen.

---

## 📄 Crear Plantilla de Lanzamiento (Launch Template)

1. Ir a **EC2 > Instancias > Plantillas de lanzamiento > Crear plantilla de lanzamiento**.
2. Configurar:
    - **Nombre:** `lt-pos-store`
    - **Orientación sobre Auto Scaling:** Sí
    - **AMI:** Seleccionar la AMI creada. (en Mis AMI)
    - **Tipo de instancia:** `t2.small`.
    - **Clave SSH:** Seleccionar la clave SSH que usarás para acceder a las instancias.
    - **Configuración de red:**
        - **Subred:** No incluir subred (para que el ASG elija automáticamente).
        - **Grupo de seguridad:** Crear uno nuevo o seleccionar uno existente que permita tráfico HTTP (puerto 80).

3. Crear la plantilla de lanzamiento.

---

## 🔁 Crear Auto Scaling Group (ASG)

1. Ir a **EC2 > Auto Scaling Groups > Crear Auto Scaling Group**.
2. Configurar:
    - **Nombre:** `asg-certamen3`
    - **Plantilla de lanzamiento:** `asg-certamen3-plantilla`
3. **Elegir las opciones de lanzamiento:**
    - **VPC:** Seleccionar la VPC donde se encuentra la plantilla de lanzamiento.
    - **Zonas de disponibilidad:** Seleccionar al menos 2 zonas para alta disponibilidad.
    - **Distribución de zonas de disponibilidad:** Mejor esfuerzo equilibrado.
4. **Integrar en otros servicios:**
    - **Balanceador de carga:** Asociar a un nuevo balanceador de carga.
    - **Tipo de balanceador de carga:** Application Load Balancer (ALB).
    - **Nombre del balanceador de carga:** `alb-certamen3`.
    - **Esquema del balanceador de carga:** Internet-facing.
    - **Zonas de disponibilidad:** Seleccionar las mismas que el ASG (Están seleccionadas automáticamente).
    - **Agentes de escucha y direccionamiento:**
        - **Puerto de escucha:** 80 (HTTP).
        - **Grupo de destino:** Crear uno nuevo con nombre `asg-certamen3-1`.
    - **Comprobaciones de estado:**
        - **Activar las comprobaciones de estado de Elastic Load Balancing:** `check`.
5. **Configurar escalamiento y tamaño del grupo:**
    - **Tamaño del grupo:** Mínimo 2, deseado 2, máximo 5.
    - **Política de escalamiento:** Basada en CPU.
    - **Política de mantenimiento:** Lance antes de terminar.
6. **Revisar y crear el ASG con Balanceador de Carga.**

---

##  ✅ 1. Verificar que el ALB funciona
Ve a la Consola EC2 > Balanceadores de Carga > Application Load Balancer (ALB).

Copia la DNS del ALB, que se ve algo así como:
```bash
alb-certamen3-1234567890.us-east-1.elb.amazonaws.com
```
Abre esa URL en tu navegador:

Si carga la app Django: ¡funciona!

##✅ 2. Verificar que las instancias estén saludables
Ve a EC2 > Auto Scaling Groups > asg-certamen3.

En la pestaña “Instancias”:

¿Tienes al menos 2 instancias corriendo?

¿Estado = “InService” y comprobación de estado = ✅ OK?

En el grupo de destino (asg-certamen3):

Todas las instancias deben estar marcadas como healthy.
- Verifica que la aplicación responde correctamente y que el tráfico se distribuye entre instancias.
- Simula carga para probar el escalado automático.

---
