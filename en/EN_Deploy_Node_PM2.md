## 🛠️Step 1: Prepare the VPS

### 🔁 1. Update Linux packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 📦2. Install necessary dependencies

```bash
sudo apt install curl git unzip build-essential -y
```

### 📦3. Install fnm (Fast Node Manager) to manage Node versions

```bash
curl -fsSL https://fnm.vercel.app/install | bash
```

Then add it to PATH and load it

```bash
export PATH="$HOME/.fnm:$PATH"
eval "$(fnm env)"
```

```bash
fnm list
```

### 📦4. Install pnpm (optional)

```bash
npm install -g pnpm
```

```bash
pnpm -v
```

### 📦5. Install pm2

```bash
npm install -g pm2
```

```bash
pm2 -v
```

## ⚙️6. Configure PM2 to restart on boot

This ensures your backend starts after a server reboot:

```bash
pm2 startup systemd
```

The command will show a line (example: sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u root --hp /root) → copy it and execute it.

Then save active processes:

```bash
pm2 save
```

---

📏Step 2: Prepare project folder structure

```bash
/root/api
     /current      # symlink to the active release
     /releases     # folders with timestamp for each deploy
     /shared       # persistent files (.env, logs, uploads)
```

> Note: DO NOT make a mkdir current; it is created as a symlink.

```bash
mkdir -p /root/api/releases
mkdir -p /root/api/shared
nano /root/api/shared/.env
mkdir -p /root/api/shared/logs
```

---

## 🛠️Step 3: Configure Git repository and Node

1. Make sure the Git repo is accessible via SSH.

   - Create an SSH key on the VPS (`ssh-keygen -t ed25519`) if it doesn’t exist.
   - Add the public key to GitHub.
   - Test the connection: `ssh -T git@github.com`.

2. Place a `.node-version` file in the repo to indicate the Node version.

---

## 🚀Step 4: Create `deploy.sh`

File in the project root (`/root/api/deploy.sh`):

```bash
#!/bin/bash
set -e

APP_NAME="api"
APP_DIR="/root/$APP_NAME"
RELEASES_DIR="$APP_DIR/releases"
SHARED_DIR="$APP_DIR/shared"
TIMESTAMP=$(date +"%Y%m%d%H%M%S")
NEW_RELEASE="$RELEASES_DIR/$TIMESTAMP"
REPO="git@github.com:yourusername/your-repo.git"
BRANCH="main"

echo "🚀 Starting deployment at $TIMESTAMP"

# Automatic rollback if something fails
trap 'echo "❌ Deployment error, cleaning failed release..."; rm -rf $NEW_RELEASE' ERR

# Create release
mkdir -p $NEW_RELEASE

# Clone repo into new folder
git clone -b $BRANCH --depth=1 $REPO $NEW_RELEASE

cd $NEW_RELEASE

# Activate Node version
export PATH="$HOME/.fnm:$PATH"
eval "$(fnm env)"

# Install dependencies
pnpm install

# Generate Prisma client
npx prisma generate

# Compile TypeScript
pnpm run build

# Link persistent files (e.g., .env)
ln -sfn $SHARED_DIR/.env $NEW_RELEASE/.env
mkdir -p $NEW_RELEASE/logs
ln -sfn $SHARED_DIR/logs $NEW_RELEASE/logs

# Update symlink
ln -sfn $NEW_RELEASE $APP_DIR/current

echo "💡 Active release: $(readlink -f $APP_DIR/current)"

# Reload with pm2 (zero-downtime)
pm2 startOrReload ecosystem.config.js --env production

# Keep only the last 5 releases
cd $RELEASES_DIR
ls -1tr | head -n -5 | xargs -r rm -rf --

echo "✅ Deployment completed at $TIMESTAMP"
```

---

Step 5: Deploy for the first time

```bash
chmod +x deploy.sh
./deploy.sh
```

- Check logs with:

```bash
pm2 logs api --lines 100
```

- Verify active release:

```bash
readlink -f /root/api/current
```

---

## Step 6: Future updates

Every time you want to deploy a new version:

```bash
./deploy.sh
```

- PM2 reloads automatically.
- `current` points to the new release.
- If something fails, the release is automatically removed and production remains unaffected.

---

## Extra

### ⚙️Configure `ecosystem.config.js`

PM2 uses it to manage your Node.js applications. It defines how to start your app, which environment to use, zero-downtime reload, environment variables, number of instances, logging, and more.

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

<div align="center"> 
  <a href="https://github.com/junior-r"> 
    <img src="https://avatars.githubusercontent.com/junior-r" loading="lazy" width="100" style="border-radius: 50%;" alt="Junior R's GitHub Profile"> 
  </a> <br /> <strong>👨‍💻 Junior Ruiz</strong> 
  <br /> 
  <a href="https://github.com/junior-r" target="_blank">GitHub</a> • 
  <a href="https://junior-dev.vercel.app/" target="_blank">Website</a> • 
  <a href="mailto:juniorruiz331@gmail.com">Contact</a> 
</div>
