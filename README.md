# [SA]Practica1_201901816

---
## Proceso de Instalación de Terraform en macOS
Para poder instalar Terraform en macbook el proceso es algo distinto a cualquier máquina de linux, aunque las máquinas de mac son basadas en unix (hermana de linux), el proceso no es mismo y es necesario tener instalado otra herramienta

### Paso 1: Instalar Homebrew

Homebrew es un administrador de paquetes para macOS que facilita la instalación de software.
- Para poder instalarlo debemos de ingresar el siguiente comando en la terminal:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Paso 2: Instalar Terraform

Una vez que tienes Homebrew instalado, puedes usarlo para instalar Terraform.

```bash
brew install terraform
```

### Paso 3: Verificar la Instalación

Después de la instalación, puedes verificar si Terraform se instaló correctamente ejecutando:

```bash
terraform --version
```

---
## Proceso de instalación de Ansible en Linux

Ansible es una herramienta de automatización que simplifica la administración de sistemas y la configuración de infraestructuras. Aquí se detallan los pasos para instalar Ansible en una máquina Linux. Debido a que en dicha práctica ansible se utiliza en una virtual machine en la nube, esta instalación será orientada en Linux.

### Paso 1: Actualizar el sistema

Antes de instalar cualquier paquete, es recomendable actualizar el sistema para asegurarse de que todos los paquetes instalados estén en su última versión. Para ello, ejecuta el siguiente comando:

```bash
sudo apt update && sudo apt upgrade
```

### Paso 2: Instalar Ansible

Una vez que el sistema esté actualizado, puedes instalar Ansible utilizando el administrador de paquetes de tu distribución. En sistemas basados en Debian/Ubuntu, se puede instalar con el siguiente comando:

```bash
sudo apt install ansible
```

Si estás utilizando una distribución diferente, consulta la documentación oficial de Ansible para obtener instrucciones específicas.

### Paso 3: Verificar la instalación

Una vez completada la instalación, puedes verificar si Ansible se instaló correctamente ejecutando el siguiente comando:

```bash
ansible --version
```

Esto debería mostrar la versión de Ansible instalada y otra información relevante.

### Paso 4: Configuración inicial (opcional)

Ansible no requiere mucha configuración para empezar a usarlo. Sin embargo, puedes ajustar la configuración según tus necesidades en el archivo de configuración `ansible.cfg`, que suele encontrarse en `/etc/ansible/ansible.cfg`.

---
## Como Funciona Terraform y Como se Utiliza

Para poder utilizar terraform debemos de tener dos archivos .tf, uno que sirve como provider, donde colocamos el proveedor de máquinas virtuales, en este caso puede ser google cloud o aws.

En este ejemplo se crearan máquinas virtuales en google cloud, y para eso necesitamos crear una cuenta de servicio y otorgarle los accesos necesarios, luego descargamos el json.key que servirá para que terraform tenga permisos para la creación de máquinas virutales.

`/terraform/provider.tf`
```js
provider "google" {
    credentials = file("key.json")
    project = "numeric-advice-342702"
}
```

Luego creamos un archivo `/terraform/main.tf` donde colocamos el contenido y las configuraciones necesarias para poder crear la máquina virtual y para poder ejecutar instrucciones dentro de la máquina virtual.

```js
resource "google_service_account" "default" {
  account_id   = "my-custom-sa"
  display_name = "Custom SA for VM Instance"
}

resource "google_compute_instance" "default" {
  name         = "soa-instance"
  machine_type = "n2-standard-2"
  zone         = "us-central1-a"

  tags = ["foo", "bar"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      labels = {
        my_label = "value"
      }
    }
  }

  // Local SSD disk
  scratch_disk {
    interface = "NVME"
  }

  network_interface {
    network = "default"
    subnetwork = "default"

    access_config {
      // Ephemeral public IP
    }
  }

  metadata = {
    foo = "bar"
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    sudo apt-get update
    sudo apt-get install -y git
    sudo git clone https://github.com/AngelArteaga1/-SA-Practica1_201901816.git /github
    sudo apt-get install -y ansible
    sudo ansible-playbook /github/ansible/playbook.yml
  EOF

  service_account {
    scopes = ["cloud-platform"]
  }

}

resource "google_compute_firewall" "allow-http" {
  name    = "allow-http"
  network = "default"

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  source_ranges = ["0.0.0.0/0"]
}
```

Despues de tener los archivo de configuración para poder ejecutar terraform debemos de iniciar un proyecto en la raiz de donde se situen los archivos que creamos `/terraform/`

Para poder iniciar el proyecto ejecutamos el siguiente comando:

```bash
terraform init
```

Terraform nos creará algunos archivos, esto es indicio de que todo funcionó correctamente

Luego para agregar los archivos al proyecto o para agregar cualquier cambio se debe de ejecutar el siguiente comando:

```bash
terraform plan
```

Cuando se hayan añadido los cambios podemos ejecutar el proyecto y crear las vms en el servicio que hayamos configurado, para poder realizar eso debemos de ejecutar el siguiente comando:

```bash
terraform apply --auto-approve
```

Y listo! ya tendríamos creadas las vm en el proveedor de servicios seleccionado

