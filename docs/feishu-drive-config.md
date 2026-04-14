# 飞书云盘配置完整指南

## 📋 配置总结

**配置日期：** 2026 年 4 月 14 日  
**应用名称：** 龙虾 1 号  
**App ID：** `cli_a92c1d5a91375cd9`  
**应用类型：** 企业内部应用  

---

## 🎯 配置目标

实现通过 Hermes Agent API 访问飞书个人云盘，支持：
- ✅ 列出文件/文件夹
- ✅ 下载文件
- ✅ 创建文件夹
- ✅ 删除文件
- ⚠️ 上传文件（API 限制，需手动上传）

---

## 📝 配置步骤

### 步骤 1：创建飞书应用

1. 打开飞书开放平台：https://open.feishu.cn/app
2. 创建**企业内部应用**
3. 记录 App ID 和 App Secret

### 步骤 2：配置应用权限

#### 2.1 应用身份权限

在 **"权限管理"** → **"应用身份权限"** 中添加：
- `drive:drive` - 云盘完整读写
- `drive:drive:readonly` - 云盘只读

#### 2.2 用户身份权限

在 **"权限管理"** → **"用户身份权限"** 中添加：
- `drive:drive` - 云盘完整读写（用户身份）
- `drive:drive:readonly` - 云盘只读（用户身份）

**⚠️ 关键发现：** 访问个人云盘必须使用**用户身份权限**，应用身份权限只能访问企业共享空间。

#### 2.3 发布应用

权限添加后，必须到 **"版本管理与发布"** 点击 **"发布"** 才能生效。

### 步骤 3：配置重定向 URI

在 **"开发配置"** 中添加重定向 URI：
- `https://open.feishu.cn`
- 或 `urn:ietf:params:oauth:2.0:oob`

### 步骤 4：OAuth 用户授权

#### 4.1 生成授权链接

```
https://open.feishu.cn/open-apis/authen/v1/authorize?app_id=cli_a92c1d5a91375cd9&response_type=code&scope=drive:drive&state=auth
```

**⚠️ 关键发现：** 授权链接必须包含 `scope=drive:drive` 参数，否则获取的 token 不包含云盘权限。

#### 4.2 换取 user_access_token

使用授权码换取 token：

```bash
curl -X POST "https://open.feishu.cn/open-apis/authen/v1/access_token" \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "authorization_code",
    "code": "<授权码>",
    "app_id": "<APP_ID>",
    "app_secret": "<APP_SECRET>"
  }'
```

响应示例：
```json
{
  "code": 0,
  "data": {
    "access_token": "u-xxxxxxxxxxxx",
    "refresh_token": "ur-xxxxxxxxxxxx",
    "expires_in": 7200
  }
}
```

### 步骤 5：保存凭证

将 token 保存到 `~/.hermes/.env`：

```bash
FEISHU_APP_ID="cli_a92c1d5a91375cd9"
FEISHU_APP_SECRET="<完整的 secret>"
FEISHU_USER_ACCESS_TOKEN="u-xxxxxxxxxxxx"
FEISHU_USER_REFRESH_TOKEN="ur-xxxxxxxxxxxx"
```

### 步骤 6：测试云盘 API

```bash
curl -X GET "https://open.feishu.cn/open-apis/drive/v1/files?page_size=50" \
  -H "Authorization: Bearer $FEISHU_USER_ACCESS_TOKEN"
```

成功响应：
```json
{
  "code": 0,
  "data": {
    "files": [
      {
        "name": "数学",
        "type": "folder",
        "token": "RFMLfhVsIleNotdrlSPcMUbrnrf"
      }
    ]
  }
}
```

---

## 🔧 脚本配置

### 创建管理脚本

位置：`~/.hermes/scripts/feishu_drive.sh`

支持命令：
```bash
# 列出文件
~/.hermes/scripts/feishu_drive.sh list

# 列出指定文件夹
~/.hermes/scripts/feishu_drive.sh list <folder_token>

# 下载文件
~/.hermes/scripts/feishu_drive.sh download <file_token>

# 读取文件内容
~/.hermes/scripts/feishu_drive.sh read <file_token>

# 创建文件夹
~/.hermes/scripts/feishu_drive.sh create_folder <文件夹名>

# 删除文件
~/.hermes/scripts/feishu_drive.sh delete <file_token>

# 上传文件（API 限制，可能不可用）
~/.hermes/scripts/feishu_drive.sh upload <本地文件路径> [folder_token]
```

