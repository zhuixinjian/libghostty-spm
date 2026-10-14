# 交接：让 ccp 用 `foreground_pid` 精确认领 shell-tab ↔ session

> 本文是唯一事实源，取代旧版。图例：🔒 = **需你亲自执行/批准（禁止自动）**；✅ = 可自动。

## 为什么

ccp 里用户点 `+` 开 `.shell` tab、手敲 `claude`。点 sidebar 对应 session 时要导航回
那个 tab，否则会另起 `--resume` 把工作晾在原 tab。唯一稳的锚点是 **进程 pid**：
`claude agents --json` 给了每个 claude 的 `pid`，只要 surface 能返回前台进程 pid，就能
`entry.pid == surface.foregroundPid` 精确匹配（同 cwd 多 shell 也能区分，cwd 做不到）。

## 关键事实（已核实，别再推）

- ghostty C API：`uint64_t ghostty_surface_foreground_pid(ghostty_surface_t)`（返回
  `tcgetpgrp(pty)` 前台进程组，无进程时返回 **0**）；同 commit 还有
  `ghostty_string_s ghostty_surface_tty_name(ghostty_surface_t)`。实现在
  `src/apprt/embedded.zig`（`export fn`）。
- 引入 commit：**`9a9002202b8767e6e99c2bb48fad09fc0ae02870`**（author 2026-03-28 /
  committer 2026-04-20，"macos: add pid and tty properties"）。前一个 `4803d58` 没有。
- 该 API **不在任何 tag**。ghostty 最新 tag = v1.3.1；`v1.3.1...9a9002` = ahead 697 / behind 0
  （即候选 = v1.3.1 + 697 个 main commit，天然含 v1.3.1 全部）。
- Lakr233 最新 release = 1.2.9 / storage.1.2.9，基于 **v1.3.1** → 升级现成产物拿不到，
  **必须自建 xcframework**。

## 全局判断（决定执行顺序）

「同步到 1.2.9」和「拿到 pid API」是**正交**两件事：
- fork HEAD `d318b6ea9` 是 tag `1.2.9` 的**严格祖先**，6 个自定义 action-bridge case 已被
  Lakr233 按原样 SHA 并入 → 同步 = **纯 FF，零冲突，无需保留改动**。
- 但 1.2.9 的 xcframework 基于 v1.3.1，无 pid API → 仍要用 `build.sh --ref 9a9002…` 自建，
  并解一处补丁冲突。

顺序：**FF 到 1.2.9 拿干净基线 → 自建含 pid 的 xcframework → 落 Swift 访问器 → 本地 path
验证（含关键 pid 等式校验）→ 改 ccp 认领逻辑 → （对齐后）给 Lakr233 提 PR。**

---

## 执行计划

### Phase 0 — 预检（✅ 只读）
```bash
git -C /Users/chris/Projects/libghostty-spm fetch upstream --tags
git -C /Users/chris/Projects/libghostty-spm merge-base --is-ancestor d318b6e 1.2.9   # 期望 exit 0
git -C /Users/chris/Projects/libghostty-spm log HEAD --not 1.2.9                      # 期望空
```
两条都成立 → 「纯 FF」前提在，可继续；否则 fork 已分叉，停下重评估。

### (a) 同步 fork → 1.2.9 — 🔒
```bash
# 推荐：从 1.2.9 起干净新分支（不动老分支）
git -C /Users/chris/Projects/libghostty-spm checkout -b sync-1.2.9 1.2.9
# 备选：git checkout feat-extended-action-bridge && git merge --ff-only 1.2.9
```
验证：顶点 = `67506e1`（tag 1.2.9）；`Patches/ghostty/` 仍 0001–0007；`build.sh` 的 `--ref` 机制在。
binaryTarget 会从 `storage.1.0.1773686495`（checksum `ebd31d18…`）变成 `storage.1.2.9`
（checksum `0d5d4612…`）——但下一步我们要换成自建产物，这里不重要。

### (b) 自建含 pid 的 xcframework

