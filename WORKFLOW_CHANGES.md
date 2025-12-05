# GitHub Actions Workflow 改进说明

## 改进内容总结

本次修改针对 `.github/workflows/build_and_release.yml` 文件，实现了 Docker 工具链构建的增强和优化。

## 主要改进

### 1. 添加手动触发支持 (workflow_dispatch)

```yaml
workflow_dispatch:
  inputs:
    cross_host:
      description: 'Cross compilation target (e.g., mipsel-unknown-linux-musl)'
      required: false
      default: 'mipsel-unknown-linux-musl'
    musl_cross_version:
      description: 'MUSL cross toolchain version'
      required: false
      default: '20250206'
```

**功能说明：**
- 支持通过 GitHub Actions 界面手动触发工作流
- 可自定义交叉编译目标架构 (cross_host)
- 可自定义 MUSL 工具链版本 (musl_cross_version)
- 保留默认值：mipsel-unknown-linux-musl 和 20250206

### 2. Docker Hub 镜像加速配置

新增步骤 "Configure Docker daemon with mirror acceleration"，配置多个镜像加速源：

```yaml
"registry-mirrors": [
  "https://docker.m.daocloud.io",
  "https://docker.1panel.live",
  "https://dockerhub.icu",
  "https://docker.nastool.de"
]
```

**功能说明：**
- 解决 Docker Hub 500 错误和访问慢的问题
- 配置多个镜像源提供冗余
- 自动重启 Docker 守护进程应用配置
- 等待 Docker 服务就绪后继续执行

### 3. Docker 构建重试机制

在 "Build GCC 14.2 musl toolchain image" 步骤中添加重试逻辑：

```bash
retry_count=0
max_retries=3
retry_delay=10

while [ $retry_count -lt $max_retries ]; do
  echo "Building Docker image (attempt $((retry_count + 1))/$max_retries)..."
  if docker build ...; then
    echo "Docker build succeeded!"
    break
  else
    retry_count=$((retry_count + 1))
    if [ $retry_count -lt $max_retries ]; then
      echo "Docker build failed, retrying in ${retry_delay}s..."
      sleep $retry_delay
    fi
  fi
done
```

**功能说明：**
- 最多重试 3 次
- 每次重试间隔 10 秒
- 失败后自动重试，提高构建成功率
- 所有尝试失败后才报错退出

### 4. 动态环境变量支持

修改环境变量配置以支持手动触发时的自定义参数：

```yaml
env:
  CROSS_HOST: ${{ github.event.inputs.cross_host || 'mipsel-unknown-linux-musl' }}
  MUSL_CROSS_VERSION: ${{ github.event.inputs.musl_cross_version || '20250206' }}
```

**功能说明：**
- 手动触发时使用用户输入的参数
- 自动触发时使用默认值
- 保持向后兼容性

## 构建配置

### Docker 镜像构建
- **基础镜像**: ubuntu:24.04
- **目标架构**: mipsel-unknown-linux-musl
- **工具链版本**: MUSL_CROSS_VERSION 20250206
- **镜像标签**: custom-musl-toolchain:mipsel-unknown-linux-musl-20250206

### 工具链安装流程
1. 安装必要的依赖包 (ca-certificates, curl, xz-utils)
2. 从 GitHub 下载预编译的 musl-cross 工具链
3. 验证 SHA256 校验和
4. 解压到 /cross_root 目录
5. 清理临时文件和依赖包

## 工作流触发方式

支持以下触发方式：

1. **手动触发** (workflow_dispatch): 通过 GitHub Actions 页面手动运行
2. **推送触发** (push): 推送到任意分支
3. **Pull Request** (pull_request): 创建或更新 PR
4. **发布** (release): 创建新的 release
5. **定时任务** (schedule): 每周六 00:00 UTC 自动运行

## 使用方法

### 手动触发工作流

1. 访问 GitHub 仓库的 Actions 页面
2. 选择 "Build and Release" 工作流
3. 点击 "Run workflow" 按钮
4. 可选择自定义参数：
   - Cross compilation target (默认: mipsel-unknown-linux-musl)
   - MUSL cross toolchain version (默认: 20250206)
5. 点击 "Run workflow" 开始构建

### 自动触发

工作流会在以下情况自动运行：
- 代码推送到任意分支
- 创建或更新 Pull Request
- 创建新的 Release
- 每周六自动构建

## 优化效果

1. **可靠性提升**: 通过镜像加速和重试机制，显著提高构建成功率
2. **灵活性增强**: 支持手动触发和参数自定义
3. **速度优化**: 使用国内镜像源加速 Docker 镜像拉取
4. **用户体验**: 提供清晰的构建日志和状态信息

## 技术细节

### Docker 守护进程配置
- 最大并发下载数: 10
- 日志驱动: json-file
- 日志文件大小限制: 20MB
- 日志文件数量限制: 3

### 错误处理
- Docker 服务启动超时时间: 60 秒 (30 次检查，每次等待 2 秒)
- 构建重试次数: 3 次
- 重试间隔: 10 秒

## 兼容性

本次修改完全向后兼容，不会影响现有的自动化流程：
- 保留所有原有触发条件
- 保留原有环境变量和默认值
- 保留原有的构建步骤和输出

## 测试建议

建议进行以下测试以验证工作流正常运行：

1. 手动触发测试（使用默认参数）
2. 手动触发测试（使用自定义参数）
3. 推送代码触发测试
4. 验证 Docker 镜像成功构建
5. 验证 aria2 编译产物生成

## 注意事项

1. Docker 镜像加速源可能会变化，如果某个源失效，Docker 会自动尝试下一个
2. 重试机制针对临时性网络问题，持续性错误需要人工介入
3. 手动触发时输入的参数必须是有效的工具链目标和版本号
