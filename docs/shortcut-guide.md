# WLOC 虚拟定位 - 使用说明

## 工作原理

```
用户在手机 Safari 打开选点页面
  → 地图选位置 / 搜索地名 / 粘贴地图链接
  → 点击「储存到设备」
  → 页面请求 https://gs-loc.apple.com/wloc-settings/save?lon=x&lat=y
  → 代理模块拦截请求 → wloc-settings.js 写入 $persistentStore
  → 下次 Apple 定位触发 → wloc.js 读取坐标 → 修改定位响应
```

如果模块未启用 → 请求不会被拦截 → 页面提示检查 MITM/模块配置。

---

## 使用方法

### 1. 安装模块（一次性）
从 [本仓库订阅地址](../README.md#订阅地址) 安装对应平台的模块并启用 MITM。原仓库失效后，停用或删除旧模块并重新订阅，不要同时启用两份 WLOC 模块。

### 2. 打开选点页面
按下方部署说明创建你自己的 Worker/Pages，然后在 Safari 中打开部署得到的地址（建议添加到主屏幕）:
```
https://你的worker域名/
```

> Worker 不存储任何数据。坐标直接写入你的设备本地；`/api/parse` 只在解析链接时
> 临时向地图服务发一次请求，处理完即丢，不写存储、不落日志。

### 3. 选择位置
- **点击地图** — 直接点选
- **搜索地名** — 输入"上海外滩"等
- **粘贴链接** — 从 Apple Maps / Google Maps / 高德 / 百度复制分享链接
- **当前位置** — 使用浏览器定位

### 4. 储存到设备
点击「储存到设备」→ 显示 ✓ 即成功。

---

## 部署自己的选点页面

在自己的 Cloudflare 账户部署，无需 KV 或数据库绑定：

```bash
git clone https://github.com/WANGXU-SU/Yu9191-wloc.git
cd Yu9191-wloc/worker
npm install
npx wrangler login
npm run deploy
```

不需要 KV、不需要数据库、不需要环境变量。

> Worker 由 `worker/src/` 下的多个模块打包而成（页面模板、链接解析、坐标换算），
> 不能靠在 Dashboard 里粘贴单个文件来部署，请用上面的 wrangler 流程。

---

## 模块配置

模块包含两条脚本规则（已自动配置，用户无需操作）：

| 规则 | 类型 | 路径 | 作用 |
|------|------|------|------|
| Apple WLOC | http-response | `/clls/wloc` | 修改定位响应 |
| WLOC Settings | http-request | `/wloc-settings/save` | 接收选点页面写入 |

MITM 主机名: `gs-loc.apple.com, gs-loc-cn.apple.com`（已包含在模块中）

---

## 储存失败排查

页面显示红色提示时，检查：
1. **模块已启用** — 在代理工具中确认 WLOC 模块开关打开
2. **MITM 证书** — 已安装并信任 CA 证书
3. **MITM 主机名** — 包含 `gs-loc.apple.com`
4. **代理连接** — 当前网络走代理（Safari 请求会经过代理）

---

## 备选：不部署页面，直接保存坐标

确认新模块已启用且 MITM 证书已信任，在 Safari 打开以下地址。将占位符替换为实际的 **WGS84** 坐标数值；`lon` 是经度，`lat` 是纬度：

```text
https://gs-loc.apple.com/wloc-settings/save?lon=<经度>&lat=<纬度>
```

返回 `"success":true` 表示保存成功。查询或清除坐标可分别访问：

```text
https://gs-loc.apple.com/wloc-settings/save?action=query
https://gs-loc.apple.com/wloc-settings/save?action=clear
```

这些地址由代理模块在设备上拦截，不是需要自行部署的网站。不要把域名换成 GitHub。清除后若模块参数中有自定义坐标，脚本仍会使用模块参数；恢复真实定位最直接的方法是停用模块。高版本 iOS 可能需要重启清除定位缓存。

## 原版快捷指令迁移

在快捷指令编辑器中，把原作者的地图链接解析接口改为你自己部署的 `https://<你的 Worker 域名>/api/parse`，保留原有查询参数。Apple 的设置和清除接口保持不变。没有部署 Worker/Pages 时，先使用上面的直接保存方式。

## 备选：手动编辑（BoxJS）

不使用选点页面时，可在 BoxJS 中直接编辑 `wloc_settings`:
```json
{"longitude":121.4737,"latitude":31.2304,"accuracy":25}
```

优先级: 已储存坐标 > 模块参数 > 默认值
