# Guía de Instalación y Configuración de Git en Ubuntu Server

Esta guía detalla cómo instalar Git y configurarlo de forma segura para clonar y actualizar repositorios utilizando GitHub CLI (`gh`). Este método evita el uso de contraseñas manuales y automatiza la creación de claves SSH.

## 1. Instalación de Git

### Paso 1.1: Instalar el paquete

Asegúrate de que los repositorios estén actualizados y descarga Git:

```bash
sudo apt update
sudo apt install git -y
```

### Paso 1.2: Verificar la instalación

Comprueba la versión instalada para confirmar que todo está correcto:

```bash
git --version
```

## 2. Configuración Global Básica

Configura tus datos personales. Estos datos quedarán registrados si llegas a hacer _commits_ directamente desde el servidor (aunque lo habitual en un servidor de producción es solo hacer _pull_).

```bash
# Configurar tu nombre
git config --global user.name "Tu Nombre"

# Configurar tu correo electrónico
git config --global user.email "tu@correo.com"

# Forzar el nombre de la rama principal por defecto a 'main'
git config --global init.defaultBranch main
```

Para revisar que la configuración se guardó correctamente:

```bash
git config --list
```

## 3. Instalación de GitHub CLI (`gh`)

Para evitar crear claves SSH manualmente, usaremos la herramienta oficial de línea de comandos de GitHub, la cual automatiza todo el proceso de conexión.

Ejecuta el siguiente bloque de comandos para instalar `gh` en Ubuntu:

```bash
(type -p wget >/dev/null || (sudo apt update && sudo apt-get install wget -y)) \
&& sudo mkdir -p -m 755 /etc/apt/keyrings \
&& wget -qO- [https://cli.github.com/packages/githubcli-archive-keyring.gpg](https://cli.github.com/packages/githubcli-archive-keyring.gpg) | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
&& sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] [https://cli.github.com/packages](https://cli.github.com/packages) stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
&& sudo apt update \
&& sudo apt install gh -y
```

## 4. Autenticación Interactiva (Login)

Ahora vincularemos el servidor con tu cuenta de GitHub de forma interactiva. Ejecuta:

```bash
gh auth login
```

El asistente te hará una serie de preguntas. Selecciona las siguientes opciones utilizando las flechas de tu teclado:

1. **What account do you want to log into?** `GitHub.com`
2. **What is your preferred protocol for Git operations on this host?** `SSH`
3. **Generate a new SSH key to add to your GitHub account?** `Yes`
4. **Enter a passphrase for your new SSH key (Optional):** _(Presiona Enter para dejarlo en blanco, esto es vital para despliegues automáticos)_
5. **Title for your SSH key:** _(Presiona Enter para usar el nombre sugerido o escribe uno como "Ubuntu Server")_
6. **How would you like to authenticate GitHub CLI?** `Login with a web browser`

En este punto, la consola te mostrará un **código de 8 dígitos** (ej. `A8F3-91C2`) y te pedirá que abras la dirección `https://github.com/login/device`.

- Ve a tu PC o celular.
- Abre esa URL en el navegador donde tienes tu sesión de GitHub iniciada.
- Pega el código de 8 dígitos y autoriza la aplicación.

¡Listo! `gh` se encargará de crear la clave SSH y agregarla a tu cuenta de GitHub automáticamente.

### Paso 4.1: Probar la conexión

Puedes verificar que tu servidor quedó autenticado correctamente con:

```bash
gh auth status
```
