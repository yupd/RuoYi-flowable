# RuoYi-flowable 编译打包与 Docker 部署文档

## 项目概述

- **项目名称**: RuoYi-flowable (若依管理系统 + Flowable 工作流引擎)
- **版本**: 3.8.9
- **后端**: Java 8, Spring Boot 2.5.15, Maven 多模块
- **前端**: Vue 2.6 + Element UI (vue-cli-service)
- **数据库**: MySQL 8.0 (库名: `tony-flowable-24`)
- **缓存**: Redis 7
- **工作流**: Flowable 6.8.0

---

## 一、编译打包

### 1.1 环境要求

| 工具 | 版本要求 |
|------|----------|
| JDK | 1.8+ (实际编译可用 JDK 17) |
| Maven | 3.6+ |
| Node.js | 16+ |
| npm | 8+ |

### 1.2 编译前端

```bash
cd ruoyi-ui

# 安装依赖
npm install

# 生产构建
npm run build:prod

# 构建产物在 ruoyi-ui/dist/ 目录
```

### 1.3 将前端产物嵌入后端

```bash
# 清空原有静态资源
rm -rf ruoyi-admin/src/main/resources/static/*

# 复制前端 dist 到 Spring Boot 静态资源目录
mkdir -p ruoyi-admin/src/main/resources/static
cp -r ruoyi-ui/dist/* ruoyi-admin/src/main/resources/static/
```

### 1.4 编译后端

```bash
# 在项目根目录执行（确保已嵌入前端产物）
mvn clean package -DskipTests

# 构建产物: ruoyi-admin/target/ruoyi-admin.jar (约 101MB)
```

---

## 二、Docker 部署

### 2.1 文件结构

```
docker/
├── Dockerfile                  # 应用镜像构建文件
├── docker-compose.yml          # 编排文件 (MySQL + Redis + App)
├── application-docker.yml      # Docker 环境配置覆写
├── ruoyi-admin.jar             # 编译好的 jar 包
└── mysql-init/                 # 数据库初始化 SQL
    ├── 01_ruoyi.sql            # 核心表
    ├── 02_quartz.sql           # 定时任务表
    └── 03_flowable.sql         # 工作流扩展表
```

### 2.2 Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /app
RUN mkdir -p /home/ruoyi/uploadPath

COPY ruoyi-admin.jar /app/ruoyi-admin.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "-Duser.timezone=Asia/Shanghai", "/app/ruoyi-admin.jar"]
```

### 2.3 docker-compose.yml 服务拓扑

```
┌──────────────────────────────────────────┐
│          docker-compose stack            │
│                                          │
│  ┌──────────┐  ┌───────┐  ┌─────────┐  │
│  │  MySQL   │  │ Redis │  │   App   │  │
│  │  :3306   │  │ :6379 │  │  :8080  │  │
│  │ healthy  │  │healthy│  │         │  │
│  └────┬─────┘  └───┬───┘  └────┬────┘  │
│       │            │           │        │
│       └────────────┼───────────┘        │
│                    │ (depends_on)        │
│              health check               │
└──────────────────────────────────────────┘
```

关键配置覆写（`application-docker.yml`）：

```yaml
spring:
  datasource:
    druid:
      master:
        url: jdbc:mysql://mysql:3306/tony-flowable-24?...&allowPublicKeyRetrieval=true
        username: root
        password: 123456
  redis:
    host: redis
    port: 6379
ruoyi:
  profile: /home/ruoyi/uploadPath
```

### 2.4 将配置打入 jar

```bash
# application-docker.yml 需打入 jar 包，Spring Boot 才能加载 docker profile
cd docker
jar uf ruoyi-admin.jar application-docker.yml
```

---

## 三、部署流程

### 3.1 目标服务器

```
IP:   192.168.11.67
用户: root
密码: DSg.0147
端口: 22 (SSH)
```

### 3.2 服务器要求

- Docker (已安装: `/usr/bin/docker`)
- Docker Compose (已安装: `/usr/local/bin/docker-compose`)

### 3.3 完整一键部署步骤

```bash
# ========== 在本地开发机执行 ==========

