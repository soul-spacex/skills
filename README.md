# SoulSpaceX Skills

给 AI Agent 用的 SoulSpaceX 技能包。装上之后，跟 agent 说「画只猫」「按这个剧本出九宫格分镜」，它就会用 `ssx` 命令在 SoulSpaceX 的画布上完成，并把画布链接给你。

```bash
npx skills add soul-spacex/skills
```

装完让 agent 帮你登录：

> 帮我安装并配置 SoulSpaceX CLI，然后登录。

支持 Claude Code、Kimi Code CLI、Codex CLI、腾讯 WorkBuddy 等认 Agent Skills 规范的工具。

## 这个包里只有一份 SKILL.md

没有脚本。能力全在 `ssx` 命令里（npm 包 [@soulspacex/cli](https://www.npmjs.com/package/@soulspacex/cli)），SKILL.md 负责告诉 agent **什么时候该用它、怎么用才对**——比如复杂任务先搭画布结构再生成（建结构不花钱，生成花钱）、失败了先诊断别重建整张画布、不要自己写轮询循环烧 token。

## 谁能用

订阅卡、会员、无限模式用户。命令行创作按积分计费，不使用无限模式额度。
