# 把 Pi Web 从 Next.js 搬到 vinext：一次构建管线的迁移实录

> 本文的 HTML 版本带有可交互的图表（构建时间切换、构建竞速动画、磁盘占用对比），见 `pi-web-to-vinext.html`。

先说结论，因为图表更有意思：这次搬迁赢的是**开发和发布的流程**，不是用户的网速。构建时间快了将近 6 倍，发布物体积小了四成，磁盘上那个 695 MB 的缓存目录彻底消失；而打开页面时浏览器真正下载的东西，迁移前后是同一个数量级——我用 WebKit 实测过，差距在 1% 以内。

这篇文章记录我把自己的一个项目从 Next.js 16 搬到 [vinext](https://github.com/cloudflare/vinext) 的完整过程：为什么要搬、vinext 是什么、中间踩了哪些坑、以及最后那一堆实测数字。所有数字都是在这台机器上现测的，不是从 release note 里抄的。

## 01 缘起：一次 28 秒的构建

[Pi Web](https://github.com/brynne8/pi-web-bun) 是我维护的一个开源项目：[pi 编程智能体](https://github.com/earendil-works/pi)的本地 Web 界面。它和 pi 共用同一套配置和会话文件（就在 `~/.pi/agent` 里），所以在浏览器里你可以浏览和继续之前的对话、运行 agent 回合、配置模型和 MCP 服务器、顺便看看项目里的文件和 Git 改动。服务端跑在本机，界面是个 React 单页应用。

它原本是个标准的 Next.js 16 应用，而我这边是一条 Bun 优先的分支。问题出在每次发布前的那一次构建上：

```bash
# 迁移前，Bun 下构建 Next.js 16
$ time bun run build
real    27.97s
$ du -sh .next
727M    .next        # 其中 695M 是 webpack 的 cache
```

28 秒听起来还能忍，但它是**每次改一行代码都要付的钱**，而且那 695 MB 的缓存目录会一直躺在仓库里，CI 上每次都是全新的 checkout，等于每次都按冷构建的价格结算。更别说 Next 16 在我们这个项目上还只能用 webpack（Turbopack 在我们依赖的某个原生模块上有问题），这玩意儿在 Next 16 里已经是「legacy 但还能用」的状态。

所以当我知道 Cloudflare 做了 vinext 之后，就决定试试。

## 02 vinext 是什么

[vinext](https://github.com/cloudflare/vinext) 是 Cloudflare 开源的的一个项目，目标一句话能说完：**让 Next.js 的 App Router 代码跑在 Vite 上**。

它不是又一个元框架，也不是让你重写页面。你的 `app/` 目录、路由约定、`route.ts` 里的 `GET`/`POST` 导出、`next/link`、Server Component——全都保持原样。变的是底下的引擎：

```
next dev    →  vinext dev
next build  →  vinext build
next start  →  vinext start
```

配置上，它把 Next 的配置接过来改名：`next.config.ts` 变成 `next-config.ts`（导出还是一个 `NextConfig` 对象，类型直接从 vinext 里 import），再在 `vite.config.ts` 里作为插件引入。我这边改完是这样的：

```ts
import { defineConfig } from "vite";
import vinext from "vinext";
import { nextConfig } from "./next-config.ts";

export default defineConfig({
  plugins: [vinext({ nextConfig })],
  // 我们项目自己的两个补丁，下面会讲
  build: { target: "safari16.2" },
});
```

底层是 **Vite 8 + Rolldown**。Rolldown 是 Rolldown 团队用 Rust 写的打包器，接口对齐 Rollup，目标是替代 webpack/esbuild 那一层——所以「快到不正常」这件事是有来处的，不是魔法。另外它还自带一个兼容性检查命令 `vinext check`，会扫描你的代码里用到的 Next 特性，告诉你哪些不支持。

> **值得说明**
> vinext 做的是「Next.js 兼容层」，不是「Next.js 的替代品」。它不实现 Next 的全部能力，也不打算让你迁移到 Cloudflare Workers 才能用——本地 `vinext start` 就是一个普通的 Node/Bun 生产服务器。对我们这种「一个本机 Web UI」的项目，正合适。

## 03 迁移过程：代码几乎没动，坑都在边上

业务代码的改动小得出乎意料。API 路由里的 `NextRequest`、`NextResponse` 改成从 `vinext/shims/server` 导入，一共 62 处，机械替换；`proxy.ts`、PWA 的 `sw.js`、静态资源都没动。真正花时间的是下面这三件事，它们全都不是 vinext 的问题，但都会在迁移后第一次暴露出来。

### 坑一：dev 下所有写请求都变成了 GET

迁完之后我第一次在 dev 里发消息，前端所有的 `POST` 都变成了 `GET`，界面直接瘫掉。我一开始怀疑 vinext，写了最小复现：一个非原生的 Request-like 对象加上 init 传给 `new Request()`，Bun 会把 method 丢掉。

> **根因不在 vinext**
> 这是 Bun 的 bug（[oven-sh/bun#42759](https://github.com/oven-sh/bun/issues/42759)）：`new Request(input, init)` 在 init 没有 `method` 时，没有回落到 input 上的 `method`。vinext 的 Node 适配器正是这么构造请求的，Node 的 undici 会抛错从而走重建逻辑，Bun 不会，于是静默降级成 GET。修复在 Bun 的 PR #41994 里，被 Jarred Sumner 的 HTTP 整合大 PR #46528 吸收。我装了那个 PR 的 CI 构建，问题消失。

这个坑的教训是：**换构建工具的时候，会把运行时里一直存在但没被触发的 bug 激活**。Next 的 dev server 自己不这么构造请求，所以这个 bug 在我们的代码里躺了很久都没人发现。

### 坑二：一个文件锁让整个侧栏状态写入 500

第二个坑更朴实。侧栏的置顶、归档、项目排序要写一个 JSON 状态文件，用了 `proper-lockfile` 加文件锁。迁移后所有写入都返回 500，错误信息是 `options.fs.mkdir is not a function`。

原因是 Vite 的模块运行器会给 dev server 的环境注入自己的一份 `fs` 对象——它没有 `mkdir`。而 `proper-lockfile` 是 CJS，它 `require("graceful-fs")` 时拿到的是被加工过的那份。修法是把这类包声明成 `serverExternalPackages`，让 Node 原生加载它们：

```ts
serverExternalPackages: [
  "node-pty", "undici", "web-push",
  // proper-lockfile 是 CJS，会对自己 require 到的 fs 调 mkdir；
  // Vite 模块运行器注入的 fs 没有这个方法，于是每次写入都 500。
  "proper-lockfile", "graceful-fs",
],
```

### 坑三：一个动态 import 让 WebKit 报错

第三个坑是浏览器测试跑出来的。有一处 `await import("@/lib/tool-presets")`，生产构建下它会被拆出一个独立的 chunk；页面在它还没下载完的时候发生了导航，WebKit 直接把这次 import 判失败，报 `Importing a module script failed`。改成静态导入就好——那个模块本来就没有循环依赖的问题，动态 import 只是个多余的习惯。

> **方法论**
> 这三个坑有一个共同点：**只有把浏览器端到端测试跑起来才会发现**。我这次是先把 `bun test`、`tsc`、`eslint`、`vite build` 全跑绿，然后用 Playwright 的 WebKit 把 43 项检查在 dev 和 `vinext start` 两种模式下各跑一遍，第二个和第三个坑才浮出来。

## 04 实测对比

以下所有数字都在同一台机器（Apple Silicon macOS）上现测，用 `/usr/bin/time -p` 计时间。构建分「冷」（先删掉产物目录）和「热」（紧接着再构建一次）两种；运行时分 Bun 和 Node，因为发布走的是 Node。

### 构建时间

| 运行时 | 冷构建 Next | 冷构建 vinext | 热构建 Next | 热构建 vinext |
| --- | --- | --- | --- | --- |
| Bun 1.4.4 | 27.97 s | 4.81 s | 12.35 s | 4.05 s |
| Node 26.7 | 31.33 s | 6.00 s | 17.81 s | 5.16 s |

读出来就是：**Bun 下冷构建 27.97 秒 → 4.81 秒，快 5.8 倍；热构建 12.35 秒 → 4.05 秒，快 3.0 倍。**换成 Node 也是同一个量级（5.2 倍 / 3.5 倍）。

Next.js 16 使用 webpack 构建（本项目下 Turbopack 有兼容问题）。热构建的 Next 数字依赖 `.next/cache` 里约 695 MB 的持久化缓存；Vite 不留构建缓存，所以它的冷热两次几乎一样。

### 磁盘上的产物

| 目录 | 合计 | 其中构建缓存 | 真实产物 |
| --- | --- | --- | --- |
| `.next`（Next.js 16） | 727 MB | 695 MB（96%） | 32 MB |
| `dist`（vinext） | 19 MB | 0 | 19 MB |

去掉缓存后，真实产物是 32 MB → 19 MB，小 41%。服务端那部分从 22 MB 降到 10 MB。客户端静态资源两者接近（9.0 MB vs 8.8 MB）。

### 发布包

| 项目 | Next.js 16 | vinext |
| --- | --- | --- |
| npm tarball | 6.85 MB | 5.86 MB（−14%） |
| 解包后体积 | 32.2 MB | 18.7 MB（−42%） |
| 文件数 | 775 | 691（−11%） |
| 客户端 JS 分片 | 164 | 108（−34%） |

### 浏览器第一次打开，下载了多少

| | raw | gzip | 请求数 |
| --- | --- | --- | --- |
| Next.js 16 | 3722 KiB | 1087 KiB | 29 |
| vinext | 3767 KiB | 1100 KiB | 25 |
| 变化 | **+1.2%** | **+1.2%** | −14% |

两次测量各跑三遍，数字稳定。请求数 29 → 25，因为 Vite 把 chunk 合并得更少更大（一个 2.7 MB 的主分片，对比 Next 的 1.1 + 1.0 + 0.3 + 0.3…）；但总字节数没变。

> **为什么没变小**
> 因为首屏下载的是**同一份代码**：React、整个应用、依赖库。换打包器改变的是这些代码怎么被切分、怎么被压缩传输，不改变它们有多少。真正能让这个数字下降的是拆包、按需加载和删依赖——那是另一篇文章的事。

### 还有一项是变慢的

为了不把这篇写成软文，这一项也得放上来：生产服务器就绪的时间，Next 是 0.50 秒，vinext 是 1.27 秒，**慢了 2.5 倍**。日常是常驻进程，影响有限；但如果你在 CI 里频繁起停服务，或者拿它做 serverless，这一项要算进成本。

## 05 一张对照表

| 项目 | 迁移前（Next.js 16） | 迁移后（vinext） |
| --- | --- | --- |
| 构建命令 | `next build --webpack` | `vite build` |
| 开发命令 | `next dev` | `vite dev` |
| 生产服务 | `next start` | `vinext start` |
| 构建耗时（Bun / 冷） | 27.97 s | 4.81 s |
| 构建耗时（Bun / 热） | 12.35 s | 4.05 s |
| 产物目录 | `.next/`（727 MB，含 695 MB 缓存） | `dist/`（19 MB） |
| 开发状态目录 | `.next-dev/` | `.vinext/` |
| 配置文件 | `next.config.ts` | `next-config.ts` + `vite.config.ts` |
| 发布包（tarball） | 6.85 MB | 5.86 MB |
| 首屏下载（raw / gzip） | 3722 / 1087 KiB | 3767 / 1100 KiB |
| 生产服务器就绪 | 0.50 s | 1.27 s |

## 06 测量方法

免得这些数字不可复现，把方法写清楚。全部在同一台 Apple Silicon Mac 上完成，两次构建之间没有其他负载：

```bash
# 构建时间：冷 = 先删产物目录，热 = 紧接着再来一次
rm -rf dist .next
/usr/bin/time -p bun run build      # 或 npm run build（Node）

# 磁盘与发布包
du -sh .next dist
npm pack --dry-run --json           # size / unpackedSize / 文件数

# 首屏下载量：Playwright WebKit，累加同源 response body
page.on("response", async (r) => {
  if (new URL(r.url()).origin !== origin) return;
  bytes += (await r.body()).length;
});
```

运行时的版本：Bun `1.4.4-canary.1`（带上面提到的 Request 修复），Node `26.7.0`。每一项测量至少跑两遍，文章中写的是稳定值；Vite 的构建时间波动在 ±10% 以内，Next 的冷构建波动更小（它的耗时主要由 CPU 决定）。

## 07 所以，值得迁吗

如果你和我一样，是一个人在维护一个本机/小规模部署的 Next.js 应用，迁移动力主要来自开发体验：**每次构建从半分钟变成五秒，这个差别会改变你提交代码的节奏**。加上产物目录从 727 MB 变成 19 MB，clone 和 CI 都轻了一截。

但如果你的瓶颈是首屏性能、是 SEO、是边缘部署，那这次搬迁不会直接帮你——首屏字节数没变，服务器启动还变慢了。这些地方该做的优化（拆包、缓存策略、流式渲染）迁移前后一样要做。

至于风险：vinext 是个年轻的项目，兼容性是「覆盖主流用法」而不是「全部」。动手之前先跑一遍 `vinext check`，再确认你的测试套件能跑起来——我这次三个坑全是测试跑起来之后才发现的。对一个功能已经稳定的项目来说，先在有测试护航的分支上试一次，成本是一天。

项目在这边：[pi-web-bun](https://github.com/brynne8/pi-web-bun)（迁移在 `try/vinext` 分支），vinext 在 [cloudflare/vinext](https://github.com/cloudflare/vinext)。Bun 那个 bug 的修复进展在 [oven-sh/bun#46528](https://github.com/oven-sh/bun/pull/46528)。

---

文中所有数据由作者于 2026-10-25 在 Apple Silicon macOS 上实测。图表为纯手写 SVG/CSS 实现，无第三方图表库。