---

## ⚠️ 遇到的问题与解决方案

### 问题 1：权限错误 99991672

**错误信息：**
```
Access denied. One of the following scopes is required: [drive:drive, drive:drive:readonly]
```

**原因：** 应用身份权限未开通

**解决：** 在飞书开放平台添加应用身份权限 `drive:drive` 并发布应用

---

### 问题 2：权限错误 99991679

**错误信息：**
```
Unauthorized. You do not have permission to perform the requested operation on the resource.
required one of these privileges under the user identity: [drive:drive, drive:drive:readonly]
```

**原因：** 访问个人云盘需要**用户身份权限**，而不是应用身份权限

**解决：** 
1. 在飞书开放平台添加**用户身份权限** `drive:drive`
2. 重新进行 OAuth 授权（带 `scope=drive:drive` 参数）
3. 使用新的 user_access_token

---

### 问题 3：授权码过期 20003

**错误信息：**
```
code is expired
```

**原因：** 授权码有效期只有 5 分钟

**解决：** 重新生成授权链接，扫码后立即换取 token

---

### 问题 4：重定向 URI 错误

**错误信息：**
```
Invalid redirect URL
```

**原因：** 授权链接中的 redirect_uri 未在飞书开放平台配置

**解决：** 在"开发配置"中添加重定向 URI，或使用不依赖重定向的 OOB 模式

---

### 问题 5：上传 API 404

**错误信息：**
```
404 page not found
```

**原因：** 飞书云盘文件上传 API 需要分片上传流程，或使用特定的 multipart 格式

**解决：** 使用飞书 App 或网页版手动上传文件

---

## 📊 API 对比

| API | 权限类型 | 访问范围 | 用途 |
|-----|---------|---------|------|
| `app_access_token` | 应用身份 | 企业共享空间 | 访问企业云盘 |
| `user_access_token` | 用户身份 | 个人云盘 + 共享空间 | 访问个人云盘 ✅ |

**结论：** 访问个人云盘必须使用 `user_access_token`

---

## 🎯 关键配置清单

### 飞书开放平台配置

- [x] 创建企业内部应用
- [x] 添加应用身份权限 `drive:drive`
- [x] 添加用户身份权限 `drive:drive`
- [x] 配置重定向 URI
- [x] 发布应用
- [x] 配置可见范围（添加用户）

### 本地配置

- [x] 保存 App ID 和 App Secret 到 `~/.hermes/.env`
- [x] OAuth 授权获取 user_access_token
- [x] 保存 user_access_token 到 `~/.hermes/.env`
- [x] 创建云盘管理脚本
- [x] 测试 API 连接

---

## 📁 云盘文件结构示例

```
📂 根目录
├── 📁 数学/
│   ├── 第 5 讲 + 三角函数综合讲义解析.pdf
│   ├── 第 4 讲 + 正切函数和正弦型函数讲义解析.pdf
│   ├── 春季第五讲课堂笔记.pdf
│   └── 春季第四讲课堂笔记.pdf
└── 📄 非谓语动词语法汇总.pdf
```

---

## 🔑 重要经验

1. **权限类型区分：** 应用身份权限 vs 用户身份权限
   - 应用身份 → 企业共享空间
   - 用户身份 → 个人云盘

2. **授权链接必须带 scope：**
   ```
   &scope=drive:drive
   ```

3. **授权码有效期短：** 只有 5 分钟，获取后立即换取 token

4. **权限变更后需发布：** 添加权限后必须发布应用才能生效

5. **上传 API 限制：** 文件上传 API 对第三方调用有限制，建议手动上传

6. **token 有效期：** user_access_token 有效期约 2 小时，过期后使用 refresh_token 刷新

---

## 📚 参考文档

- 飞书开放平台：https://open.feishu.cn/app
- 云盘 API 文档：https://open.feishu.cn/document/ukGEyYjL0QjL0SN
- OAuth 授权：https://open.feishu.cn/document/ugTN1YjL4UTN24CO1UjN

---

**配置完成时间：** 2026 年 4 月 14 日  
**配置人员：** 大兵哥  
**应用名称：** 龙虾 1 号
