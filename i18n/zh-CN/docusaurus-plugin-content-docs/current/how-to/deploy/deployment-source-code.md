---
sidebar_position: 3
---

# 源码编译部署

本文档介绍如何通过源码编译方式部署各模块。假定按照如下链接方式部署：

![deploy](/featureprobe_deploy_from_code.png)

需要编译部署的模块主要有三个：

| 示例机器   | 部署模块            | 端口          |
| ---------- | ------------------- | ------------- |
| 10.100.1.1 | FeatureProbe API    | 4008          |
| 10.100.1.1 | FeatureProbe UI     | 4009（Nginx） |
| 10.100.1.2 | FeatureProbe Server | 4007          |

## 创建数据库

1. 环境准备

   - MySQL 5.7+

2. 创建 `feature_probe ` 数据库：

   ```sql
   CREATE DATABASE feature_probe; 
   ```

:::note
无须手工创建表结构。初次启动 API 服务时会自动创建所有表和初始化数据。
:::



## 编译部署 API 服务

### 编译步骤 

1. 编译环境

   - JDK 1.8+
   - Maven 2.0+ : [how to install](https://maven.apache.org/install.html)

    

2. 获取源码并编译出部署包：

```bash
git clone https://gitee.com/FeatureProbe/feature-probe-api.git
cd feature-probe-api
mvn clean package
```

  完成编译后会在当前目录生成以版本命名的 jar 部署文件，如 ` target/feature-probe-api-1.1.0.jar`。

### 部署步骤

1. 部署环境

   - JDK 1.8+

2. 将 `feature-probe-api-1.1.0.jar` 放置部署服务器中，填入数据库链接配置，并以 `4008` 端口启动：

   ```bash
    java -jar feature-probe-api-1.1.0.jar --server.port=4008 \
         -Dspring.datasource.jdbc-url=jdbc:mysql://{MYSQL_DATABASE_IP}:{MYSQL_PORT}/feature_probe \  # 数据库 IP/端口和库名
         -Dspring.datasource.username={MYSQL_USERNAME} \
         -Dspring.datasource.password={MYSQL_PASSWORD} 
   ```
   
   :::info
   API 服务更详细的启动参数说明见 [FeatureProbe API 参数说明文档](../../reference/deployment-configuration#featureprobe-api)
   :::

3. 检查服务是否运行成功

   在部署机运行：
   ```bash
   curl "http://localhost:4008/api/actuator/health"
   ```

   显示如下信息则表示启动成功：

   ```json
   {
   	status: "UP"
   }
   ```

## 编译部署 Server 服务

### 编译步骤

1. 环境准备

   * Rust 
     * [官网安装](https://www.rust-lang.org/tools/install)
     * [国内镜像](https://rsproxy.cn/)


2. 获取源码并编译出部署包：

   :::info
   国内建议切换为 cargo 中国镜像：[配置 Cargo 国内镜像源](https://rsproxy.cn/)
   :::

   ```bash
   git clone https://gitee.com/FeatureProbe/feature-probe-server.git
   cd feature-probe-server
   cargo build --release --verbose
   ```

   完成编译后会在 `target/release/` 目录下生成可执行的二制文件：

   ```bash
   $ ls target/release/ 
   feature_probe_server
   ```


### 部署步骤

1. 环境准备

   - 无

2. 将生成的 `feature_probe_server` 文件放在服务器上，并创建启动脚本 `start-feature-probe-server.sh`：

   ```bash
   #/bin/bash
   
   export FP_SERVER_PORT=4007  # FeatureProbe Server 端口
   export FP_TOGGLES_URL=http://10.100.1.1:4008/api/server/toggles  # FeatureProbe API IP 和端口号
   export FP_EVENTS_URL=http://10.100.1.1:4008/api/server/events
   export FP_KEYS_URL=http://10.100.1.1:4008/api/server/sdk_keys
   export FP_REFRESH_SECONDS=1
   
   ./feature_probe_server 
   ```

:::info
Server 服务更详细启动参数说明详见 [FeatureProbe Server 参数说明文档](../../reference/deployment-configuration#featureprobe-server)
:::

3. 执行启动脚本运行服务：`sh ./start-feature-probe-server.sh`

4. 检查服务是否运行成功，在部署机运行：
   ```bash
   curl "http://localhost:4007/"
   ```

   显示如下信息则表示启动成功：

   ```json
   <h1>Feature Probe Server</h1>
   ```


## 编译部署 UI 服务

### 编译步骤 

1. 环境准备

   * Node.js 16+ : [下载](https://nodejs.org/zh-cn/download/)
   * yarn
     * 安装： `npm install -g yarn`
   * python3
     * [安装](https://realpython.com/installing-python/#)
     
   :::info
   国内建议切换为 npm 中国镜像站：`npm config set registry https://npmmirror.com/mirrors/`
   :::

2. 获取源码并编译出可部署的静态文件：

   ```bash
   git clone https://gitee.com/FeatureProbe/feature-probe-ui.git
   cd feature-probe-ui
   yarn install --frozen-lockfile
   yarn build
   ```
   
   完成编译后会在 `build` 目录下生成可部署的静态文件。如下所示：
   
   ```bash
   $ ls build 
   asset-manifest.json favicon.ico         index.html          static/
   ```

### 部署步骤

1. 环境准备

   - Nginx

2. 将编译生成的 `build` 下的所有文件和文件夹复制到 `/usr/share/nginx/html/` 目录下。

## 配置 Nginx

1. 创建 Nginx 配置：`/etc/nginx/conf.d/feature_probe.conf`

   ~~~bash
   upstream featureProbeAPI {
       server 10.100.1.1:4008; # FeatureProbeAPI IP和端口
   }
   
   
   upstream featureProbeServer {
       server 10.100.1.2:4007; # FeatureProbe Server IP和端口
   }
   
   server {
     listen 4009;  # UI 端口
   
     location / {
       index  index.html index.htm;
       root /usr/share/nginx/html;  # UI 静态文件目录
       try_files $uri /index.html;
     }
   
      location /api { # 访问 /api 时统一转发到 featureProbeAPI 服务
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-NginX-Proxy true;
       proxy_pass http://featureProbeAPI/api;
       proxy_ssl_session_reuse off;
       proxy_set_header Host $http_host;
       proxy_cache_bypass $http_upgrade;
       proxy_redirect off;
     }
   
     location /server/api { # 访问 /server/api 时统一转发到 featureProbeServer 服务
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-NginX-Proxy true;
       proxy_pass http://featureProbeServer/api;
       proxy_ssl_session_reuse off;
       proxy_set_header Host $http_host;
       proxy_cache_bypass $http_upgrade;
       proxy_redirect off;
     }
   }
   ~~~

2. 执行 `reload nginx` 配置，使上述配置生效：

   ```
   nginx -s reload
   ```

## 验证安装

### 平台使用
在浏览器中访问 `http://10.100.1.1:4009` 并使用如下账号密码登录来验证是否部署成功：
- username: `admin`
- password: `Pass1234`

### Server Side SDK 访问
SDK连接使用的服务器地址为ngnix机器地址，以上例子中为： `http://10.100.1.1:4009/server`
