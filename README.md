# Claude 封号笔记

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Stars](https://img.shields.io/github/stars/ChinaGuCheng/Claude-ban-notes?style=social)
![Last Update](https://img.shields.io/github/last-commit/ChinaGuCheng/Claude-ban-notes)

最后更新：2026-11-03

在国内用 Claude、Claude Code，被封号是常有的事。这里把常见的封号情形、官方条款怎么写、几种用法各自的麻烦整理在一起。能查到出处的给出处，查不到的直接写「没有数据」。

[点击加入 GPT Claude 防封号交流群 1125726780](https://qm.qq.com/q/K2acaywxCG)

---

## 目录

- [先看这几条](#先看这几条)
- [哪些情况容易出事](#哪些情况容易出事)
- [新号的头几天](#新号的头几天)
- [几种用法比一比](#几种用法比一比)
- [自己搭：海外服务器跑 Claude Code](#自己搭海外服务器跑-claude-code)
- [常见问题](#常见问题)
- [参考资料](#参考资料)
- [参与和许可](#参与和许可)

---

## 先看这几条

- Anthropic 公布的支持地区里没有中国大陆。在国内用，本身就在条款外面。
- 账号只能自己用。拼车、转卖、把订阅转发给别人，官方条款写明禁止，封了不退款。
- 封号邮件不会告诉你是哪条触发的。下面列的「原因」大多是社区经验，官方没确认过。
- 没有哪种办法能保证不封。谁跟你说「永不封号」「封号包赔」，先当广告看。

---

## 哪些情况容易出事

| 情况 | 为什么会被盯上 | 能做的 |
|---|---|---|
| IP 经常变，或者出口在不支持的地区 | 登录地、付款地、使用地对不上，看着像盗号或倒卖 | 固定一条线路，登录和使用都走它 |
| 付款用虚拟卡、来路不明的卡 | 支付走 Stripe，卡被判风险高会拒付，事后撤销付款也会连累账号 | 用自己名下的卡，或在官方 App 里订阅 |
| 新号一注册就大量使用 | 社区普遍反映新号更敏感，官方没说过 | 头几天正常用，别一上来就跑大任务 |
| 多人合用一个号 | 同一账号在多个地方、多台设备同时活跃 | 一人一号 |
| 买成品号、找人代充 | 卡和号的来路你看不到，出事卖家通常不管 | 别买 |
| 用脚本批量调用订阅账号 | 条款只允许通过 API Key 做自动化调用 | 批量任务用官方 API |

几处补充：

**IP。** 数据中心 IP 和住宅 IP 哪个更安全，社区说法打架。有人说机房 IP 一上来就被封，也有人用家宽住宅 IP 照样被封。还没看到靠得住的结论，能确定的只有一条：出口别来回跳。

**付款。** 网上说 Wildcard、Bewildcard、Nobepay 这几家虚拟卡已经陆续停了，还能搜到的相关教程大多过时。

**成品号和代充。** 一两百块一个的「成品 Pro 号」，卡可能是盗刷的，几个月后银行追回付款，号也跟着没了。卖家说「封号包换」，换来的号往往是同一批货。

**自动化。** Consumer Terms 原文禁止用脚本、机器人之类的自动方式访问服务，通过 API Key 访问或官方另行允许的情况除外。想跑批量，就去开 API。

---

## 新号的头几天

不少人的号是在注册或付费后不久被封的，于是有了「观察期」的说法。官方从没公布过这个东西，多长、看什么，都没有数据。

稳妥点的做法跟上面差不多：邮箱用 Gmail、Outlook 这类常见的，别用临时邮箱；IP 别跳；先正常聊几天，再上重活。

---

## 几种用法比一比

| 用法 | 官方条款 | 主要问题 |
|---|---|---|
| 自己订阅，自己的固定网络 | 账号自己用，但人在不支持地区 | 卡难办，网络要稳 |
| 买成品号 / 代充 | 来路不明，常是转卖来的 | 被封后卖家常失联 |
| 拼车 / 合租 | 共享账号，明确禁止 | 一人出事，整车一起没 |
| 第三方 API 中转 | 看中转背后用的是什么号 | 按量计费偏贵，中转本身也会被封 |
| 海外服务器 + 自己的号 | 账号自己用 | 要会 SSH，自己维护 |

花费没法给统一数字，看你选哪档订阅、什么服务器和线路。Max 套餐官网标价每月 100 美元（5 倍用量）和 200 美元（20 倍用量），以官网为准，可能调整。

---

## 自己搭：海外服务器跑 Claude Code

思路很简单：Claude Code 只在一台海外服务器上跑，你在本地 SSH 过去用。Anthropic 那边看到的一直是这台机器的 IP，不会一会儿国内一会儿国外。

大致步骤：

1. 买一台海外 VPS，2 核 4G 够用，系统装 Ubuntu 24.04。
2. 装 Node.js 20，再装 Claude Code：`npm install -g @anthropic-ai/claude-code`
3. 在服务器上运行 `claude`，按提示登录你自己的账号。
4. 用 tmux 开会话，SSH 断了也不丢进度。

想把订阅账号变成一个 API 地址，给自己的几台设备用，可以看 [CPA 部署指南](docs/CPA部署指南.md)（只讲个人自用）。

麻烦的地方先说清楚：

- 订阅付款还是要一张能过的卡，服务器解决不了这个。
- 国内直连海外机器延迟高，到新加坡常见两三百毫秒，敲命令会卡。想快得自己配中转。
- 每月续费照样过一遍支付风控，卡出问题会员就掉。
- 服务器挂了、进程停了没人通知你，监控得自己写。

---

## 常见问题

### Q1. 号被封了能申诉吗？

能，官方有申诉表单。公开能查到的恢复案例很少，多数人收到的是模板拒信。

### Q2. 挂 VPN 一直被封，换住宅 IP 有用吗？

不好说。有换了住宅 IP 稳住的，也有住宅 IP 照样被封的。共享 VPN 的出口一大堆人在用，至少换成固定出口。

### Q3. 虚拟卡都停了，还能怎么付款？

用自己名下能付外币的卡，或者在官方 App 里走应用商店订阅（要对应地区的商店账号）。别找来路不明的代付。

### Q4. Max 5 倍和 20 倍怎么选？

数字是相对 Pro 的用量倍数。平时写代码、改脚本，5 倍一般够；整天开着 agent 跑长任务，再考虑 20 倍。

### Q5. 在国内用 Claude Code 违法吗？

Claude Code 是官方工具，拿它写代码没问题。麻烦在于官方支持地区不含中国大陆，从国内用违反服务条款。被封是违反条款的后果，跟国内法律是两码事。

### Q6. 封号会发邮件吗？

一般会，内容是通用模板，不写具体原因。

### Q7. 拼车为什么老出事？

一个号同时出现在好几个城市、好几台设备上，跟倒卖账号的样子一模一样。条款也明确禁止共享账号凭据。

### Q8. 用海外服务器，续费还会被风控吗？

会。续费时支付那边照样查卡。服务器只能让 IP 稳定，管不了卡。

### Q9. 一个号在手机和电脑上都登录行吗？

自己的几台设备登同一个号，是正常用法。几个人分着用，就成了共享账号，见 Q7。

---

## 参考资料

- [Anthropic Consumer Terms](https://www.anthropic.com/legal/consumer-terms)：账号共享、自动化访问的条款原文
- [Claude Code 法务与合规](https://code.claude.com/docs/en/legal-and-compliance)
- [PANews：2026 Claude 防封号与 KYC 避坑指南](https://www.panewslab.com/en/articles/019db34c-eb93-707b-9c25-dfe946dc11b1)
- [博客园：2026 Claude 订阅防封攻略](https://www.cnblogs.com/notlate-cn/p/20002343)
- [知乎：Claude 封号原因解析与防封策略](https://zhuanlan.zhihu.com/p/2019847407014293704)
- [腾讯云开发者社区：Claude 防封指南](https://cloud.tencent.com/developer/news/3854714)
- [GitHub · imarvinle/Claude-Pro](https://github.com/imarvinle/Claude-Pro)

社区文章是个人经验，有的夹带推广，看的时候自己分辨。

---

## 参与和许可

GPT、Codex 降智怎么测、怎么办，见 [GPT-downgrade-notes](https://github.com/ChinaGuCheng/GPT-downgrade-notes)。

被封过、发现哪里写错了、或者有新情况，欢迎开 [Issue](https://github.com/ChinaGuCheng/Claude-ban-notes/issues) 或提 PR，规则见 [CONTRIBUTING.md](CONTRIBUTING.md)。

本仓库最初整理自 [Fishason/Claude-anti-ban](https://github.com/Fishason/Claude-anti-ban)（MIT），2026-11-03 起全部重写，按 [MIT License](LICENSE) 开源。

本仓库与 Anthropic, PBC 无关。Claude、Anthropic 是 Anthropic, PBC 的商标。内容仅供参考，照着做出了问题自己负责。

[点击加入 GPT Claude 防封号交流群 1125726780](https://qm.qq.com/q/K2acaywxCG)
