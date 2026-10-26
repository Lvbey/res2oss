# res2oss

将文件 URL 下载并上传到阿里云 OSS 的 GitHub Actions 执行端。支持两种使用方式：

- **批量任务**：维护仓库根目录的 `rep.txt`，每行一个 URL。
- **Web 单文件任务**：由私有仓库 `Lvbey/res2oss-server` 调用 `workflow_dispatch`，完成后通过 HMAC 签名回调报告结果。

## 工作方式

### 批量处理 `rep.txt`

1. 每当 `rep.txt` 被修改并推送到仓库时，自动触发工作流。
2. 工作流逐行读取 URL，跳过空行和以 `#` 开头的注释。
3. 使用 `curl` 下载文件，并使用 `ossutil` 上传到 OSS。
4. 输出成功/失败统计；单个 URL 失败不会阻止其余 URL 继续处理。

也可以在 GitHub 仓库的 **Actions → Upload to Alibaba OSS → Run workflow** 手动触发批量处理。

### Web 单文件任务

Web 服务会通过 GitHub Actions API 手动触发本工作流，并传入：

| 输入 | 说明 |
|---|---|
| `url` | 要下载的单个文件 URL |
| `request_id` | Web 服务生成的任务 UUID |
| `callback_url` | 任务结果回调到 Web 服务的完整 URL |

Web 任务会将对象放入独立目录，避免不同任务上传同名文件时互相覆盖：

```text
<OSS_PATH>/web/YYYY/MM/DD/<request_id>/<filename>
```

其中 `OSS_PATH` 为空时会省略前缀目录。工作流完成下载/上传后，会向 `callback_url` 发送任务结果，并使用 HMAC-SHA256 签名验证回调来源。批量 `rep.txt` 任务不发送 Web 回调。

## `rep.txt` 格式

每行一个 HTTP/HTTPS URL，支持注释和空行：

```text
https://example.com/files/document.pdf
https://example.com/files/data.zip

# 这是注释，会被跳过
```

## 配置 GitHub Secrets 和 Variables

在仓库 **Settings → Secrets and variables → Actions** 配置以下项目。

### 必需 Secrets

| Secret | 说明 | 示例 |
|---|---|---|
| `OSS_REGION` | OSS 区域 | `cn-hangzhou` |
| `OSS_BUCKET` | Bucket 名称 | `my-bucket` |
| `OSS_ACCESS_KEY_ID` | 阿里云 AccessKey ID | 不要提交到代码 |
| `OSS_ACCESS_KEY_SECRET` | 阿里云 AccessKey Secret | 不要提交到代码 |
| `OSS_ENDPOINT` | OSS Endpoint | `oss-cn-hangzhou.aliyuncs.com` |
| `RES2OSS_CALLBACK_HMAC_SECRET` | Web 任务结果回调的 HMAC 密钥 | 与服务端 `CALLBACK_HMAC_SECRET` 完全一致 |

`OSS_PATH` 可作为 Secret 配置，用于指定 OSS 目标目录，例如 `rep2oss/files`。若未配置，则上传到 Bucket 根目录（Web 任务仍会使用 `web/...` 路径）。

### 可选 Repository Variable

| Variable | 默认值 | 说明 |
|---|---:|---|
| `RES2OSS_MAX_FILE_SIZE_BYTES` | `1073741824`（1 GiB） | 单个文件允许的最大字节数 |

下载同时配置了连接超时、总超时、重试和最大文件大小限制。文件名从 URL 路径提取，URL 查询参数不会用作文件名。

## 安全与运行说明

- 仅提交你有权访问和下载的文件 URL。
- 不要将 OSS 密钥、GitHub Token 或回调 HMAC 密钥写入仓库文件、日志或工作流输入。
- Web 回调 URL 必须能从 GitHub Actions runner 访问；服务端会验证时间戳和 HMAC-SHA256 签名。
- 服务端配置的 `OSS_WEB_PREFIX` 必须与实际 Web 对象前缀一致。例如 `OSS_PATH=rep2oss/files` 时，服务端应配置 `OSS_WEB_PREFIX=rep2oss/files/web`。
- Web 服务使用的 GitHub Fine-grained PAT 应遵循最小权限原则，仅授予触发目标仓库 Actions 所需的权限。

## 相关仓库

- [res2oss-server](https://github.com/Lvbey/res2oss-server)：个人 Web 页面、任务历史和签名下载链接服务。