**ref 选定：直接 `--ref 9a9002202b8767e6e99c2bb48fad09fc0ae02870`，不 cherry-pick**
（v1.3.1 已是其祖先，cherry-pick 只会丢 697 commit 还得单独解冲突）。

**b.1 解补丁冲突 — 🔒（唯一关卡：`0002-host-managed-io.patch`）**
补丁共 **7 个**（0001–0007）。对候选 ref 实测：0003 `git apply --check` 干净；0004/0006/0007
锚点在；0005 仅 iOS（`--platforms macos` 不触发）；0001 sed 区间锚点在（构建时看 marker）。
**只有 0002 冲突**，三处（含 `foreground_pid` 的 `embedded.zig` hunk 本身**不**冲突）：

1. `include/ghostty.h` — 仅行号漂移，加 fuzz / 改 hunk 头（琐碎）。
2. `src/Surface.zig` — 候选新增 `GHOSTTY_SURFACE_ID` env.put 块致上下文不匹配；把 exec-init
   的 `switch (rt_surface.termioBackend())` 手工重套，`SURFACE_ID` 块留在 `.exec` 臂内（~15-20 分钟）。
3. `src/termio/backend.zig` — **真关卡且直接耦合 pid**：0002 给 `Kind` 加 `host_managed` 后，
   候选新增的、支撑 `foreground_pid` 的 `getProcessInfo` switch 变非穷尽 → zig 编译错。补一臂：
   ```zig
   pub fn getProcessInfo(self: *Backend, comptime info: ProcessInfo) ?ProcessInfo.Type(info) {
       return switch (self.*) {
           .exec => |*exec| exec.getProcessInfo(info),
           .host_managed => null,   // host-managed 无本地进程
       };
   }
   ```
   `childExitedAbnormally` 那一 trivial hunk 也需按行漂移重贴。改完把 0002 重导出覆盖旧文件。

净工作量约半小时，无结构性障碍。

**b.2 构建 — 🔒（长时 zig 构建，本机 zig 0.15.2 已具备）**
```bash
cd /Users/chris/Projects/libghostty-spm
./Script/build.sh --ref 9a9002202b8767e6e99c2bb48fad09fc0ae02870 --platforms macos --skip-tests
# 产物: BinaryTarget/GhosttyKit.xcframework  +  build/GhosttyKit.xcframework.zip
```
编译期铁证：
```bash
grep ghostty_surface_foreground_pid BinaryTarget/GhosttyKit.xcframework/macos-*/Headers/ghostty.h
nm -gU BinaryTarget/GhosttyKit.xcframework/macos-*/libghostty.a 2>/dev/null | grep foreground_pid
```
> ⚠️ Darwin + `app-runtime=none` 默认不装 `libghostty.a`，靠 `0001` 强制 install。构建报
> "failed to locate built libghostty archive" 先查 0001 marker 有没有落。
> 全平台（macos,ios,maccatalyst）产物留到验证通过、且确认 0005 iOS 补丁可套后再出。

### (c) Swift 访问器 — 🔒（改 fork 源码）

**Diff 1 — `Sources/GhosttyTerminal/Surface/TerminalSurface.swift`（平台无关核心，必须）**
```diff
@@ public final class TerminalSurface {  (在 isMouseCaptured 后、// MARK: - Lifecycle 前)
+    // MARK: - Process
+
+    /// PID of the pty's foreground process group (`tcgetpgrp(pty)`).
+    /// Ghostty returns 0 when the surface has no process yet — surfaced as nil.
+    var foregroundPid: pid_t? {
+        guard let s = surface else { return nil }
+        let pid = ghostty_surface_foreground_pid(s)
+        return pid == 0 ? nil : pid_t(pid)
+    }
+
+    /// Controlling tty name (e.g. `/dev/ttys004`), nil until there is a process.
+    var ttyName: String? {
+        guard let s = surface else { return nil }
+        let str = ghostty_surface_tty_name(s)
+        defer { ghostty_string_free(str) }
+        guard let ptr = str.ptr, str.len > 0 else { return nil }
+        return String(decoding: UnsafeRawBufferPointer(start: ptr, count: Int(str.len)), as: UTF8.self)
+    }
```
按 `str.len` 解码（不依赖 null 终止符）；`defer` 保证任何路径都 free。无新增 import
（已 `import Foundation` + `GhosttyKit`，`pid_t` 来自 Foundation）。

