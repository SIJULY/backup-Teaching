# 🚀 VPS 全自动备份与一键恢复指南

本项目用于实现 VPS 服务器数据的全自动定时备份，以及在新服务器上的一键恢复。

**备份对象：**
* `cloud_manager` (位于 `/opt/cloud_manager`)
* `xui_manager` (位于 `/root/xui_manager`)

---

## 🛠️ 第一阶段：准备工作 (GitHub)

1.  创建一个 **私有 (Private)** 仓库（建议命名为 `vps-data-backup`）。
2.  记下仓库的 **SSH 地址**（例如：`git@github.com:your_name/vps-data-backup.git`）。

---

## 📤 第二阶段：旧 VPS 设置 (自动备份)

此步骤在**需要备份数据的源服务器**上操作。

### 1. 配置 SSH 密钥
为了让脚本自动上传数据，需要配置 Deploy Key。

```bash
# 1. 生成密钥 (一路回车)
ssh-keygen -t ed25519 -C "vps_backup"

# 2. 查看公钥内容
cat /root/.ssh/id_ed25519.pub
```

复制输出的内容。

进入 GitHub 仓库 -> Settings -> Deploy keys -> Add deploy key。

粘贴密钥，并勾选 "Allow write access" (允许写入)，保存。

### 2. 部署备份脚本
创建脚本文件：

```bash
vim /root/sync_to_github.sh
```
👇 复制以下内容 (注意修改 REPO_URL 为你自己的地址)：

```bash
#!/bin/bash

# ================= 配置区域 =================
# ⚠️ 请修改这里：你的 GitHub 仓库 SSH 地址
REPO_URL="git@github.com:YOUR_USERNAME/YOUR_REPO.git"

# 需要备份的项目 "文件夹名:真实路径"
PROJECTS=(
    "cloud_manager:/opt/cloud_manager"
    "xui_manager:/root/xui_manager"
)

BACKUP_ROOT="/root/private_backup"    # 本地中转目录
DATE=$(date "+%Y-%m-%d %H:%M:%S")
# ===========================================

# 检查 rsync
if ! command -v rsync &> /dev/null; then
    if [ -f /etc/debian_version ]; then
        apt-get update -qq && apt-get install -y rsync
    elif [ -f /etc/redhat-release ]; then
        yum install -y rsync
    fi
fi

# 初始化 Git 仓库
if [ ! -d "$BACKUP_ROOT" ]; then
    mkdir -p "$BACKUP_ROOT"
    cd "$BACKUP_ROOT" || exit
    git init
    git branch -m main
    git remote add origin "$REPO_URL"
else
    cd "$BACKUP_ROOT" || exit
    # 确保远程地址正确
    if ! git remote | grep -q origin; then
        git remote add origin "$REPO_URL"
    fi
fi

echo "🔄 [Start] 开始备份..."

# 循环复制文件
for entry in "${PROJECTS[@]}"; do
    NAME="${entry%%:*}"
    SOURCE_PATH="${entry#*:}"

    if [ -d "$SOURCE_PATH" ]; then
        mkdir -p "$BACKUP_ROOT/$NAME"
        # 增量同步，排除不必要文件
        rsync -av --delete \
            --exclude='.git' --exclude='__pycache__' --exclude='*.pyc' \
            --exclude='venv' --exclude='node_modules' \
            "$SOURCE_PATH/" "$BACKUP_ROOT/$NAME/"
        # 删除子目录的 gitignore 防止冲突
        rm -f "$BACKUP_ROOT/$NAME/.gitignore"
    else
        echo "⚠️ 警告：路径不存在 $SOURCE_PATH"
    fi
done

# 提交并强制推送
git add .
if git diff-index --quiet HEAD --; then
    echo "✅ 无文件变化，跳过上传。"
else
    git commit -m "AutoBackup: $DATE"
    # 使用 -f 强制推送，以 VPS 本地数据为准
    if git push -f origin main; then
        echo "✅ 备份成功上传！时间: $DATE"
    else
        echo "❌ 上传失败，请检查 SSH Key 或网络。"
    fi
fi
```
3. 设置定时任务
赋予权限并添加到 Crontab (每天凌晨 4 点运行)：

```bash
chmod +x /root/sync_to_github.sh
echo "0 4 * * * /root/sync_to_github.sh >> /var/log/backup.log 2>&1" | crontab -
```
📥 第三阶段：新 VPS 设置 (一键恢复)
此步骤在新购买或重装后的 VPS 上操作。

1. 配置 SSH 密钥
新机器需要读取权限。

```bash
# 生成密钥
ssh-keygen -t ed25519 -C "new_vps_restore"
# 查看公钥
cat /root/.ssh/id_ed25519.pub
```

复制内容。

进入 GitHub 仓库 -> Settings -> Deploy keys -> Add deploy key。

粘贴密钥 (恢复数据不需要勾选 "Allow write access")。

2. 运行恢复脚本
创建脚本：

Bash

vim /root/restore_from_github.sh
👇 复制以下内容 (注意修改 REPO_URL)：

```bash

#!/bin/bash

# ================= 配置区域 =================
# ⚠️ 请修改这里：你的 GitHub 仓库 SSH 地址
REPO_URL="git@github.com:YOUR_USERNAME/YOUR_REPO.git"

TEMP_DIR="/root/temp_restore_data"
# ===========================================

# 安装工具
if ! command -v git &> /dev/null || ! command -v rsync &> /dev/null; then
    if [ -f /etc/debian_version ]; then
        apt-get update -qq && apt-get install -y git rsync
    elif [ -f /etc/redhat-release ]; then
        yum install -y git rsync
    fi
fi

# 拉取数据
echo "📥 正在从 GitHub 拉取..."
rm -rf "$TEMP_DIR"
git clone "$REPO_URL" "$TEMP_DIR"

if [ ! -d "$TEMP_DIR" ]; then
    echo "❌ 拉取失败，请检查 SSH Key。"
    exit 1
fi

echo "♻️  正在恢复文件..."

# 恢复 Cloud Manager
if [ -d "$TEMP_DIR/cloud_manager" ]; then
    echo "   >> 恢复: /opt/cloud_manager"
    mkdir -p /opt/cloud_manager
    rsync -av "$TEMP_DIR/cloud_manager/" "/opt/cloud_manager/"
fi

# 恢复 XUI Manager
if [ -d "$TEMP_DIR/xui_manager" ]; then
    echo "   >> 恢复: /root/xui_manager"
    mkdir -p /root/xui_manager
    rsync -av "$TEMP_DIR/xui_manager/" "/root/xui_manager/"
fi

# 清理
rm -rf "$TEMP_DIR"
echo "✅ 所有数据已归位！"
```

3. 执行恢复与启动
赋予权限并运行：

```bash

chmod +x /root/restore_from_github.sh
/root/restore_from_github.sh
```

恢复完成后，启动服务：
```bash
# 启动 Cloud Manager
cd /opt/cloud_manager
docker compose up -d --build

# 启动 XUI Manager
cd /root/xui_manager
docker compose up -d --build
```
