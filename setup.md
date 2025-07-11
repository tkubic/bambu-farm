# 🛠️ Bambu Dashboard Setup Guide for Raspberry Pi  
**Fully Automated | Multi-Printer Ready | Single Script Deployment**

---

## 🖥️ 1. Flash Raspberry Pi OS Using Windows

**Tools Required:**
- Raspberry Pi Imager → [Download Here](https://www.raspberrypi.com/software)
- microSD card (16GB+ recommended)
- Raspberry Pi (recommended: 64-bit capable model)

**Steps:**
1. Open Raspberry Pi Imager.
2. Click **“Choose OS”** → Select **“Raspberry Pi OS Lite (64-bit)”**.
3. Click **“Choose Storage”** → Select your SD card.
4. Click the ⚙️ icon (Advanced Settings) and configure:
   - **Hostname:** `bambu-pi`
   - **Enable SSH** → Use password authentication
   - **Username:** `woodward`
   - **Password:** _your choice_
   - **Configure Wi-Fi**: enter SSID and password
5. Save → Write the image.
6. Eject SD card, insert into Raspberry Pi, and power on.

---

## 🔌 2. SSH into the Pi

From your Windows terminal:

```bash
ssh woodward@bambu-pi.local
```

---

## 🧱 3. Install Docker + Docker Compose (No Logout Required)

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```
```bash
sudo sh get-docker.sh
```
```bash
sudo usermod -aG docker woodward
```
```bash
newgrp docker
```
```bash
sudo apt update && sudo apt install docker-compose -y
```

---

## ⚙️ 4. Install and Launch the Dashboard

### 📦 Create a Working Directory

```bash
mkdir -p ~/bambu-dashboard && cd ~/bambu-dashboard
```

### 📜 Create `setup.sh` Installer

```bash
cat <<'EOF' > setup.sh
#!/bin/bash

echo "🚀 Bambu Dashboard Auto-Installer Starting..."

mkdir -p ~/bambu-dashboard && cd ~/bambu-dashboard

# Download runner JAR
echo "📦 Downloading runner..."
curl -s https://api.github.com/repos/TFyre/bambu-farm/releases/latest \
| grep browser_download_url \
| grep runner.jar \
| cut -d '"' -f 4 \
| xargs curl -LO
mv bambu-web-*-runner.jar bambu-web-runner.jar

# Create base dashboard config
echo "🗂️ Creating application.properties..."
cat <<EOL > application.properties
quarkus.http.port=8080
quarkus.http.host=0.0.0.0
quarkus.http.limits.max-body-size=30M
bambu.live-view-url=/_camerastream/
bambu.use-bouncy-castle=true
bambu.users.admin.password=admin
bambu.users.admin.role=admin
bambu.auto-login=true
EOL

# Initialize mediamtx config
echo "📡 Initializing mediamtx.yml..."
cat <<EOL > mediamtx.yml
webrtcLocalTCPAddress: :8189
webrtcIPsFromInterfaces: no
webrtcAdditionalHosts: [bambu-pi.local]

paths:
EOL

# Initialize go2rtc config
echo "📶 Initializing go2rtc.yaml..."
cat <<EOL > go2rtc.yaml
streams:
EOL

# Create Docker Compose file
echo "🐳 Creating docker-compose.yml..."
cat <<EOL > docker-compose.yml
version: "3.8"

services:
  bambuweb:
    container_name: bambuweb
    image: eclipse-temurin:21-jre
    ports:
      - "8080:8080"
    volumes:
      - ./application.properties:/app/application.properties
      - ./bambu-web-runner.jar:/app/bambu-web-runner.jar
    working_dir: /app
    entrypoint:
      - java
      - -Djdk.tls.useExtendedMasterSecret=false
      - -Dquarkus.config.locations=/app/application.properties
      - -jar
      - bambu-web-runner.jar
    depends_on:
      - mediamtx

  mediamtx:
    image: bluenviron/mediamtx
    container_name: mediamtx
    ports:
      - "8189:8189/tcp"
      - "8189:8189/udp"
      - "8888:8888"
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
    restart: unless-stopped

  go2rtc:
    image: alexxit/go2rtc
    container_name: go2rtc
    ports:
      - "1984:1984"
      - "8554:8554"
      - "8555:8555"
    volumes:
      - ./go2rtc.yaml:/config/go2rtc.yaml
    restart: unless-stopped
EOL

# Install printer provisioning tool
echo "🤖 Installing add-x1c.sh..."
cat <<'EOL' > add-x1c.sh
#!/bin/bash

APP_PROPS="application.properties"
MTX_YAML="mediamtx.yml"
GO2RTC_YAML="go2rtc.yaml"

echo "🔧 Add a new Bambu X1C printer profile"
read -p "Printer ID (e.g., x1c01): " printer_id
read -p "Device ID (e.g., 00M09A381800XXX): " device_id
read -p "Access Code (e.g., 2f02xxxx): " access_code
read -p "IP Address (or hostname): " ip

stream_url="http://bambu-pi.local:1984/stream.html?src=$printer_id&mode=mse"

if grep -q "bambu.printers.$printer_id.name" "$APP_PROPS" || \
   grep -q "$printer_id:" "$MTX_YAML" || \
   grep -q "$printer_id:" "$GO2RTC_YAML"; then
  echo "⚠️ Printer ID '$printer_id' already exists. Aborting."
  exit 1
fi

echo "🔐 Getting fingerprint..."
fingerprint=$(openssl s_client -connect "$ip:322" -showcerts </dev/null 2>/dev/null \
| openssl x509 -fingerprint -noout \
| sed 's/^SHA1 Fingerprint=//g' \
| tr -d ':')

[[ -z "$fingerprint" ]] && { echo "❌ Fingerprint fetch failed."; exit 1; }

cat <<EOM >> "$APP_PROPS"

# -- Printer: $printer_id --
bambu.printers.$printer_id.name=$printer_id
bambu.printers.$printer_id.device-id=$device_id
bambu.printers.$printer_id.access-code=$access_code
bambu.printers.$printer_id.ip=$ip
bambu.printers.$printer_id.stream.live-view=true
bambu.printers.$printer_id.ftp.url=ftps://$ip:990
bambu.printers.$printer_id.ftp.port=990
bambu.printers.$printer_id.ftp.log-commands=false
bambu.printers.$printer_id.stream.url=$stream_url
EOM

cat <<EOM >> "$MTX_YAML"

  $printer_id:
    source: rtsps://bblp:$access_code@$ip:322/streaming/live/1
    rtspTransport: tcp
    sourceFingerprint: $fingerprint
EOM

cat <<EOM >> "$GO2RTC_YAML"

  $printer_id:
    - rtsps://bblp:$access_code@$ip:322/streaming/live/1
EOM

echo "♻️ Restarting containers..."
docker-compose restart bambuweb mediamtx go2rtc

echo "🎉 '$printer_id' onboarded!"
EOL

chmod +x add-x1c.sh

# Launch stack
echo "🚀 Starting Bambu Dashboard..."
docker-compose up -d

echo "✅ Setup complete! Visit: http://bambu-pi.local:8080"
EOF
```

### 🚀 Run the Installer

```bash
chmod +x setup.sh
./setup.sh
```

---

## 🖨️ 5. Onboard Printers with Automation

To add any X1C printer:

```bash
~/bambu-dashboard/add-x1c.sh
```

You’ll be prompted for:

- Printer ID (e.g., `x1c01`)
- Device ID (e.g., `00M09A381800XXX`)
- Access Code (e.g., `2f02xxxx`)
- IP Address or hostname (e.g., `192.168.x.x` or `x1c01.local`)

The script will:
- Fetch SSL fingerprint
- Update `application.properties`, `mediamtx.yml`, and `go2rtc.yaml`
- Restart all containers

---

## 📍 6. Access Your Dashboard

Visit:

```
http://bambu-pi.local:8080
```

You're auto-logged in as `admin`. All onboarded printers will be listed.

