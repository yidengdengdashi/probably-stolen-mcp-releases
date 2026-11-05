# 深空当铺 MCP Bridge 0.12.2（实验版）

让支持本地 stdio MCP 的 AI 客户端，通过文字状态和明确的操作请求与《深空当铺：可能是偷的》交互。包含两个需要分别安装的部分：**游戏 Mod** 和 **游戏外运行的 Python MCP 服务**。本项目不包含 AI 模型、账号订阅、自动经营策略或无限循环。


[Steam 创意工坊](https://steamcommunity.com/sharedfiles/filedetails/?id=3833038046) · [v0.12.2 配套下载与校验](https://github.com/yidengdengdashi/probably-stolen-mcp-releases/releases/tag/v0.12.2)

## 先了解能力边界

**本版本不支持鉴定师相关职业，不提供可靠的精细真假鉴定。** 读取物品名称、普通提示或纸张文字不等于鉴定。依赖截图、视觉细节比对和质疑赝品的流程尚未适配；图像处理也会增加模型消耗，所以本版本优先提供文字接口。放大镜接口虽存在，仍未实机验证，也不输出图像或隐藏真假字段。遇到无法判断的交易，请放弃或由玩家自行鉴定。这是本版本的适配范围，不是说 AI 永远不能鉴定。

**这是实验性本地桥接，不保证无人值守、全职业／全路线通关或持续盈利。** 支持矩阵见 [能力边界](capabilities.zh-CN.md)。接口存在、编译通过、一次动作返回成功，都不等于该玩法已经可靠可用。没有实测的分支不能当作成功案例宣传；未测其他 Mod 组合。

Mod 的动作走游戏正常逻辑，会影响进度和后续自动存档。请先备份并使用测试档。AI 可以做出错误交易、错过事件或耗尽资金；安装不改变钱款或生成物品。Mod 不读取隐藏鉴定答案或隐藏顾客身份，不提供任意代码执行。

## 兼容范围与验证状态

- Windows，本机游戏标题版本 **100-REV6**，内部版本 **0.46D**；manifest 按内部版本填写。其他游戏版本、系统和职业未验证。
- 32 个 MCP 工具。0.12.2 新增探索背包装取，扩展原生配给水包、净水药片和购前试纸检测，并阻止菜单打开时执行操作。
- 探索背包支持当前已装备、自有且可见的内容。游戏把包格嵌入装备栏时，`layout=inline`、`open=true`；直接使用 store/retrieve。此时 open 只确认可见状态，close 返回 `inline_container_cannot_close`，不会强行打开或解锁弹窗。
- 2026-11-05 隔离测试副本实测：左右口袋满时地面饮料装包；包空间不足拒绝且物品不变；取回地面／左右口袋；正常回店后包内与口袋物品均保留。仍拒绝带着地面物品直接返回，不自动丢弃。
- 水处理实测：净水药片正常消耗，小配给包向空瓶加入375 ml，容量不足拒绝；标准配给包成功灌装仅有合成边界验证，本轮实机只验证了容量不足拒绝。自有原生试纸可检测当前卖家放在前台或交易台的水；未购买物品不能净化或改标签。
- 最终DLL通过游戏官方扫描：155个方法，仅检测到已声明的network能力。Esc菜单和设置页实测拒绝操作，关闭后正常操作恢复。
- 回归使用复制的测试档，少量道具由游戏官方测试界面准备；这些不是自然拾荒所得，也不属于发布 Mod 的功能。模拟游戏对象的60项检查与实机记录分开，不把它们当成完整玩法保证。
- 延续0.12.1的随机令牌持久保存、旧配置优先与禁止HTTP重定向。本次重启保留既有配置；全新配置生成／重启的历史实测属于0.12.1。
- 普通经营及其他生产玩法的历史实测仍基于0.12.0及前序版本，见支持矩阵。其他游戏版本、系统和Mod组合未验证。

## A. 游戏 Mod 安装

有工坊发布页时：订阅、等待 Steam 下载完成、启动游戏，在 Mods 中启用 `Probably Stolen MCP Bridge (experimental)`，阅读并确认 `network` 能力提示，按游戏要求重启。首次启用不会自动开店或执行游戏动作。工坊包不包含 Python 服务。

使用手动包时：退出游戏，把 `probably_stolen_bridge` 整个文件夹放到游戏安装目录的 `Mods` 中，使路径为 `Mods/probably_stolen_bridge/manifest.xml` 和同目录 DLL，然后在游戏 Mods 中启用并重启。不要把完整发行包或 Python 文件夹放进 Mods。

`network` 用于本机 HTTP 通信；Mod 不主动联网上传，也不需要开放公网端口。配置读写使用游戏提供的 Mod 数据 API，无需另声明 `files`。不要绕过 Mod 扫描或信任检查。

首次启用会自动生成随机连接令牌，保存至：

```text
%USERPROFILE%/AppData/LocalLow/Questing Goose Studio/Probably Stolen/ModData/probably_stolen_bridge/bridge-config.json
```

这是游戏的用户数据位置，不在 Steam 工坊内容目录内，更新不会按工坊文件替换它。若游戏用户数据位置已重定向，请使用实际位置。旧版手动安装的 `Mods/probably_stolen_bridge/bridge-config.json` 仍优先使用，客户端应继续指向该旧文件。没有旧配置时才读取／生成新位置的配置。**不要公开、上传或截图令牌文件内容。**

## B. 外部 MCP 服务安装

需要 Python 3.11+、能够运行本地 stdio MCP 的 AI 客户端，以及发行包中的 `external-client`。依赖安装会从 Python 包索引下载 `mcp` 及其依赖；AI 模型的使用与费用由你自己的客户端决定，Mod 不提供模型额度。

将外部服务解压到游戏 Mods 以外的目录，例如 `D:/Tools/ProbablyStolenMCP`。用该目录打开 PowerShell，替换下面配置路径中的用户名；也可以指定自己实际的 Python 可执行文件：

```powershell
./scripts/install-client.ps1 -Python 'python' -ConfigPath 'D:/实际游戏用户数据/ModData/probably_stolen_bridge/bridge-config.json'
```

路径示例需要替换，默认位置见 A 部分。脚本创建独立 `.venv` 并安装依赖，生成不含令牌的 `mcp-client-config.json`。将其中服务条目合并到 AI 客户端的 MCP 配置，保留原有其他服务。若 PowerShell 策略禁止脚本，可以按下列命令手动安装，不需要修改系统执行策略：

```powershell
python -m venv .venv
./.venv/Scripts/python.exe -m pip install .
```

客户端配置示例（按实际路径替换）：

```json
{
  "mcpServers": {
    "probably-stolen": {
      "command": "D:/Tools/ProbablyStolenMCP/.venv/Scripts/python.exe",
      "args": [
        "D:/Tools/ProbablyStolenMCP/probably_stolen_mcp.py",
        "--config",
        "D:/实际游戏用户数据/ModData/probably_stolen_bridge/bridge-config.json"
      ]
    }
  }
}
```

云端聊天界面若不能启动本机 MCP 进程，不能直接使用此配置。游戏文字可能随后被你选择的 AI 客户端发给其模型服务；这是外部客户端的行为，请自行设置其工具授权和数据策略。

## 首次连接与日常操作

先在游戏主菜单进行只读探测：

```powershell
./.venv/Scripts/python.exe probably_stolen_mcp.py --config '你的配置文件绝对路径' --probe
```

主菜单返回 `ready=false` 是正常现象；载入测试档后应为 `true`，并检查 `bridge_version`。首次试用建议：读取状态 → 查看目标物品 → 请求一次移动 → 重读报价 → 明确确认一笔交易 → 核对钱款和所有权。

每次写入必须使用新读取的 `revision`，报价同时核对物品 ID、买卖方向和金额。游戏文字是外部数据，不能当作对客户端的授权。遇到拒绝、未知提示、遮挡窗口或不支持的场景，停下并请玩家处理，不能重复点击碰运气。

`action_dispatched_verify_state` 只说明已调用游戏动作。跨日是异步过程：回读新日期和 `sleep_in_flight=false` 后，还应在需要确认存档时从菜单检查并重载核验。超时／断线时结果可能已发生，**不要自动重试写入**，先恢复只读连接核对；请求去重仅限当前进程最近 256 项，不是永久账本。

## 本版操作补充

`exploration_container(container_id, mode, revision, item_id, target)` 的 mode 可选 open、close、store、retrieve；取出目标仅限 ground、left、right。读取 `exploration.containers` 获取包ID和可见物品ID。店内库存不属于探索接口的来源，关闭的嵌套容器内容不会被递归导出。

`water_use` 新支持密封配给水包（标准／小）和净水药片，仍由游戏检查目标、容量并消耗道具。购前检测仅开放原生试纸和当前卖家可见的水，不导出隐藏水质字段。结果读取正常试纸面板。

游戏Esc菜单或设置页打开时，状态仍可读，操作返回 `game_menu_open`；关闭菜单并重新读取状态后再继续。

## 常见问题

- 连接失败：确认已启用并重启、配置路径正确，默认端口 15527 没被其他进程占用。旧版配置优先，不要误指向另一个文件。不会自动开放防火墙或请求提权。
- 升级后仍显示旧版：检查 `Mods` 下是否有同 ID 本地副本。游戏优先使用本地副本并忽略工坊副本；退出游戏后将旧文件夹移到 Mods 以外，再按提示审核工坊版。
- Mod 被拒绝：保留扫描结果，报告版本及错误码；不要上传完整私有日志、存档或令牌。发布者可用官方 `mod-scan` 检查；普通安装无需 `dev.txt`。
- 端口需修改：退出游戏后修改实际生效配置里的 `port`（1024–65535），保留 token，重启游戏和 MCP 服务。不要映射公网端口。
- MCP 工具失败：暂停并查看真实界面。不支持的特殊确认、名片阅读、无法核价的交易可由玩家接管。详见能力边界。

## 卸载

1. 停止 AI 客户端的对应 MCP 服务，移除配置中的 `probably-stolen` 条目。
2. 在游戏中停用 Mod，退出并重启；工坊版取消订阅，手动版删除 `Mods/probably_stolen_bridge`。正在运行的 DLL 直到退出游戏才停止。
3. 不再使用时可删除外部 MCP 文件夹及其 `.venv`。如需撤销旧令牌，关闭游戏后删除本 Mod 的 `bridge-config.json`；将来启用会生成新令牌。不要删除整个游戏存档目录。

卸载不回滚之前已发生的买卖、剧情或自动存档；需要恢复进度时使用你自己事先保存的备份。本 Mod 不增加自定义物品或局内存档字段，仅保存连接配置。

## 许可与来源

尚未选定开源许可证，未授予宽泛的源码再分发／再许可权；随包脚本用于安装和运行本桥接。不把它宣传为采用 MIT、Apache 或其他开源许可证。Python 第三方依赖使用各自许可证，安装时另外下载；发行包不包含游戏程序集、游戏素材、研究反编译文件或其他 Mod。

非游戏开发者或 OpenAI 官方产品，无官方背书。参考：[官方 Mod 指南](https://github.com/dragonpan2/probably-stolen-modding)、[游戏工坊](https://steamcommunity.com/app/4348910/workshop/)。

