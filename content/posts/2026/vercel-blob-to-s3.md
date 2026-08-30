---
title: 基于 Vercel Blob 构建 S3 兼容的对象存储网关
date: 2026-08-30 20:11:00
updated: 2026-08-30 20:13:22
categories: [技术]
aside: [toc]
tags: [Twikoo, Vercel, S3, 图床, R2]
---

::timeline
{背景}

博客评论系统 Twikoo 的图片上传依赖第三方图床，近期服务稳定性无法保证。

{尝试}

考虑迁移至 Cloudflare R2，因支付方式绑定受限而搁置。

{结果}

转向 Vercel Blob，并为其构建了一层 S3 兼容网关。
::

## 问题的由来

本站评论区由 [Twikoo](https://twikoo.js.org/) 驱动，其图片上传通过 `imgUploader` 回调实现，存储需要自行对接图床。此前一直使用 sm.ms，近期该服务可靠性明显下降：已上传的图片在数日后出现失效，且无明确公告。免费第三方图床的可持续性本就存疑，迁移势在必行。

候选方案的评估结果如下：

| 方案 | 问题 |
| --- | --- |
| Cloudflare R2 | 需要绑定支付方式，本人验证无法通过 |
| 直接使用 Vercel Blob SDK | 仅支持服务端调用，浏览器端无法直接使用 |
| AWS S3 | 成本与账户管理开销与需求不匹配（因本人特殊原因无法使用） |
| 自建图床 | 运维成本过高 |

排除以上选项后，Vercel Blob 成为最合适的存储后端：开通即用，免费额度为 1GB 存储与每月 10GB 带宽，与本站评论区的图片量级完全匹配；且博客本身部署于 Vercel，存储与计算处于同一生态。

但它存在一个关键障碍：**Vercel Blob 的 API 与 S3 不兼容**——既不支持 S3 的 XML 协议，也不支持 SigV4 签名认证，官方 SDK 又只能在服务端运行。而 Twikoo 的上传行为发生在浏览器端，存储能力无法直接接入。

解决思路有二：编写一个专用的上传转发接口，或者将 Blob 完整封装为 S3 兼容服务。后者具备更普遍的复用价值，于是实现了本项目：

::link-card
---
title: Krits03/vercel-blob-to-s3
description: 将 Vercel Blob 包装为 S3 兼容协议 + 简单 HTTP 双接口的对象存储网关
icon: https://github.com/favicon.ico
link: https://github.com/Krits03/vercel-blob-to-s3
class: gradient-card active
---
::

## 设计与实现

项目在 Vercel Blob 之上构建网关层，对外同时提供两套接口，共享同一个 Blob Store 后端：

- **S3 兼容接口**：路径形如 `/s3/{bucket}/{key}`，支持 PUT、GET、HEAD、DELETE 四类对象操作，认证采用标准 SigV4 签名校验。任何使用 `@aws-sdk/client-s3` 或 AWS CLI 的调用方均可零改动接入。网关本身没有 bucket 实体，`bucket` 仅作为路径前缀用于逻辑分组，对象实际存储于 Blob 中 `{bucket}/{key}` 对应的位置。
- **简单 HTTP 接口**：上传为携带 Bearer Token 的 `fetch` POST 请求；下载默认以 302 重定向至 Blob CDN 地址，附加 `?proxy=1` 参数则切换为流式代理，隐藏源地址并支持 Range 请求。

完整请求链路如下：

```
浏览器 (Twikoo imgUploader)
  │  S3 模式：@aws-sdk/client-s3 自动计算 SigV4 签名
  │  HTTP 模式：请求头携带 Authorization: Bearer <token>
  ▼
网关层 (Vercel Function / Cloudflare Worker)
  │  认证校验 → 解析路径 → 提取 bucket 与 key
  ▼
Vercel Blob (存储后端)
  │  返回公开 URL / 元信息
  ▼
浏览器 → Twikoo 渲染图片
```

### 两个部署版本

::tab{:tabs='["Vercel 版","Cloudflare 版"]'}
#tab1
基于 Next.js 15 App Router 的 Route Handler 实现，通过官方 `@vercel/blob` SDK 访问存储，签名校验使用 `node:crypto` 完成 HMAC 计算，并以 `timingSafeEqual` 进行时序安全的比较。首页内置拖拽上传页面，可用于日常手动传图。

限制：Serverless 存在冷启动，请求体大小上限为默认的 4.5MB。

#tab2
基于 Hono 4 运行于 Cloudflare Workers。Worker 环境无法加载 Node SDK，改为直接调用 Vercel Blob REST API；加密部分换用 Web Crypto API，并手动实现了基于 XOR 累积的常量时间比较以替代 `timingSafeEqual`。

优势：无冷启动，请求体上限提升至 100MB。代价：无内置上传界面，HEAD 操作暂不支持。
::

两版功能基本对齐，选择依据为：需要网页上传入口或希望减少维护面，选 Vercel 版；对延迟与文件大小敏感，选 Cloudflare 版。

## 接入 Twikoo

推荐使用简单 HTTP 接口，前端无需引入 S3 SDK：

```js [twikoo.init]
twikoo.init({
  envId: '<你的 envId>',
  imgUploader: {
    async upload(file) {
      const res = await fetch(
        'https://<网关域名>/api/upload?name=' +
          encodeURIComponent(file.name) + '&path=comments',
        {
          method: 'POST',
          headers: {
            'Authorization': 'Bearer <UPLOAD_TOKEN>',
            'Content-Type': file.type,
          },
          body: file,
        }
      );
      const data = await res.json();
      return { url: data.url };
    },
  },
});
```

若调用方已具备 S3 客户端，也可将 `endpoint` 指向网关的 `/s3` 路径并启用 `forcePathStyle`，按标准 S3 协议读写。

### 环境变量

| 变量名 | 必填 | 说明 |
| --- | :-: | --- |
| `BLOB_READ_WRITE_TOKEN` | 是 | Vercel Blob 读写令牌 |
| `S3_ACCESS_KEY` | 是 | S3 签名校验用 AccessKey，自定义 |
| `S3_SECRET_KEY` | 是 | S3 签名校验用 SecretKey，自定义 |
| `UPLOAD_TOKEN` | 是 | 简单接口的 Bearer Token，自定义 |

自定义密钥建议使用强随机值，可执行 :copy{prompt code="openssl rand -hex 32"} 生成。

## 限制与注意事项

::alert{type="warning" card}
#title
密钥暴露于浏览器端
#default
前端直传意味着 Token 会出现在客户端代码中，无法回避。缓解因素是 Vercel Blob 的公开 URL 不可枚举，读取无权限要求，整体风险可控。对安全性要求更高的场景，应将上传逻辑移至服务端，网关仅对内开放。
::

其余已知限制：

- S3 接口仅覆盖对象级操作，不支持 ListObjects 等 bucket 级 API——网关没有真实的 bucket 概念。
- 上传关闭了 `addRandomSuffix`，文件名依赖时间戳区分，理论上存在冲突可能。
- 免费额度为 Blob 的 1GB 存储与每月 10GB 带宽；博客评论区场景下通常不会触及上限。

## 结语

项目规模不大，两个版本合计数千行代码。收获主要在于将 SigV4 签名流程完整实现了一遍：从规范请求构造、待签字符串派生，到多级 HMAC 密钥派生与时序安全比较，对 AWS 认证机制的设计意图有了具体认识。

仓库中的 `vercel/DEPLOY.md` 与 `cf/DEPLOY.md` 包含从零开始的部署步骤、环境变量配置与常见问题排查。如有类似需求——不限于 Twikoo，任何需要在浏览器端使用 Vercel Blob 的场景——欢迎使用或提交 issue。

本站评论图床已切换，欢迎在评论区验证。

这应该是开学前的最后一篇了awa.
