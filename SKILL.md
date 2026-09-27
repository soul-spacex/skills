---
name: soulspacex
description: 用 SoulSpaceX 生成和编辑图片、视频，把创作过程落在可继续编辑的画布上。覆盖场景：生成（文生图、文生视频、图生视频、画一个xxx、做张海报、来段视频、做个封面）、编辑修改（把xxx换成yyy、去掉xxx、加上xxx、改成xxx、调整画面、换个角度）、风格转换（转绘、换风格、风格迁移）、成组创作（分镜、故事板、九宫格、短剧、MV、产品广告片、宣传片）、以及给已有画布加东西。当用户提到 SoulSpaceX、ssx、画布、节点时也应触发。关键判断：只要请求涉及 AI 图片或视频的创作、生成、编辑，无论怎么措辞（"画只猫""做个海报""这个视频帮我改改""按这个剧本出分镜"），都用这个技能。
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "🎬",
        "requires": { "bins": ["node"] }
      }
  }
---

# 在 SoulSpaceX 画布上创作

命令行工具是 `ssx`。它把创作过程落在一张**画布**上：文本、图片、视频是画布上的节点，节点之间连线，上游的产物自动成为下游的输入。用户事后能在网页上打开这张画布继续编辑——这是它和"调一个生图接口"的根本区别。

## 先确认能用

```bash
ssx whoami
```

- `command not found` → 没装，用 `npx @soulspacex/cli` 代替 `ssx` 跑，或 `npm i -g @soulspacex/cli` 装上。
- 提示未登录 → 让用户跑 `ssx login`（会打开浏览器授权）。**这一步必须用户自己做**，你代替不了。
- 提示需要会员 → 告诉用户命令行功能对订阅卡 / 会员 / 无限模式用户开放，让他去开通，不要反复重试。

## 标准流程

```bash
ssx workflow create "项目名"     # 建画布，自动绑定当前目录，后续命令不用再传 id
ssx model list                   # 看有哪些模型可用（带 id 和计价）
ssx schema                       # 看节点有哪些字段可以设

ssx node create "剧本" -t textNode --prompt "用户的原话" --model "模型名" --run
ssx node create "主视觉" -t imageNode --left 剧本 --model "MJ8.2" --run
ssx node list                    # 看状态
```

**选模型用 `--model "名字"`，不要自己填 `--set imageModelId=<数字>`。** 名字会按节点类型解析成对应的 id 并在输出里回显 `model: {id, name}`——你能确认生效的到底是哪个模型。名字写一半也认（`--model Seedance`），命中多个会把候选列给你挑。填错类型（给图片节点一个视频模型）会当场报错，而不是静默存下一个不生效的字段。

`--left` 接上游，值是上游节点的名字。文本上游会成为下游的提示词，图片上游会成为参考图或视频首帧。

节点类型：`textNode`（文本）、`imageNode`（图片）、`storyVideo`（视频）。这三种能 `--run`；`audioNode` / `agentNode` / `director3dNode` 能建但暂不能触发生成。

## 复杂任务：先搭结构，再生成

短剧、广告片、分镜这类多节点任务，**先把画布结构建完给用户确认，再触发生成**。

理由很实在：建结构不花钱，生成花钱。用户看过结构说"第三个镜头不对"，改一下再跑，比生成完九张图再返工便宜得多。

```bash
# 第一步：只建不跑（不加 --run）
ssx node create "分镜1" -t imageNode --left 剧本 --model "MJ8.2"
ssx node create "分镜2" -t imageNode --left 剧本 --model "MJ8.2"
ssx workflow show          # 把结构给用户看，附上画布链接

# 用户确认后再逐个跑
ssx node run 分镜1
ssx node run 分镜2
```

## 看清画布上有什么，再决定连谁

`ssx node list` 回的不只是状态，每个节点都带着产物 URL、生效的模型和入边：

```json
{ "id": "…", "type": "imageNode", "name": "参考图", "status": "completed",
  "modelId": 4, "modelName": "MJ8.2", "resultKind": "image",
  "url": "https://…/a.png", "inputs": ["剧本"] }
```

用户说「用画布上那张参考图」而没指名是哪个节点时，别猜：

```bash
ssx node list                              # 拿到每张图的 URL
ssx download <url> -o /tmp/ref1.png        # 下下来自己看一眼
ssx node update 主视觉 --left 参考图        # 确认是哪张之后再连
```

用户给的是本地图片时，先传上去再落成图片节点，它就和生成出来的图一样能连给下游：

```bash
ssx upload ./ref.png                           # 回 { assetId, url }
ssx node create 参考图 --asset <assetId>        # 落成已出图的 imageNode
ssx node update 主视觉 --left-add 参考图        # 连给下游
```

