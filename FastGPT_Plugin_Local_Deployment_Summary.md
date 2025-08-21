# FastGPT 插件服务本地化部署总结

## ⚠️ 重要安全提醒

**本文档包含示例配置和密码，仅供开发测试使用！**

生产环境部署前务必：
- 修改所有默认用户名和密码
- 使用强密码和随机生成的密钥
- 不要将真实的生产配置提交到公共代码仓库
- 定期更换密码和访问密钥

## 项目背景

在 FastGPT 项目中，将 `fastgpt-plugins` 服务从 Docker 容器中剥离，改为本地部署，以便于开发和调试插件功能（特别是基础图表工具）。

## 架构变化

### 变化前（全容器化部署）
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   FastGPT       │    │ fastgpt-plugins  │    │   基础服务      │
│   (Docker)      │───▶│   (Docker)       │    │   (Docker)      │
│   :3000         │    │   :3004          │    │   MongoDB       │
└─────────────────┘    └──────────────────┘    │   Redis         │
                                               │   MinIO         │
                                               │   PostgreSQL    │
                                               └─────────────────┘
```

### 变化后（混合部署）
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   FastGPT       │    │ fastgpt-plugins  │    │   基础服务      │
│   (本地)        │───▶│   (本地)         │    │   (Docker)      │
│   :3000         │    │   :3004          │    │   MongoDB       │
└─────────────────┘    └──────────────────┘    │   Redis         │
                                               │   MinIO         │
                                               │   PostgreSQL    │
                                               └─────────────────┘
```

## 主要配置变化

### 1. Docker Compose 配置修改

**文件**: `deploy/docker/docker-compose-local.yml`

```yaml
# 变化前：使用Docker镜像
fastgpt-plugin:
  image: ghcr.io/labring/fastgpt-plugin:v0.1.5
  container_name: fastgpt-plugin
  restart: always
  networks:
    - fastgpt
  ports:
    - '3004:3000'
  environment:
    - AUTH_TOKEN=1234567
    - MINIO_ENDPOINT=fastgpt-minio
    - MINIO_CUSTOM_ENDPOINT=http://localhost:9000/fastgpt-plugins

# 变化后：注释掉Docker服务配置
# fastgpt-plugin: 本地运行，不使用Docker
# (整个服务配置被注释)
```

### 2. 主服务环境变量配置

**文件**: `projects/app/.env.local`

```bash
# 插件服务配置（保持不变）
PLUGIN_BASE_URL=http://localhost:3004  # 指向本地插件服务
PLUGIN_TOKEN=1234567                   # 认证token

# 基础服务连接配置（改为localhost）
MONGODB_URI=mongodb://myusername:mypassword@localhost:27017/fastgpt?authSource=admin&directConnection=true&maxPoolSize=5&minPoolSize=1&maxConnecting=5
REDIS_URL=redis://default:mypassword@localhost:6379
PG_URL=postgresql://username:password@localhost:5432/postgres
MINIO_CUSTOM_ENDPOINT=http://localhost:9000/fastgpt-plugins
```

### 3. 插件服务环境变量配置

**新建文件**: `fastgpt-plugins/.env.local`

```bash
PORT=3004
AUTH_TOKEN=1234567

# Log level
LOG_LEVEL=debug

# MinIO 配置 - 连接到Docker中的MinIO
MINIO_CUSTOM_ENDPOINT=http://localhost:9000/fastgpt-plugins
MINIO_ENDPOINT=localhost               # 改为localhost，不再是fastgpt-minio
MINIO_PORT=9000
MINIO_USE_SSL=false
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=fastgpt-plugins

# 监控服务配置
SIGNOZ_BASE_URL=
SIGNOZ_SERVICE_NAME=fastgpt-plugin
```

## 遇到的问题及解决方案

### 问题1: 系统模型列表加载失败

**错误现象**:
```
TypeError: Cannot read properties of undefined (reading 'map')
at handler (./src/pages/api/core/ai/model/list.ts:37:35)
```

**根本原因**: 
- FastGPT 主服务在启动时需要从插件服务获取模型列表
- 由于认证失败，`pluginClient.model.list()` 返回失败
- 导致 `global.systemModelList` 为 `undefined`

