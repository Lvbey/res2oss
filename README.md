# res2oss
resource to oss

将 `rep.txt` 中的链接文件自动下载并上传到阿里云 OSS。

## 工作原理

每当 `rep.txt` 被修改并推送到仓库时，GitHub Actions 会：

1. 读取 `rep.txt` 中的每一行链接
2. 使用 `curl` 下载对应文件
3. 使用 `ossutil` 将文件上传到阿里云 OSS，保留原始文件名

## rep.txt 格式

每行一个 URL，支持注释和空行：

```
https://example.com/files/document.pdf
https://example.com/files/data.zip

# 这是注释，会被跳过
```

## 触发方式

- **自动触发**：push 修改了 `rep.txt` 时自动运行
- **手动触发**：在 GitHub Actions 页面点击 "Run workflow"

## 配置 Secrets

在仓库 **Settings → Secrets and variables → Actions** 中添加以下 secrets：

| Secret | 说明 | 示例 |
|--------|------|------|
| `OSS_REGION` | OSS 区域 | `cn-hangzhou` |
| `OSS_BUCKET` | Bucket 名称 | `my-bucket` |
| `OSS_ACCESS_KEY_ID` | 阿里云 AccessKey ID | `LTAI5t...` |
| `OSS_ACCESS_KEY_SECRET` | 阿里云 AccessKey Secret | `...` |
| `OSS_ENDPOINT` | OSS Endpoint | `oss-cn-hangzhou.aliyuncs.com` |
| `OSS_PATH` | 上传目标目录（可选） | `rep2oss/files` |

## 上传路径规则

- 未设置 `OSS_PATH`：`oss://{BUCKET}/{filename}`
- 设置了 `OSS_PATH`：`oss://{BUCKET}/{OSS_PATH}/{filename}`

文件名从 URL 中提取，URL 查询参数会被去除。若出现同名文件，会追加行号后缀避免冲突。

## 特性

- 下载失败自动重试 3 次，超时 30 秒
- 自动跳过空行和 `#` 开头的注释行
- 运行结束后输出成功/失败统计
