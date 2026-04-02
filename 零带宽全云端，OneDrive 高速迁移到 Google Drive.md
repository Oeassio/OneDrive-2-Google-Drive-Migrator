# 🚀 零带宽全云端，OneDrive高速迁移到Google Drive

利用 Google Colab 的服务器带宽，将 OneDrive (E5) 上的文件直接对拷到 Google Drive。全程无需消耗本地网络流量，速度极快。

---

### ⚠️ 迁移前必看

* **空间限制**：普通的 Google 个人账号只有 15GB 免费空间，迁移前请务必确认源文件大小没有超出 Google Drive 的剩余容量，否则会中途报错。
* **Colab 运行机制**：Colab 免费实例最长运行 12 小时。如果传输文件过大，建议自行添加 Colab 防掉线脚本。

---

### 第一阶段：获取微软 API 授权

1. **登录管理中心**：使用 E5 管理员账号登录 [Microsoft Entra admin center](https://entra.microsoft.com/)。
2. **注册新应用**：
   * 左侧菜单依次点击 **Entra Id** -> **App registrations (应用注册)** -> **New registration (新注册)**。
   * **名称**：随意填写（如 `Rclone_Transfer`）。
   * **受支持的帐户类型**：选择第三项（任何组织目录中的帐户和个人 Microsoft 帐户）。
   * **重定向 URI**：平台选择 **Web**，网址填：`http://localhost:53682/`。
   * 点击注册。
3. **保存专属 ID 和 密钥**：
   * 注册成功后，复制 **Application (client) ID (应用程序(客户端) ID)**，保存到记事本。
   * 左侧点击 **Certificates & secrets (证书和密码)** -> **New client secret (新客户端密码)**。
   * 创建后，复制 **Value (值)**，保存到记事本。
4. **授予 API 权限**：
   * 左侧点击 **API permissions (API 权限)** -> **Add a permission (添加权限)** -> **Microsoft Graph** -> **Delegated permissions (委托的权限)**。
   * 搜索并勾选以下权限：`Files.Read`, `Files.ReadWrite`, `Files.Read.All`, `Files.ReadWrite.All`, `offline_access`, `User.Read`。
   * 点击添加。
   * 回到权限列表，点击 **Grant admin consent for ... (代表...授予管理员同意)**，状态显示绿勾即可。

---

### 第二阶段：获取 Rclone Config 配置文件

1. [下载本地系统对应的 Rclone](https://rclone.org/downloads/) 并解压。
2. 在解压目录右键在终端打开。输入：
   ```bash
   rclone config
   ```
3. 按照以下步骤交互配置：
   * 输入 `n` 新建，命名为 `onedrive`。
   * 选择 `Microsoft OneDrive` 对应的序号，如“41”。
   * **client_id**：粘贴第一阶段保存的 Application (client) ID。
   * **client_secret**：粘贴第一阶段保存的密钥 Value。
   * **region**：选 `global`（通常输入 `1`）。
   * **Edit advanced config?**：输入 `n`。
   * **Use auto config?**：输入 `y` （此时弹出浏览器，登录 E5 账号完成授权，页面显示 Success）。
   * 选择驱动器类型：输入 `OneDrive for Business` 或 `SharePoint` 对应的序号。选定确认后，输入 `y` 保存配置。
4. **提取配置代码**：
   * 命令行输入 `rclone config file` 找到配置文件路径（通常为 `rclone.conf`）。
   * 用记事本打开该文件，**复制里面的所有文本**备用。

---

### 第三阶段：Google Colab 执行高速云端对拷

打开浏览器，登录你的 Google 账号，进入 [Google Colab](https://colab.research.google.com/)，新建一个笔记本。依次**新建代码块**并运行以下代码。

#### 步骤 1：挂载目标盘 (Google Drive)

```python
from google.colab import drive
drive.mount('/content/drive')
```

*> 运行后按提示授权允许 Colab 访问你的 Google 账号。*

#### 步骤 2：安装 Rclone 环境

```bash
!curl [https://rclone.org/install.sh](https://rclone.org/install.sh) | sudo bash
```

*> 点击运行代码。*

#### 步骤 3：载入 Rclone 配置文件

将下面 `config_content` 三引号之间的内容，**替换为第二阶段最后复制的 `rclone.conf` 中 [onedrive] 项文本**：

```python
import os

os.makedirs('/root/.config/rclone', exist_ok=True)

# 请替换下方三引号内的全部内容
config_content = """
[onedrive]
type = onedrive
client_id = 你的ID
client_secret = 你的密钥
token = {"access_token":"......"}
drive_id = b!......
drive_type = business
"""

with open('/root/.config/rclone/rclone.conf', 'w') as f:
    f.write(config_content)

print("Rclone 配置写入成功！")
```

*> 点击运行代码。*

#### 步骤 4：执行迁移

```bash
!mkdir -p "/content/drive/MyDrive/OneDrive_Backup"

!rclone copy onedrive: "/content/drive/MyDrive/OneDrive_Backup" --transfers=4 --checkers=8 --stats 10s --stats-one-line -v
```

*> 点击运行代码后，对拷就开始了。等代码块运行完毕，Onedrive 文件就完整的迁移到 Google Drive 中 OneDrive_Backup 文件夹里了。*

---

### 提示

如果在传输中途断开了网页，直接重新运行上述代码块即可，Rclone 支持断点续传，会自动跳过已传输的内容。