**解决方案**:
1. 修改插件服务认证方式，支持 `Authorization: Bearer <token>` 格式
2. 确保主服务和插件服务的token配置一致

### 问题2: MinIO URL 访问问题

**错误现象**:
图表生成后返回 URL: `http://fastgpt-minio:9000/fastgpt-plugins/xxx.svg`
但此URL无法从外部访问（fastgpt-minio是容器内部名称）

**根本原因**:
插件服务使用内部容器名称生成URL

**解决方案**:
1. 设置 `MINIO_CUSTOM_ENDPOINT=http://localhost:9000/fastgpt-plugins`
2. 配置MinIO bucket为公开访问：`mc anonymous set public myminio/fastgpt-plugins`

### 问题3: 插件服务认证失败

**错误现象**:
```json
{"message":"Invalid token"}
```

**根本原因**:
认证方式不匹配，插件服务期望的认证格式与主服务发送的不一致

**解决方案**:
插件服务端修改认证逻辑，支持标准的Bearer token认证方式

## 网络连接变化

### 服务间通信方式

| 服务通信 | 变化前 | 变化后 |
|---------|--------|--------|
| 主服务→插件服务 | Docker网络内部通信 | localhost:3004 |
| 插件服务→MinIO | Docker网络内部通信 | localhost:9000 |
| 主服务→MongoDB | Docker网络内部通信 | localhost:27017 |
| 主服务→Redis | Docker网络内部通信 | localhost:6379 |

### 端口映射变化

| 服务 | 变化前 | 变化后 |
|-----|--------|--------|
| FastGPT主服务 | Docker内部 | localhost:3000 |
| 插件服务 | Docker 3004→3000 | 直接 localhost:3004 |
| MinIO | Docker 9000→9000 | 保持 localhost:9000 |

## 验证结果

### 功能验证
- ✅ 主服务正常启动，无模型列表错误
- ✅ 插件服务认证成功
- ✅ 基础图表工具正常工作
- ✅ 图表URL可正常访问
- ✅ MinIO文件存储正常

### API测试结果
```bash
# 插件服务模型列表
curl -H "Authorization: Bearer 1234567" "http://localhost:3004/model/list"
# 返回：大量模型配置数据 ✅

# 插件服务工具列表  
curl -H "Authorization: Bearer 1234567" "http://localhost:3004/tool/list"
# 返回：包含基础图表工具在内的工具列表 ✅

# MinIO访问测试
curl -I "http://localhost:9000/fastgpt-plugins/test.svg"
# 返回：HTTP/1.1 200 OK ✅
```

## 优势分析

### 开发便利性
- ✅ 可直接修改插件服务代码
- ✅ 支持热重载开发
- ✅ 便于调试和测试
- ✅ 代码修改立即生效

### 架构清晰度
- ✅ 服务职责分离更清晰
- ✅ 依赖关系更明确
- ✅ 问题定位更容易

### 性能优化
- ✅ 减少Docker容器开销
- ✅ 网络通信效率提升
- ✅ 资源占用优化

## 注意事项

### 启动顺序
1. 先启动Docker基础服务（MongoDB、Redis、MinIO等）
2. 再启动本地插件服务
3. 最后启动FastGPT主服务

### 环境变量管理
- 确保主服务和插件服务的token配置一致
- MinIO相关配置需要正确指向localhost
- 数据库连接字符串需要使用localhost地址

### 开发工作流
```bash
# 启动基础服务
docker-compose -f deploy/docker/docker-compose-local.yml up -d mongo redis fastgpt-minio pg aiproxy

# 启动插件服务
cd fastgpt-plugins && npm run dev

# 启动主服务  
cd projects/app && npm run dev
```

## 总结

通过将 `fastgpt-plugins` 服务从Docker环境迁移到本地开发环境，成功实现了：

1. **架构优化**: 保持基础服务的稳定性（Docker），同时提供应用服务的开发便利性（本地）
2. **问题解决**: 修复了模型列表加载、MinIO URL访问、服务间认证等关键问题
3. **开发提效**: 为后续的插件功能开发和调试提供了良好的基础

这种混合部署方式在保证系统稳定性的同时，大大提升了开发效率，为基础图表工具等功能的进一步开发奠定了坚实基础。