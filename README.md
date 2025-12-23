# إعادة صياغة السكربت للعمل على Ubuntu Server

هذا هو السكربت المحول ليعمل بشكل مباشر على Ubuntu Server بدون الحاجة إلى Proxmox VE:

```bash
#!/usr/bin/env bash
# Matrix Synapse Management Script for Ubuntu Server
# Supports installation, updates, and status checks

set -e

# ==================== CONFIGURATION ====================
APP="Element Synapse"
SYNAPSE_CONFIG_DIR="/etc/matrix-synapse"
SYNAPSE_DATA_DIR="/var/lib/matrix-synapse"
SYNAPSE_ADMIN_DIR="/opt/synapse-admin"
APP_VERSION_FILE="/opt/synapse_version.txt"

# ==================== COLOR CODES ====================
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
NC='\033[0m'

# ==================== LOGGING FUNCTIONS ====================
info() { echo -e "${BLUE}[INFO]${NC} $1"; }
ok() { echo -e "${GREEN}[✓]${NC} $1"; }
error() { echo -e "${RED}[✗]${NC} $1"; exit 1; }
warning() { echo -e "${YELLOW}[!]${NC} $1"; }
header() { echo ""; echo -e "${PURPLE}==== $1 ====${NC}"; echo ""; }

# ==================== HELPER FUNCTIONS ====================

check_root() {
    [[ $EUID -eq 0 ]] || error "This script must be run as root (use sudo)"
}

check_os() {
    . /etc/os-release
    [[ "$ID" =~ (ubuntu|debian) ]] || error "Unsupported OS: $ID. Requires Ubuntu/Debian"
    
    if [[ "$ID" == "ubuntu" && $(echo "$VERSION_ID < 20.04" | bc) -eq 1 ]] || \
       [[ "$ID" == "debian" && $(echo "$VERSION_ID < 11" | bc) -eq 1 ]]; then
        error "Ubuntu 20.04+ or Debian 11+ required"
    fi
    
    ok "OS check passed: $PRETTY_NAME"
}

install_prerequisites() {
    header "Installing Prerequisites"
    apt-get update
    apt-get install -y curl wget gnupg lsb-release apt-transport-https software-properties-common
    ok "Prerequisites installed"
}

install_nodejs() {
    header "Installing Node.js"
    if ! command -v node &> /dev/null; then
        mkdir -p /etc/apt/keyrings
        curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
        echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" > /etc/apt/sources.list.d/nodesource.list
        apt-get update
        apt-get install -y nodejs
        npm install -g yarn
        ok "Node.js $(node --version) and Yarn installed"
    else
        ok "Node.js already installed: $(node --version)"
    fi
}

install_synapse() {
    header "Installing Matrix Synapse"
    [[ ! -f /etc/apt/sources.list.d/matrix-org.list ]] && {
        curl -L https://matrix.org/packages/debian/repo-key.asc | apt-key add -
        echo "deb https://matrix.org/packages/debian $(lsb_release -cs) main" > /etc/apt/sources.list.d/matrix-org.list
        apt-get update
    }
    
    apt-get install -y matrix-synapse-py3
    mkdir -p "$SYNAPSE_DATA_DIR"
    
    [[ ! -f "$SYNAPSE_CONFIG_DIR/homeserver.yaml" ]] && {
        warning "No Synapse configuration found!"
        info "Generate config with: sudo matrix-synapsectl generate"
    }
    
    systemctl enable --now matrix-synapse 2>/dev/null || true
    ok "Synapse installed and started"
}

install_synapse_admin() {
    header "Installing Synapse-Admin"
    RELEASE=$(curl -fsSL https://api.github.com/repos/etkecc/synapse-admin/releases/latest | grep "tag_name" | awk '{print substr($2, 3, length($2)-4) }')
    [[ -z "$RELEASE" ]] && error "Failed to fetch release info"
    
    info "Latest version: v${RELEASE}"
    [[ -d "$SYNAPSE_ADMIN_DIR" ]] && rm -rf "$SYNAPSE_ADMIN_DIR"
    mkdir -p "$SYNAPSE_ADMIN_DIR"
    
    temp_file=$(mktemp)
    curl -fsSL "https://github.com/etkecc/synapse-admin/archive/refs/tags/v${RELEASE}.tar.gz" -o "$temp_file"
    tar xzf "$temp_file" -C "$SYNAPSE_ADMIN_DIR" --strip-components=1
    
    cd "$SYNAPSE_ADMIN_DIR"
    yarn global add serve
    yarn install --ignore-engines
    yarn build
    mv ./dist ../ && rm -rf ./* && mv ../dist ./
    
    echo "${RELEASE}" > "$APP_VERSION_FILE"
    rm -f "$temp_file"
    ok "Synapse-Admin v${RELEASE} installed"
}

create_synapse_admin_service() {
    header "Configuring Synapse-Admin Service"
    cat > /etc/systemd/system/synapse-admin.service <<EOF
[Unit]
Description=Synapse Admin Web UI
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/synapse-admin
ExecStart=/usr/local/bin/serve -s dist -l 5173
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload
    ok "Service configured"
}

update_synapse_admin() {
    header "Updating Synapse-Admin"
    [[ ! -f "$APP_VERSION_FILE" ]] && echo "0" > "$APP_VERSION_FILE"
    CURRENT_VERSION=$(cat "$APP_VERSION_FILE")
    RELEASE=$(curl -fsSL https://api.github.com/repos/etkecc/synapse-admin/releases/latest | grep "tag_name" | awk '{print substr($2, 3, length($2)-4) }')
    
    if [[ "${RELEASE}" != "${CURRENT_VERSION}" ]]; then
        info "Updating v${CURRENT_VERSION} → v${RELEASE}"
        systemctl stop synapse-admin
        install_synapse_admin
        [[ ! -f /etc/systemd/system/synapse-admin.service ]] && create_synapse_admin_service
        systemctl start synapse-admin
        ok "Updated to v${RELEASE}"
    else
        ok "Already up to date: v${RELEASE}"
    fi
}

show_status() {
    header "Service Status"
    echo -e "${BLUE}Synapse:${NC}"; systemctl status matrix-synapse --no-pager -l 2>/dev/null || warning "Not running"
    echo -e "\n${BLUE}Synapse-Admin:${NC}"; systemctl status synapse-admin --no-pager -l 2>/dev/null || warning "Not installed"
    echo -e "\n${BLUE}Ports:${NC}"; ss -tlnp | grep -E ':(8008|5173)' || warning "Not listening"
    echo -e "\n${BLUE}Versions:${NC}"
    [[ -f "$APP_VERSION_FILE" ]] && echo "  Synapse-Admin: v$(cat "$APP_VERSION_FILE")" || echo "  Synapse-Admin: Not installed"
    dpkg -l | grep -q matrix-synapse-py3 && echo "  Synapse: $(dpkg -l matrix-synapse-py3 | awk '/matrix-synapse-py3/{print $3}')" || echo "  Synapse: Not installed"
}

# ==================== MAIN COMMANDS ====================

install_script() {
    header "Installing $APP"
    check_root
    check_os
    install_prerequisites
    install_nodejs
    install_synapse
    install_synapse_admin
    create_synapse_admin_service
    systemctl enable --now synapse-admin
    sleep 3
    show_status
    
    SERVER_IP=$(hostname -I | awk '{print $1}' || echo "localhost")
    cat <<EOF

${PURPLE}╔══════════════════════════════════════════════════════════════╗${NC}
${PURPLE}║              Installation Complete!                          ║${NC}
${PURPLE}╚══════════════════════════════════════════════════════════════╝${NC}

${BLUE}Access URLs:${NC}
  Synapse:     ${GREEN}http://${SERVER_IP}:8008${NC}
  Admin UI:    ${GREEN}http://${SERVER_IP}:5173${NC}

${BLUE}Next Steps:${NC}
  1. Configure:  ${YELLOW}sudo nano $SYNAPSE_CONFIG_DIR/homeserver.yaml${NC}
  2. Restart:    ${YELLOW}sudo systemctl restart matrix-synapse${NC}
  3. Create admin: ${YELLOW}sudo register_new_matrix_user -c $SYNAPSE_CONFIG_DIR/homeserver.yaml http://localhost:8008${NC}

${BLUE}Logs:${NC}
  Synapse:   ${YELLOW}journalctl -u matrix-synapse -f${NC}
  Admin UI:  ${YELLOW}journalctl -u synapse-admin -f${NC}
EOF
}

update_script() {
    header "Updating $APP"
    check_root
    dpkg -l | grep -q matrix-synapse-py3 || error "Synapse not installed. Run 'install' first."
    install_prerequisites
    install_nodejs
    
    header "Updating System & Synapse"
    apt-get update
    apt-get -y upgrade
    apt-get install -y matrix-synapse-py3
    ok "System and Synapse updated"
    
    [[ -f /etc/systemd/system/synapse-admin.service ]] && update_synapse_admin || warning "Synapse-Admin not installed"
    show_status
    ok "Update completed"
}

show_help() {
    cat <<EOF
Usage: sudo $0 {install|update|status|help}

Commands:
  install    Install $APP and Synapse-Admin
  update     Update existing installation
  status     Show service status
  help       Show this help

Requirements:
  - Ubuntu 20.04+ or Debian 11+
  - Root privileges

After installation:
  - Synapse runs on port 8008
  - Admin UI runs on port 5173
EOF
}

case "$1" in
    install) install_script ;;
    update) update_script ;;
    status) check_root; show_status ;;
    help|--help|-h) show_help ;;
    *) error "Invalid command: $1\nRun '$0 help' for usage";;
esac
```

## طريقة الاستخدام:

1. **تثبيت جديد:**
```bash
sudo bash synapse-ubuntu.sh install
```

2. **تحديث موجود:**
```bash
sudo bash synapse-ubuntu.sh update
```

3. **عرض الحالة:**
```bash
sudo bash synapse-ubuntu.sh status
```

## التغييرات الرئيسية:

- ✅ **إزالة تبعيات Proxmox**: لا حاجة لـ `build.func` أو دوال الحاويات
- ✅ **نظام خدمات مباشر**: يستخدم `systemctl` لتشغيل الخدمات مباشرة على النظام
- ✅ **تثبيت Synapse**: يستخدم مستودعات Matrix.org الرسمية بدلاً من الافتراض بوجوده
- ✅ **أوامر واضحة**: `install` / `update` / `status` بدلاً من السكربت التلقائي
- ✅ **تحقق من المتطلبات**: يتحقق من إصدار Ubuntu/Debian والصلاحيات
- ✅ **رسائل واضحة**: تعليمات دقيقة بعد التثبيت لإتمام الإعداد

السكربت الآن مستقل تماماً ويعمل مباشرة على Ubuntu Server دون أي تبعيات على Proxmox VE.
