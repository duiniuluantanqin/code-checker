# 自动部署流水线使用说明

本项目使用GitHub Actions实现自动化部署。当推送代码到`main`分支时，将自动触发部署流程。

## 前提条件

1. 拥有一台可以通过SSH访问的服务器
2. 服务器上已安装Node.js、npm和PM2
3. 服务器上已创建好应用程序目录

## 部署架构

本项目采用以下部署架构：

1. GitHub Actions负责构建应用并将构建产物传输到服务器
2. 使用PM2管理Node.js应用程序的运行
3. 通过环境变量配置应用程序的运行参数

## 设置GitHub Secrets

在GitHub仓库中，需要设置以下Secrets用于部署：

1. 打开你的GitHub仓库
2. 点击 "Settings" -> "Secrets and variables" -> "Actions"
3. 点击 "New repository secret" 添加以下secrets:

### 服务器连接信息
- `SERVER_HOST`: 服务器IP地址或域名
- `SERVER_USERNAME`: SSH登录用户名
- `SERVER_SSH_KEY`: SSH私钥（不是密码）
- `SERVER_PORT`: SSH端口（通常为22）

### 环境变量（必需）
- `DEEPSEEK_API_KEY`: Deepseek API密钥
- `DEEPSEEK_API_URL`: Deepseek API URL（默认为https://api.deepseek.com/v1）
- `SILICONFLOW_API_KEY`: Siliconflow API密钥
- `SILICONFLOW_API_URL`: Siliconflow API URL（默认为https://api.siliconflow.cn/v1）
- `MAX_FILE_SIZE_KB`: 最大文件大小，单位KB（默认为100）

## 如何获取SSH私钥

如果你还没有SSH密钥对，可以按照以下步骤生成：

1. 在本地终端运行：`ssh-keygen -t rsa -b 4096`
2. 将生成的公钥（通常在`~/.ssh/id_rsa.pub`）添加到服务器的`~/.ssh/authorized_keys`文件中
3. 将私钥（通常在`~/.ssh/id_rsa`）的内容复制到GitHub的`SERVER_SSH_KEY` secret中

## 修改部署配置

### 1. 修改部署目标路径

在`.github/workflows/deploy.yml`文件中，修改以下内容：

```yaml
# 修改为你的应用在服务器上的实际路径
target: "/home/admin/code-checker"

# 和

script: |
  cd /home/admin/code-checker
  # ...其余部分保持不变
```

### 2. 修改环境变量（如需要）

如果需要添加或修改环境变量，请在`.github/workflows/deploy.yml`文件的"Create .env file"步骤中进行修改：

```yaml
- name: Create .env file
  run: |
    cat > .env << EOL
    DEEPSEEK_API_KEY=${{ secrets.DEEPSEEK_API_KEY }}
    DEEPSEEK_API_URL=${{ secrets.DEEPSEEK_API_URL }}
    SILICONFLOW_API_KEY=${{ secrets.SILICONFLOW_API_KEY }}
    SILICONFLOW_API_URL=${{ secrets.SILICONFLOW_API_URL }}
    MAX_FILE_SIZE_KB=${{ secrets.MAX_FILE_SIZE_KB }}
    # 可以添加其他环境变量
    EOL
```

## 触发部署

部署会在每次推送到`main`分支时自动触发。只需执行：

```bash
git add .
git commit -m "你的提交信息"
git push origin main
```

## 部署流程

1. GitHub Actions检出代码
2. 设置Node.js环境（使用Node.js 22）
3. 安装依赖（`npm ci`）
4. 构建应用（`npm run build`）
5. 创建.env文件（使用GitHub Secrets中的环境变量）
6. 将构建产物传输到服务器（`.output/`目录、`package.json`、`ecosystem.config.js`和`.env`文件）
7. 在服务器上安装生产环境依赖
8. 使用PM2重启或启动应用

## 手动部署

如果需要手动部署，可以按照以下步骤操作：

1. 在本地构建应用：
```bash
npm ci
npm run build
```

2. 创建.env文件：
```bash
cat > .env << EOL
DEEPSEEK_API_KEY=your_deepseek_api_key
DEEPSEEK_API_URL=https://api.deepseek.com/v1
SILICONFLOW_API_KEY=your_siliconflow_api_key
SILICONFLOW_API_URL=https://api.siliconflow.cn/v1
MAX_FILE_SIZE_KB=100
EOL
```

3. 将构建产物传输到服务器：
```bash
scp -r .output/ package.json ecosystem.config.js .env user@your-server:/home/admin/code-checker/
```

4. 在服务器上重启应用：
```bash
ssh user@your-server
cd /home/admin/code-checker
npm ci --production
pm2 list
pm2 delete nuxt-app || true
pm2 start .output/server/index.mjs --name nuxt-app -i max
```

## 故障排除

如果部署失败，可以在GitHub仓库的Actions标签页查看详细的日志信息。常见问题包括：

1. **SSH连接失败**：检查SERVER_HOST、SERVER_USERNAME、SERVER_SSH_KEY和SERVER_PORT是否正确
2. **文件传输失败**：确保服务器上的目标目录存在且有写入权限
3. **Node.js或npm错误**：检查服务器上的Node.js版本是否兼容
4. **PM2错误**：确保PM2已正确安装并可用
5. **环境变量问题**：确保所有必需的环境变量都已在GitHub Secrets中设置

## PM2常用命令

```bash
# 查看应用状态
pm2 status

# 查看应用日志
pm2 logs nuxt-app

# 重启应用
pm2 restart nuxt-app

# 停止应用
pm2 stop nuxt-app

# 删除应用
pm2 delete nuxt-app
``` 