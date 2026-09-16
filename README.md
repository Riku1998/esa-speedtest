# eo-speedtest

一个基于 [LibreSpeed](https://github.com/librespeed/speedtest) 的网页版网络速度测试工具，可部署在腾讯 EdgeOne Pages 或阿里云 ESA Pages，用于测试网络的下载速度、上传速度、延迟和抖动。

![测速结果示例](/screenshot.png)

## 部署方法

### 腾讯 EdgeOne Pages

clone 项目并进入项目目录：
``` sh
edgeone pages deploy -n speedtest
```

### 阿里云 ESA Pages（本适配分支）

1. 在阿里云 ESA 控制台开启「函数和 Pages」服务；
2. 左侧导航选择「边缘计算和 AI > 函数和 Pages」，创建 Pages 并选择「导入 GitHub 仓库」；
3. 选择本仓库（`eo-speedtest-esa`），生产分支 `main`；
4. 根目录的 `esa.jsonc` 已配置为纯静态（跳过安装/构建、静态资源目录为根目录），无需在控制台修改构建命令；
5. 部署完成后，**必须配置缓存规则**（`esa.jsonc` 只管理构建与路由，不管理缓存）：
   - 进入 ESA 站点 → 规则 → 缓存规则；
   - 新建规则：匹配路径 `/garbage.bin`；
   - 设置「边缘缓存 TTL」与「浏览器缓存 TTL」为 1 年（或自定义），保存并发布；
   - 作用：让 `garbage.bin` 长期缓存在边缘节点，测速下载才会命中 CDN 缓存而非回源。

## 下载测速原理（本 fork 的修改）

原 LibreSpeed 依赖后端动态接口（`garbage.php` / `empty.php`）生成数据流，而纯静态的 EdgeOne/ESA Pages 没有这些接口：下载测速会反复请求不存在的路径、拿到 404 小页面，导致**下载速度严重偏低、上传正常**。

本 fork 改为**纯静态 + CDN 缓存下载**，链路为「用户 → 就近 CDN 节点 → 缓存测速文件」：

- `garbage.bin`：48 MiB 随机不可压缩数据，作为测速文件（48 MiB ÷ 8 MiB 段 = 6 段，正好配 6 线程 Range）；
- 下载测试对**恒定 URL** 发起**多线程 Range 请求**（默认 6 线程、每段 8 MiB，按 Range 循环下载同一文件），URL 不带随机缓存破坏参数，保证每次请求都命中 CDN 边缘缓存；
- 缓存配置：EdgeOne 版用 `edgeone.json`；阿里云 ESA 版在 ESA 控制台配置缓存规则（见上）；
- 上传/延迟/抖动沿用 LibreSpeed 逻辑（上传测本地上行进度，不受服务端影响）。

> 注意：腾讯云 `edgeone.json` 为 EdgeOne 私有格式，在阿里云 ESA 上不生效；本 ESA 适配分支以 `esa.jsonc` 管理构建，缓存规则在控制台配置，请勿把两个配置文件混用。

如需调整下载参数：修改 `speedtest_worker.js` 中的 `dl_file_size_mb`（须与实际文件大小一致）和 `dl_range_mb`。

## 说明

- 免费版 CDN 通常存在带宽 QoS 限速（腾讯 EdgeOne 免费版实测单连接约 7 Mbps、总吞吐约 15–23 Mbps；阿里云 ESA 免费版官方明示限制峰值带宽与单请求速率，但未公布具体数值），测速结果受平台免费档位限制，不代表家庭宽带真实能力；
- 中国大陆加速需完成 ICP 备案，否则自定义域名无法接入国内节点。

## 许可证

本项目基于 [LibreSpeed](https://github.com/librespeed/speedtest)，遵循其开源许可证。
