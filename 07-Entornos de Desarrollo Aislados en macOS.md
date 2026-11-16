# 🐳 Entornos de Desarrollo Aislados en macOS

Esta guía detalla cómo configurar un flujo de trabajo basado en **Contenedores de Desarrollo (Dev Containers)** en macOS.

**El objetivo principal:** Tener múltiples entornos (Node.js, Python, Java, Flutter) totalmente independientes. Las librerías, dependencias y configuraciones vivirán _dentro_ del contenedor. Cuando termines un proyecto y lo borres, **tu Mac quedará 100% limpia**, sin rastros de configuraciones globales.

---

## 🛠️ 1. Requisitos Previos (Instalación por única vez)

Para que la magia funcione, necesitas instalar el motor de contenedores y el editor en tu Mac.

1. **Instalar OrbStack** (Motor de contenedores recomendado para Mac, más rápido que Docker Desktop):
   Abre tu terminal y ejecuta:

   ```bash
   brew install orbstack
   ```

   _(Abre la aplicación OrbStack una vez para que inicie sus servicios de fondo)._

2. **Instalar Visual Studio Code:**
   Descárgalo desde [code.visualstudio.com](https://code.visualstudio.com/).

3. **Extensión "Dev Containers":**
   Abre VS Code, ve a Extensiones (`Cmd + Shift + X`), busca **"Dev Containers"** (de Microsoft) e instálala.

---

## 🏗️ 2. ¿Cómo crear y configurar un entorno virtual?

El flujo de trabajo es idéntico para cualquier lenguaje. Cada proyecto tendrá su propio entorno independiente definido por una carpeta llamada `.devcontainer`.

### Paso a Paso General:

1. Crea una carpeta para tu proyecto (ej: `mkdir mi-proyecto-node` y entra en ella `cd mi-proyecto-node`).
2. Abre la carpeta en VS Code (`code .`).
3. En la raíz de tu proyecto, crea una carpeta llamada `.devcontainer`.
4. Dentro, crea un archivo llamado `devcontainer.json`.
5. Pega la configuración del lenguaje que necesites (ver sección 3).
6. Presiona `Cmd + Shift + P`, escribe **"Reopen in Container"** (Volver a abrir en contenedor) y presiona Enter.
7. ¡Listo! La primera vez tardará unos minutos en descargar el sistema, pero luego será instantáneo.

---

## 📦 3. Plantillas de Configuraciones por Lenguaje

Copia y pega el código correspondiente en tu archivo `.devcontainer/devcontainer.json` según el lenguaje del proyecto.

Para transformar estos archivos en entornos de desarrollo de nivel profesional (_production-ready_), aplicamos cuatro patrones clave de la especificación de **Dev Containers** (`containers.dev`):

1. **Montajes en volumen (`mounts` tipo `volume`):** Resuelven el principal cuello de botella de macOS (la lentitud de lectura/escritura de archivos a través de la capa de virtualización) y aíslan al 100% las carpetas pesadas (`node_modules`, `.m2`, `.pub-cache`) dentro de OrbStack sin saturar el Finder de macOS.
2. **Ciclo de vida (`postCreateCommand`):** Automatiza la instalación de dependencias, permisos de ejecutables y comprobaciones al inicializar el contenedor.
3. **Mapeo y nombrado de puertos (`forwardPorts` y `portsAttributes`):** Etiqueta puertos como APIs, Vite o Spring Boot para abrirlos con un solo clic.
4. **Estandarización del editor (`customizations.vscode.settings`):** Fuerza reglas de equipo (formateo automático al guardar, tabuladores e indentación).

---

### 1. 🟢 Entorno Node.js (Robusto & Alta Velocidad de I/O)

Este archivo resuelve el aislamiento de `node_modules` en un volumen dedicado de Docker, añade Git/GitHub CLI oficiales, persiste el historial de comandos de la terminal y habilita el formateo automático con Prettier y ESLint.

```json
{
  "name": "Node.js Production Dev",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:24",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "mounts": [
    "source=${localWorkspaceFolderBasename}-node_modules,target=${containerWorkspaceFolder}/node_modules,type=volume",
    "source=${localWorkspaceFolderBasename}-bashhistory,target=/commandhistory,type=volume"
  ],
  "containerEnv": {
    "HISTFILE": "/commandhistory/.bash_history",
    "NODE_ENV": "development"
  },
  "forwardPorts": [3000, 5173, 8080],
  "portsAttributes": {
    "3000": {
      "label": "App/API Server",
      "onAutoForward": "notify"
    },
    "5173": {
      "label": "Vite Dev Server",
      "onAutoForward": "openBrowser"
    }
  },
  "postCreateCommand": "corepack enable && npm install",
  "customizations": {
    "vscode": {
      "settings": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.codeActionsOnSave": {
          "source.fixAll.eslint": "explicit"
        },
        "files.eol": "\n"
      },
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "usernamehw.errorlens",
        "eamodio.gitlens"
      ]
    }
  },
  "remoteUser": "node"
}
```

---

### 2. ☕ Entorno Java 25 (Microservicios & Cache de Maven/Gradle)

En Java, reconstruir un contenedor sin caché obliga a descargar gigabytes del repositorio central. Aquí aislamos la caché de Maven (`.m2`) y Gradle (`.gradle`) en volúmenes nombrados, damos permisos automáticos a los scripts `./mvnw` y configuramos el puerto de depuración remota (JDWP).

```json
{
  "name": "Java 25 Enterprise Dev",
  "image": "mcr.microsoft.com/devcontainers/java:25",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "mounts": [
    "source=${localWorkspaceFolderBasename}-m2,target=/home/vscode/.m2,type=volume",
    "source=${localWorkspaceFolderBasename}-gradle,target=/home/vscode/.gradle,type=volume",
    "source=${localWorkspaceFolderBasename}-bashhistory,target=/commandhistory,type=volume"
  ],
  "containerEnv": {
    "HISTFILE": "/commandhistory/.bash_history",
    "JAVA_TOOL_OPTIONS": "-Dfile.encoding=UTF-8"
  },
  "forwardPorts": [8080, 5005],
  "portsAttributes": {
    "8080": {
      "label": "Spring Boot / Microservice API",
      "onAutoForward": "notify"
    },
    "5005": {
      "label": "JVM Remote Debugger",
      "onAutoForward": "silent"
    }
  },
  "postCreateCommand": "chmod +x ./mvnw ./gradlew 2>/dev/null || true",
  "customizations": {
    "vscode": {
      "settings": {
        "java.configuration.updateBuildConfiguration": "automatic",
        "java.compile.nullAnalysis.mode": "automatic",
        "editor.formatOnSave": true,
        "files.eol": "\n"
      },
      "extensions": [
        "vscjava.vscode-java-pack",
        "vscjava.vscode-spring-boot-dashboard",
        "vscjava.vscode-spring-initializr",
        "redhat.vscode-xml",
        "usernamehw.errorlens"
      ]
    }
  },
  "remoteUser": "vscode"
}
```

---

### 3. 📱 Entorno Flutter + Node.js (Corrección de Usuario & Cache de Pub)

Este archivo soluciona de raíz el error `unable to find user vscode` estableciendo `"remoteUser": "root"`, monta en volumen la caché de Flutter (`.pub-cache`) y la de `node_modules`, activa Flutter Web y ejecuta `flutter doctor` automáticamente al construir.

```json
{
  "name": "Flutter & Node.js Production Dev",
  "image": "ghcr.io/cirruslabs/flutter:latest",

  "features": {
    "ghcr.io/devcontainers/features/node:1": {
      "version": "lts"
    },
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "mounts": [
    "source=${localWorkspaceFolderBasename}-pub-cache,target=/root/.pub-cache,type=volume",
    "source=${localWorkspaceFolderBasename}-node_modules,target=${containerWorkspaceFolder}/node_modules,type=volume",
    "source=${localWorkspaceFolderBasename}-bashhistory,target=/commandhistory,type=volume"
  ],
  "containerEnv": {
    "HISTFILE": "/commandhistory/.bash_history",
    "PUB_CACHE": "/root/.pub-cache"
  },
  "forwardPorts": [3000, 8080, 55555],
  "portsAttributes": {
    "3000": {
      "label": "Node.js Backend / API",
      "onAutoForward": "notify"
    },
    "55555": {
      "label": "Flutter Web App",
      "onAutoForward": "openBrowser"
    }
  },
  "postCreateCommand": "apt update && apt install -y adb && rm -f /opt/android-sdk-linux/platform-tools/adb && ln -s /usr/bin/adb /opt/android-sdk-linux/platform-tools/adb && flutter config --enable-web",
  "customizations": {
    "vscode": {
      "settings": {
        "dart.flutterSdkPath": "/sdks/flutter",
        "editor.formatOnSave": true,
        "[dart]": {
          "editor.formatOnSave": true,
          "editor.formatOnType": true,
          "editor.selectionHighlight": false,
          "editor.suggest.snippetsPreventQuickSuggestions": false
        },
        "files.eol": "\n"
      },
      "extensions": [
        "dart-code.flutter",
        "dart-code.dart-code",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "usernamehw.errorlens"
      ]
    }
  },
  "remoteUser": "root"
}
```

flutter run -d web-server --web-port=55555 --web-hostname=0.0.0.0

---

## ✅ 4. Validación del Entorno y Uso de GitHub

Una vez que VS Code termine de abrirse dentro del contenedor, es buena práctica comprobar que todo está funcionando correctamente en el sistema aislado.

### 4.1. Abrir la terminal integrada

Presiona `` Control + ` `` (o ve a _Terminal > New Terminal_ en el menú superior).
Deberías ver que el prompt de tu consola cambia a algo como `node ➜ /workspaces/mi-proyecto $`, indicando que estás dentro de Linux.

### 4.2. Validar versiones de lenguajes instalados

Verifica que las herramientas estén corriendo desde el contenedor:

```bash
# Validar Node.js
node -v
npm -v

# Validar Git
git --version
```

### 4.3. Configurar tu identidad en Git

A diferencia de tu Mac, este entorno es nuevo, así que debes indicarle a Git quién eres para hacer _commits_:

```bash
git config --global user.name "Jose Mejia"
git config --global user.email "josmejia.2401@gmail.com"
```

### 4.4. Autenticación con GitHub (GitHub CLI)

Las imágenes de Dev Containers normalmente incluyen la herramienta `gh`. Para autenticarte de forma segura y poder clonar repos privados o hacer _push_:

```bash
gh auth login
```

1. Elige **GitHub.com** y **HTTPS**.
2. Selecciona **Login with a web browser**.
3. Se generará un código. VS Code te pedirá abrir el navegador de tu Mac para que lo pegues y autorices el acceso.

_Nota: Si ya iniciaste sesión en VS Code (con el icono de la cuenta en la esquina inferior izquierda de la interfaz gráfica), a menudo ni siquiera es necesario este paso, ya que el editor pasa las credenciales de tu Mac al contenedor de forma automática._

---

## 🧹 5. Cómo eliminar el entorno y dejar la Mac limpia

La mayor ventaja de este sistema es que deshacerte de un proyecto elimina todas sus dependencias de sistema. Si te cansas del proyecto y quieres borrarlo sin dejar rastros:

1. **Cierra el entorno:** En VS Code, haz clic en el botón azul de la esquina inferior izquierda y selecciona "Cerrar conexión remota" (o simplemente cierra la ventana del editor).
2. **Borra la carpeta de tu código:** Ve al Finder en tu Mac y envía la carpeta de tu proyecto a la papelera.
3. **Limpia el motor de contenedores:** Abre la terminal de tu Mac y ejecuta el siguiente comando para limpiar contenedores detenidos, redes y cache que ya no se usan:
   ```bash
   docker system prune -a --volumes -f
   ```
   _(Alternativamente, puedes abrir la aplicación OrbStack, ir a la pestaña "Containers" e "Images" y borrarlos visualmente con un clic)._

**Resultado:** Tu Mac vuelve exactamente al mismo estado en el que estaba antes de empezar el proyecto. Ninguna variable de entorno tocada, ningún paquete huérfano en disco.
