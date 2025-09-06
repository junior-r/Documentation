## 🛠️Paso 1: Preparar el VPS

### 🔁 1. Actualizar paquetes de Linux

```bash
sudo apt update && sudo apt upgrade -y
```

### 📦2. Instalar dependencias necesarias

```bash
sudo apt install curl git unzip build-essential -y
```

### 📦3. Instalar fnm (Fast Node Manager) para manejar versiones de Node

```bash
curl -fsSL https://fnm.vercel.app/install | bash
```

Luego agregarlo al PATH y cargarlo

```bash
export PATH="$HOME/.fnm:$PATH"
eval "$(fnm env)"
```

```bash
fnm list
```

### 📦4. Instalar pnpm (opcional)

```bash
npm install -g pnpm
```

```bash
pnpm -v
```

### 📦5. Instalar pm2

```bash
npm install -g pm2
```

```bash
pm2 -v
```

## ⚙️6. Configurar PM2 para reiniciarse al boot

Esto asegura que tu backend se levante después de un reinicio del servidor:

```bash
pm2 startup systemd
```

El comando mostrará una línea (ejemplo: sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u root --hp /root) → cópiala y ejecútala.

Luego guarda los procesos activos:

```bash
pm2 save
```

---

## 📏Paso 2: Preparar la estructura de carpetas del proyecto

```bash
/root/api
     /current      # symlink a la release activa
     /releases     # carpetas por timestamp de cada deploy
     /shared       # archivos persistentes (.env, logs, uploads)
```

> Nota: **NO** hacer un `mkdir current` se crea como symlink.

```bash
mkdir -p /root/api/releases
mkdir -p /root/api/shared
nano /root/api/shared/.env
mkdir -p /root/api/shared/logs
```

---

## 🛠️Paso 3: Configurar el repositorio Git y Node

1. Asegurarte de que el repo de Git está accesible con SSH.

   - Crear llave SSH en el VPS (`ssh-keygen -t ed25519`) si no existe.
   - Agregar la llave pública a GitHub.
   - Probar conexión: `ssh -T git@github.com`.

2. Colocar un `.node-version` en el repo para indicar la versión de Node.

---

## 🚀Paso 4: Crear `deploy.sh`

Archivo en la raíz del proyecto (`/root/api/deploy.sh`):

```bash
#!/bin/bash
set -e

APP_NAME="api"
APP_DIR="/root/$APP_NAME"
RELEASES_DIR="$APP_DIR/releases"
SHARED_DIR="$APP_DIR/shared"
TIMESTAMP=$(date +"%Y%m%d%H%M%S")
NEW_RELEASE="$RELEASES_DIR/$TIMESTAMP"
REPO="git@github.com:tuusuario/tu-repo.git"
BRANCH="main"

echo "🚀 Iniciando despliegue en $TIMESTAMP"

# Rollback automático si algo falla
trap 'echo "❌ Error en el despliegue, limpiando release fallido..."; rm -rf $NEW_RELEASE' ERR

# Crear release
mkdir -p $NEW_RELEASE

# Clonar el repo en la nueva carpeta
git clone -b $BRANCH --depth=1 $REPO $NEW_RELEASE

cd $NEW_RELEASE

# Activar la versión de Node del proyecto
export PATH="$HOME/.fnm:$PATH"
eval "$(fnm env)"

# Instalar dependencias
pnpm install

# Generar Prisma client
npx prisma generate

# Compilar TypeScript
pnpm run build

# Enlazar archivos persistentes (ej: .env)
ln -sfn $SHARED_DIR/.env $NEW_RELEASE/.env
mkdir -p $NEW_RELEASE/logs
ln -sfn $SHARED_DIR/logs $NEW_RELEASE/logs

# Actualizar symlink
ln -sfn $NEW_RELEASE $APP_DIR/current

echo "💡 Release activa: $(readlink -f $APP_DIR/current)"

# Reload con pm2 (zero-downtime)
pm2 startOrReload ecosystem.config.js --env production

# Mantener solo las últimas 5 releases
cd $RELEASES_DIR
ls -1tr | head -n -5 | xargs -r rm -rf --

echo "✅ Despliegue completado en $TIMESTAMP"
```

---

## Paso 5: Desplegar la primera vez

```bash
chmod +x deploy.sh
./deploy.sh
```

- Verificar logs con:

```bash
pm2 logs api --lines 100
```

- Verificar release activa:

```bash
readlink -f /root/api/current
```

---

## Paso 6: Futuras actualizaciones

Cada vez que quieras desplegar una nueva versión:

```bash
./deploy.sh
```

- PM2 hará reload automáticamente.
- `current` apuntará a la nueva release.
- Si algo falla, la release se elimina automáticamente y no afecta la producción.

---

## Extra

### ⚙️Configurar `ecosystem.config.js`

PM2 utiliza para administrar tus aplicaciones Node.js. Sirve para definir cómo arrancar tu app, qué entorno usar, cómo hacer reload con cero downtime, variables de entorno, cantidad de instancias, logging, entre otros.

```bash
module.exports = {
    apps: [
        {
            name: "api",
            script: "/root/api/current/build/index.js",
            instances: "max",
            exec_mode: "cluster",
            env_production: {
                NODE_ENV: "production",
            },
            output: "logs/out.log",
            error: "logs/error.log"
        }
    ]
}
```

---

## 📖 Diccionario de términos claves

- **PM2**: Administrador de procesos para Node.js. Permite ejecutar aplicaciones en segundo plano, reiniciarlas automáticamente si fallan y configurarlas para que inicien al arrancar el servidor.
- **Bash**: Intérprete de comandos más común en sistemas Linux/Unix. Sirve para ejecutar instrucciones y escribir scripts automatizados (`.sh`).
- **Symlink** (Enlace simbólico): Archivo especial que actúa como un "atajo" o puntero a otra carpeta o archivo. En los despliegues se usa para que la carpeta `current` siempre apunte a la última versión liberada sin mover archivos manualmente.
- **Release**: Una copia del proyecto dentro de una carpeta con marca de tiempo. Sirve para tener versiones anteriores listas en caso de rollback.
- **Shared**: Carpeta que guarda archivos persistentes (ejemplo: `.env`, `logs`, `uploads`) y que se enlazan a cada release para que no se pierdan con cada deploy.
- **Deploy Script** (`deploy.sh`): Script en Bash que automatiza el proceso de descargar la última versión del proyecto, instalar dependencias, compilar y reiniciar el servicio con PM2.
- **fnm** (Fast Node Manager): Herramienta para instalar y manejar múltiples versiones de Node.js de manera sencilla.
- **pnpm**: Gestor de paquetes alternativo a npm/yarn, más rápido y con mejor manejo de dependencias.
- **Systemd**: Sistema de inicio de Linux que gestiona servicios y procesos, utilizado junto a PM2 para que las apps reinicien automáticamente tras un reinicio del VPS.

---

<div align="center"> 
  <a href="https://github.com/junior-r"> 
    <img src="https://avatars.githubusercontent.com/junior-r" loading="lazy" width="100" style="border-radius: 50%;" alt="Junior R's GitHub Profile"> 
  </a> <br /> <strong>👨‍💻 Junior Ruiz</strong> 
  <br /> 
  <a href="https://github.com/junior-r" target="_blank">GitHub</a> • 
  <a href="https://junior-dev.vercel.app/" target="_blank">Sitio web</a> • 
  <a href="mailto:juniorruiz331@gmail.com">Contacto</a> 
</div>
