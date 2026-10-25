# dsh-rwdai-bundle

一个 DSH **bundle**：把 RWDai 的几个插件装在一起，并在插件页上分别开关。

## 它是什么

bundle 不是「功能代码包」，而是**装配层**——只有一份 `cordis.patch.yml`，负责往插件树里插入若干行。真正干活的仍是各自的独立插件包，各自带着 host 半与 `dsh.client` 浏览器半。

```
dsh-rwdai-bundle/           ← 本包：只有 patch，无功能代码
  cordis.patch.yml          ← insert: ollama-quota + reference + unread + input-history + edit-resend 各一行
  dependencies:
    dsh-ollama-quota        ← github:RWDai/dsh-ollama-quota
    dsh-reference           ← github:RWDai/dsh-reference
    dsh-unread              ← github:RWDai/dsh-unread
    dsh-input-history       ← github:RWDai/dsh-input-history
    dsh-edit-resend         ← github:RWDai/dsh-edit-resend
```

## 包含什么

| 插件 | 作用 |
|---|---|
| [dsh-ollama-quota](https://github.com/RWDai/dsh-ollama-quota) | 输入框上方的 Ollama Cloud 套餐额度面板（余额、百分比、token、到期日、用量节奏） |
| [dsh-reference](https://github.com/RWDai/dsh-reference) | 把对话选区与终端缓冲区一键引用进输入框（选区浮标 / 引用条 / 终端面板） |
| [dsh-unread](https://github.com/RWDai/dsh-unread) | 手动把会话标为未读（会话菜单开关 + 侧边栏行前圆点，状态存 host 侧、跨浏览器同步） |
| [dsh-input-history](https://github.com/RWDai/dsh-input-history) | 在输入框按 ↑/↓ 像终端一样翻出本会话发过的消息（↑ 取上一条，↓ 回退，改字退出并暂存草稿） |
| [dsh-edit-resend](https://github.com/RWDai/dsh-edit-resend) | 编辑历史用户消息并从该处重跑（fork 出事件前缀一致的子会话后立即重发编辑文本，prompt cache 可命中） |

## 为什么拆成五个包

`@deepseek-ai/dsh-client-modules` 有一条硬约束：**同一个包不允许占用两个 Loader 行**。

```js
if (sources.length > 1) {
  throw new Error(`client-modules: package ${packageName} resolves from multiple active Loader sources ...`)
}
```

所以「一个包里塞多个插件、各开一行」不可行。官方所有多组件 bundle（terminal、plugin-market、session-titles…）也都是「空壳 patch + 若干独立插件包依赖」的形态。

## 怎么选择哪一个生效

**用插件页的行级开关**，不是 bundle 配置。每行的 `disabled` 写回 profile 的 `cordis.patch.yml`，覆盖 bundle 层，重启后保持。

bundle 层本身没有「配置」这个概念——`plugins.bundle.config` 只是插件页的一个 UI 插槽，背后是 settings 命名空间，与 cordis patch 的 `config` 是两套东西。

## 安装

在 DSH Web 的插件页 → 添加插件：

```
github:RWDai/dsh-rwdai-bundle
```

或本地开发用 `link:/storage/heletian/dsh-rwdai-bundle`。

安装后，profile 的 `cordis.patch.yml` 里**不要再手工留**这几条 insert——同 id 重复插入会冲突（见下方注意事项）。

## 行 id

| id | 模块名 |
|---|---|
| `ollama-quota` | `dsh-ollama-quota` |
| `reference` | `dsh-reference` |
| `unread` | `dsh-unread` |
| `input-history` | `dsh-input-history` |
| `edit-resend` | `dsh-edit-resend` |

## 注意

`dsh-ollama-quota` 的额度面板只在自己的 ollama 供应商下显示，`config.providers` 默认 `[ollama]`；如果你的供应商名不同，直接在 bundle 的 `cordis.patch.yml` 里改，或在 profile patch 中用同一 id 覆盖：

```yaml
- id: ollama-quota
  config:
    providers:
      - 你的供应商名
```

`dsh-unread` 的标记存在 host 的 domain 存储里（domain `dsh_unread`，表 `marks`），不是 localStorage；浏览器半只读快照、发变更，所以换浏览器、换标签页、重启之后都是同一份。端点与行为见[插件自己的 README](https://github.com/RWDai/dsh-unread)。

`dsh-input-history` 完全跑在浏览器侧：历史来自当前会话的 chat 视图快照（`user` 与 `steering` 节点的 text 块），不落盘、不上报；↑/↓ 只在光标位于首/末行时接管，`/`、`@` 触发菜单、IME 组合、提交进行中一律让路。正在写一半的草稿会在进入浏览态前暂存，↓ 走过头时原样还回。

`dsh-edit-resend` 的「编辑重跑」= 精确前缀 fork（`session.fork` 的 `atSeq`）+ 直接向子会话 prompt 编辑文本：边界之前的模型可见历史与原会话逐字节一致，供应商侧 prompt cache 能原样命中；原会话不动，子会话带 `parentSessionId`、标题自动加序号。详见[插件自己的 README](https://github.com/RWDai/dsh-edit-resend)。