_(Paso opcional) Para eliminar las vms creadas por terraform no es necesario acceder a google cloud y borrarlas desde su interfaz gráfica, podemos eliminarlas con el siguiente comando:_

```bash
terraform destroy
```

---
## Como Funciona Ansible y Como se Utiliza

Ansible es una herramienta que funciona para poder realizar configuraciones en servidores externos o internos, sin necesidad de tener que ingresar a la consola y hacerlo manualmente.

Para poder realizar dichas configuraciones hacemos uso de archivos `.yml` los cuales se llaman "playhooks", estos playhooks son archivos donde definimos acciones que podemos realizar como:

- Ejecutar comandos en consola
- Exportar archivos
- Importar archivos
- Movear archivos internos
- Configurar aplicaciones o servidores

En este ejemplo vamos a instalar nginx y vamos a mover un archivo `index.html` al `index.html` que tiene por defaul nginx, así podremos desplegar nuestra página web sin tener que instalar nginx a mano y mover el archivo a la ubicación predeterminada.

Primero debemos de crear un archivo `playbook.yml` con el siguiente código:

```yml
---
- name: Instalar Nginx localmente
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Instalar Nginx
      package:
        name: nginx
        state: present

    - name: Copiar el archivo index.html
      copy:
        src: /github/frontend/index.html
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: '0644'

    - name: Copiar el archivo favicon.ico
      copy:
        src: /github/frontend/favicon.ico
        dest: /var/www/html/favicon.ico
        owner: root
        group: root
        mode: '0644'

    - name: Reiniciar Nginx
      service:
        name: nginx
        state: restarted
```
Este archivo lo que hace es realizar las acciones que le definamos, primero realizamos la instalación de nginx, luego movemos el archivo `index.html` a la carpeta destino de nginx y por último reiniciamos el servidor para que tome los cambios necesarios.

---
## Definición de Tecnologias Utilizadas

## Terraform
Terraform es una herramienta de código abierto desarrollada por HashiCorp que permite la creación, gestión y provisionamiento de infraestructuras de manera declarativa mediante código. Utiliza configuraciones escritas en un lenguaje específico llamado HashiCorp Configuration Language (HCL) o en formato JSON para describir los recursos de infraestructura necesarios, como máquinas virtuales, redes, y servicios en la nube. Terraform se utiliza ampliamente en la automatización de la infraestructura y en la implementación de la infraestructura como código (IaC).

## Ansible
Ansible es una herramienta de automatización de TI de código abierto que permite la gestión de configuraciones, el despliegue de aplicaciones y la orquestación de tareas en infraestructuras de tecnología de la información. Utiliza un enfoque basado en agentes SSH para ejecutar módulos en los sistemas remotos. Ansible utiliza un lenguaje simple llamado YAML para describir las configuraciones y tareas, lo que facilita su uso y comprensión. Es ampliamente utilizado en DevOps y en la automatización de la administración de sistemas.

## Git
Git es un sistema de control de versiones distribuido de código abierto, diseñado para gestionar proyectos de software de manera eficiente y colaborativa. Permite a los desarrolladores realizar un seguimiento de los cambios en el código fuente, coordinar el trabajo en equipo, y facilitar la integración continua y la entrega continua (CI/CD). Git utiliza un modelo de ramificación y fusión flexible que permite a los equipos trabajar en paralelo en diferentes características o versiones del proyecto. Es ampliamente utilizado en la industria del desarrollo de software y es la base de plataformas populares como GitHub y GitLab.

## Google Cloud
Google Cloud es una plataforma de servicios en la nube ofrecida por Google que proporciona una variedad de servicios de infraestructura, plataformas y herramientas para el desarrollo, implementación y gestión de aplicaciones y servicios en la nube. Incluye servicios de cómputo, almacenamiento, bases de datos, análisis de datos, inteligencia artificial, aprendizaje automático, Internet de las cosas (IoT), seguridad, y más. Google Cloud permite a las organizaciones escalar sus operaciones de manera flexible, pagar solo por los recursos que utilizan y acceder a tecnologías innovadoras para impulsar la transformación digital.

## VM
VM (Virtual Machine) se refiere a una máquina virtual, que es un entorno de software que emula una computadora física y permite ejecutar sistemas operativos y aplicaciones como si estuvieran en hardware físico independiente. Las máquinas virtuales se utilizan para consolidar servidores físicos, facilitar la portabilidad de aplicaciones, realizar pruebas de software, y crear entornos de desarrollo aislados. Pueden ser alojadas en servidores físicos o en la nube, y se gestionan mediante software de virtualización como VMware, Hyper-V, VirtualBox, y soluciones de nube como Amazon EC2 y Google Compute Engine.

## Brew
Brew (también conocido como Homebrew) es un sistema de gestión de paquetes de código abierto para macOS y Linux que simplifica la instalación y gestión de software de código abierto y de otros tipos. Permite a los usuarios instalar rápidamente herramientas de desarrollo, utilidades de línea de comandos, aplicaciones y bibliotecas mediante simples comandos en la terminal. Brew utiliza repositorios de fórmulas para almacenar y distribuir paquetes de software, y ofrece una amplia gama de paquetes disponibles para su instalación con un solo comando.