**Diff 2（PR 交付推荐）— 新建 `Sources/GhosttyTerminal/Platform/Shared/TerminalView+Process.swift`（跨端）**
```swift
import Foundation

public extension TerminalView {
    var foregroundPid: pid_t? { surface?.foregroundPid }
    var ttyName: String? { surface?.ttyName }
}
```
`TerminalView` typealias 在各平台解析到 `AppTerminalView`/`UITerminalView`，两者都有内部
`var surface`，故 macOS/iOS/Catalyst 都编得过。ccp 仅 macOS，用 AppKit-only 变体（往
`AppTerminalView+PublicInput.swift` 的 public extension 里加同名两个 forward）也够。

### (d) 本地 path 验证环（✅ 改动可自动，依赖 b/c 的 🔒 产物）

- fork `Package.swift`：binaryTarget 临时改本地
  `.binaryTarget(name: "libghostty", path: "BinaryTarget/GhosttyKit.xcframework")`
- ccp `Package.swift`：`.package(url: "…zhuixinjian/libghostty-spm…", revision: "d318b6ea9…")`
  → 临时换 `.package(path: "/Users/chris/Projects/libghostty-spm")`

**关键验证门（决定 (e) 能不能上）**：ccp 开 shell tab 跑 claude，log 出的
`foregroundPid` 应 **==** `claude agents --json` 该 session 的 `pid`；shell 空闲时是 zsh pid。
**这条不成立就别改 (e)**——见风险表「pid vs pgid」。

正式发布链路（🔒）：`build.sh --download-url <U>` 生成带 url+checksum 的 Package.swift →
**手工建 `storage.*` release 传 zip**（`tag-release.sh` 不生成 storage.* tag）→ ccp revision
指向新 fork commit。

### (e) ccp 侧 pid 匹配 — ✅（改 ccp 业务代码，我可代劳）

`TerminalSessionManager.openSession`（`mac/MarkdownPreview/Services/Terminal/`，当前 cwd 块
约 283–306 行）换成 pid 精确匹配，pid 全局唯一 → 消除同目录多 session 歧义，去掉 count==1 妥协：
```diff
-        if let entry = lastAgentSnapshot[sessionUUID] {
-            let entryCwd = URL(fileURLWithPath: entry.cwd, isDirectory: true).resolvingSymlinksInPath().path
-            let shellCandidates = sameTarget.filter { tab in
-                tab.effectiveSessionUUID == nil && tab.session.isShellStart
-                    && tab.session.currentPwd.resolvingSymlinksInPath().path == entryCwd
-            }
-            if shellCandidates.count == 1, let shellHit = shellCandidates.first {
+        if let entry = lastAgentSnapshot[sessionUUID] {
+            let shellHit = sameTarget.first { tab in
+                tab.effectiveSessionUUID == nil && tab.session.isShellStart
+                    && tab.session.terminalView.foregroundPid.map(Int.init) == entry.pid
+            }
+            if let shellHit {
                 shellHit.adopt(sessionUUID: sessionUUID, label: sessionLabel)
                 shellHit.session.anchorStartedAt = entry.startedAt
                 onTabsChanged?()
                 shellHit.session.startIfNeeded()
                 activeTabByTarget[target.sessionKey] = shellHit.id
                 summary?.acknowledgeTab(tabId: shellHit.id)
                 recordActiveSelection(for: shellHit)
                 return shellHit
             }
         }
```
不匹配自然落到既有 `.resume(sessionUUID)` 分支。`entry.pid: Int`，`foregroundPid: pid_t?(Int32?)`。
需给 `TerminalSession` 暴露 `terminalView`（或加 `var foregroundPid`）。