你自己生成或改过的图也一样走这条路，不要让用户手动把图拖进画布。连的是 SoulSpaceX 的 MCP 插件而不是 `ssx` 时，用 `create_upload_url` 拿上传地址，按它给的 curl 命令传，回来的 `data.id` 当 `asset_id`。

只把 URL 塞进某个节点的 `refImages` 也能当参考图，但画布上看不到这张图，也没法连给别的节点。用户想在画布上看到、复用这张图时用 `--asset`。已有的图片节点要换图：`ssx node update 参考图 --asset <assetId>`。

`inputs` 是这个节点当前的入边（上游节点名）。改连线前先看它，才知道该 `--left`（覆盖成这些）还是 `--left-add`（只加一条）。

**`modelId` 是当前设置，`modelName` 是上次生成时的快照。** 节点改过模型但还没重跑时两者会对不上，以 `modelId` 为准。

## 出错了先诊断，不要重建

```bash
ssx node list      # 哪个节点失败了、用的哪个模型、产物在哪
ssx workflow show  # 看它的连线和参数
```

失败原因通常是这几种，分别对应不同处理：

| 现象 | 怎么办 |
| --- | --- |
| 字段名或类型不对（报错会指名道姓） | 按报错改，`ssx schema` 查正确字段名 |
| 没选模型 | `ssx model list` 挑一个，`--model "名字"` |
| 积分不足 | 告诉用户去充值，不要重试 |
| 上游没有产物 | 先把上游节点跑出来 |
| 上游模型服务错误 | 可以重试一次，连续失败就换个模型 |

**不要因为一个节点失败就重建整张画布。** 用户在这张画布上的其他成果都还在。

## 存进素材库

画布记的是「这次怎么做的」，素材库存的是「这个东西以后还要用」。同一张主角定妆照，下次开新画布还想拿它当参考图——留在旧画布里得翻历史，存进素材库才能直接引用。

```bash
ssx asset list --category character   # 先看库里有没有现成的，省掉一次生成
ssx asset save 主视觉 --category scene --name 赛博街道
ssx upload ./ref.png --library --category character   # 本地文件直接入库
ssx space list                        # 有多个空间时才用，默认落个人空间
```

分类只有四个：`character` 人物、`scene` 场景、`item` 道具（不传就是它）、`voice` 音频。

**素材库不收视频**，分类里根本没有视频档，存 mp4 会被直接拒。视频留在画布里，用画布链接交付。

什么时候该存：用户说「这个角色以后还要用」「把这些存起来」，或者你在做短剧、分镜这类系列内容里会反复出现的元素。一次性的图不用存——画布里已经有了。

## 硬约束

这几条不遵守会直接失败或浪费用户的钱：

- **不要自己写轮询循环。** `--run` 和 `ssx node run` 默认阻塞到出结果。你每轮询一次都是一次模型调用，token 是用户在付。要立刻返回才用 `--wait 0`。
- **不要手填 `imageModelId` / `videoModelId` 这类数字 id。** 用 `--model "名字"`，让工具去查 id。手填的数字对不对你无从验证，错了也不会报错，只会让画布上跑的是另一个模型。
- **不要设置 `status` / `result` / `assetId` / `generationId`。** 这些由生成结果回写，你设了会被服务端拒。拼 `--set` 之前跑 `ssx schema` 看哪些字段能设。本地图要落成节点产物用 `--asset`。
- **不要替用户改写提示词。** 用户说什么就传什么，不要加"电影级光影、8K、超写实"这类词——服务端有自己的提示词处理，你加的词通常是负优化。
- **不要把一个任务拆成多次生成。** 要九张图就 `--set count=9`，不是跑九次。
- **结果走 stdout，进度走 stderr。** `ssx node list | jq` 是安全的。
- **`ssx upload` 默认不进素材库**，它只把文件放上对象存储给画布用。要用户在网页素材库里看得到，得加 `--library`。

## 计费

命令行创作按积分计费，**不使用无限模式额度**——买了无限模式的用户会问这件事，提前说清楚。每次生成完 `ssx` 会打印扣了多少积分。

`ssx balance` 看余额。

## 交付

任务完成时同时给用户两样东西：

1. **产物**——下载到本地的文件路径（`ssx download <url> -o 名字.png`），或者结果链接。
2. **画布链接**——每条命令的输出里都有 `url` 字段。用户点进去能看到整个创作过程，还能继续改。

只给一个图片链接的话，用户拿到的是一个孤立文件，之后没法修改也没法复用。画布才是他真正得到的东西。