# 1. 编译前端并嵌入后端
cd ruoyi-ui && npm install && npm run build:prod
mkdir -p ../ruoyi-admin/src/main/resources/static
cp -r dist/* ../ruoyi-admin/src/main/resources/static/

# 2. 编译后端
cd .. && mvn clean package -DskipTests

# 3. 准备 Docker 部署文件
mkdir -p docker/mysql-init
cp ruoyi-admin/target/ruoyi-admin.jar docker/
cp sql/ry_20240629.sql docker/mysql-init/01_ruoyi.sql
cp sql/quartz.sql docker/mysql-init/02_quartz.sql
cp sql/tony-flowable.sql docker/mysql-init/03_flowable.sql

# 4. 将 Docker 配置打入 jar
cd docker
jar uf ruoyi-admin.jar application-docker.yml

# 5. 打包并传输到服务器
cd ..
tar czf /tmp/ruoyi-deploy.tar.gz -C docker .
sshpass -p 'DSg.0147' scp /tmp/ruoyi-deploy.tar.gz root@192.168.11.67:/opt/

# ========== 在远程服务器执行 ==========

# 6. SSH 到服务器，解压并启动
ssh root@192.168.11.67

mkdir -p /opt/ruoyi-flowable
tar xzf /opt/ruoyi-deploy.tar.gz -C /opt/ruoyi-flowable
cd /opt/ruoyi-flowable

# 7. 构建镜像并启动
docker-compose build --no-cache
docker-compose up -d

# 8. 查看启动日志（首次启动需 2-3 分钟初始化数据库）
docker logs -f ruoyi-app

# 9. 确认启动成功
curl http://localhost:8080/captchaImage
```

### 3.4 快速更新部署

```bash
# 仅更新应用（jar 包更新后）
cd /opt/ruoyi-flowable
docker-compose down app        # 仅停应用
docker-compose build --no-cache app
docker-compose up -d
```

---

## 四、运维命令

```bash
# 查看所有容器状态
cd /opt/ruoyi-flowable && docker-compose ps

# 查看应用日志
docker logs -f ruoyi-app

# 查看近 100 行错误日志
docker logs ruoyi-app 2>&1 | grep -i error | tail -100

# 重启所有服务
docker-compose restart

# 重启单个服务
docker-compose restart app

# 停止所有服务
docker-compose down

# 停止并删除数据卷（⚠️ 会清除数据库数据）
docker-compose down -v

# 进入应用容器
docker exec -it ruoyi-app /bin/bash
```

---

## 五、访问地址

| 地址 | 说明 |
|------|------|
| `http://192.168.11.67:8080/index.html` | 前端登录页面 |
| `http://192.168.11.67:8080/captchaImage` | 验证码接口 |
| `http://192.168.11.67:8080/druid/` | Druid 监控面板 (ruoyi/123456) |

---

## 六、关键配置说明

### 数据库连接

| 参数 | 值 |
|------|-----|
| 地址 | mysql (Docker 内部服务名) |
| 端口 | 3306 |
| 数据库 | tony-flowable-24 |
| 用户 | root |
| 密码 | 123456 |

> ⚠️ **注意**: MySQL 8.0 需要在 JDBC URL 中添加 `allowPublicKeyRetrieval=true`，否则会报 `Public Key Retrieval is not allowed` 错误。

### 数据持久化

| 数据卷 | 用途 |
|--------|------|
| `mysql-data` | MySQL 数据库文件 |
| `redis-data` | Redis 持久化数据 |
| `upload-data` | 文件上传目录 (`/home/ruoyi/uploadPath`) |

### Flowable 自动建表

应用配置中 `flowable.database-schema-update: true`，首次启动时 Flowable 会自动创建所需的数据表（约耗时 1-2 分钟）。生产环境建议改为 `false`。

---

## 七、故障排查

### 7.1 应用反复重启

```bash
docker logs ruoyi-app 2>&1 | grep -E "Exception|Error" | tail -20
```

### 7.2 数据库连接失败

```bash
# 检查 MySQL 是否健康
docker exec ruoyi-mysql mysqladmin ping -h localhost -u root -p123456

# 检查数据库是否存在
docker exec ruoyi-mysql mysql -u root -p123456 -e "SHOW DATABASES;"
```

### 7.3 Redis 连接失败

```bash
# 测试 Redis 连接
docker exec ruoyi-redis redis-cli ping
```

### 7.4 端口被占用

```bash
# 查看端口占用
netstat -tlnp | grep -E "3306|6379|8080"
```