---

## 🔒 需你把关的动作
1. 长时 xcframework 构建 `./Script/build.sh …`（b.2）
2. 对 fork 的任何写：checkout/merge/reset（a）
3. 改 fork 补丁 `0002-*.patch` 并重导出（b.1）
4. 改 fork 源码加 Swift 访问器 / Shared 文件（c）
5. 改 fork `Package.swift`（b/d）
6. 任何 `git push` / rebase（a/b/c）
7. 建 `storage.*` release、传 zip（d 正式发布）
8. **给 Lakr233 提 PR（提交前先同步你）**

ccp 侧本地改动（d 的 path 切换、e 的认领逻辑）不在把关内，可自动。

---

## 风险
| 风险 | 说明 | 缓解 |
|---|---|---|
| **0002 补丁冲突（backend.zig）** | 唯一结构性关卡，`getProcessInfo` switch 非穷尽 | 补 `.host_managed => null,`；重导出 patch |
| **pid vs pgid** | `entry.pid` 是 claude 进程 pid；`foregroundPid` 是 `tcgetpgrp` 前台进程组 leader。若 claude 被 node 包一层 / 非前台 pgroup leader，二者可能不等 → 认领失效退 resume。**(e) 正确性核心假设** | **(d) 先实测等式成立再改 (e)**；不成立则用 pgid 或保留 cwd 作二级 fallback |
| 0001 sed marker 未落 | 不落则不生成 `libghostty.a` | 构建失败优先查 |
| ref 漂移 | pin 到含 API 最早 commit；upstream 若改 export 签名头会漂 | 发布前 `nm`/`grep` 确认符号在 |
| 0005 iOS 补丁未逐一验 | 全平台产物前未确认 | macos-only 阶段不触发；出全平台前 grep 锚点 |

---

## Lakr233 PR 草稿（🔒 提交前先同步你，原样）

**Title:** `Expose foreground process PID and tty name on TerminalSurface / TerminalView`

**Body:**
```markdown
## What
Adds read-only access to the pty's foreground process:
- `TerminalSurface.foregroundPid: pid_t?` — wraps `ghostty_surface_foreground_pid`
  (`tcgetpgrp(pty)` semantics), mapping the C `0` "no process yet" sentinel to `nil`.
- `TerminalSurface.ttyName: String?` — wraps `ghostty_surface_tty_name`, reading the
  returned `ghostty_string_s` by length and freeing it with `ghostty_string_free`.
- Public accessors of the same names on `TerminalView` (AppKit `AppTerminalView` and
  UIKit `UITerminalView`, via a shared `Platform/Shared/TerminalView+Process.swift`
  extending the `TerminalView` typealias).

No new imports: `TerminalSurface` already imports `Foundation` + `GhosttyKit`.

## Why
Embedding hosts need to correlate a terminal surface with an external process list
(e.g. matching a running CLI in the pty to its entry in that tool's `--json` output).
The foreground pid is the only stable anchor — cwd cannot disambiguate two shells in the
same directory. `ttyName` is a secondary cross-check.

## Build dependency ⚠️
`ghostty_surface_foreground_pid` / `ghostty_surface_tty_name` were added upstream in
ghostty commit `9a9002202b87` ("macos: add pid and tty properties…", 2026-03-28) and are
**not** in any tagged release (latest is v1.3.1, which the current prebuilt xcframework is
based on). This Swift code won't compile until the bundled xcframework is rebuilt from a
ghostty ref that includes that commit.

**Request:** bump the ghostty build ref used by `Script/build.sh` to `9a9002202b` (or a
later `main` descendant keeping these exports) and republish the xcframework. Both symbols
are `export fn` in `src/apprt/embedded.zig` and return 0/empty when there is no process,
so they degrade safely.

## Scope
Purely additive read accessors; no behavior change. Core logic on the platform-agnostic
`TerminalSurface`; view-level accessors are one-line forwards.
```
