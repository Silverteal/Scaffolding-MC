<!---

MIT License

Copyright (c) 2025 Burning_TNT (pangyl08@163.com)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

-->

# Scaffolding：Minecraft 联机客户端数据交换协议

## 1.目标

- 在 [EasyTier](https://easytier.cn/) 的基础上建立协议；
- 允许联机客户端相互交换和 Minecraft 联机的相关数据；
- 带有充足可扩展性：允许各联机客户端自定义扩展协议交换更多数据。

## 2.概念定义

- 联机玩家：实际使用联机客户端的自然人；
- 联机客户端：运行在联机玩家各自的电脑上，拼接参数并拉起 EasyTier 实例的应用程序；
- 联机中心：承担 Scaffolding 协议中心服务器职责的联机客户端，运行在与 Minecraft 服务器 相同的网络终端上；
- 联机房客：通过 EasyTier 网络加入多人游戏的联机玩家所使用的联机客户端；
- 联机房间：由联机中心和联机房客组成的 EasyTier 网络；
- 联机房间码：联机房间的唯一标识符，由联机中心生成，通过外部通信方式在联机玩家间传递。

# 3.重要字段定义

## 3.1 联机房间码

联机中心应在 Minecraft 服务器启动后，通过密码学安全的随机数生成器，创建符合 `U/NNNN-NNNN-SSSS-SSSS` 形式的联机房间码，满足以下约束：

- N 和 S 为任意大写字母（除去 I 和 O）和数字；
- 按照 0-9、A-H、J-N、P-Z顺序映射至 \[0, 33\] 后，依“小端序”读得的整型应能被7整除。

## 3.2 EasyTier 网络名称

若一个联机房间的房间码为`U/NNNN-NNNN-SSSS-SSSS`，则该联机房间的 EasyTier 网络名称应为 `scaffolding-mc-NNNN-NNNN`，网络密钥应为 `SSSS-SSSS`。
例如，输入联机房间码 `U/YNZE-U61D-2206-HXRG` 后，联机客户端应使用网络密钥 `2206-HXRG` 加入名称为 `scaffolding-mc-YNZE-U61D` 的 Easytier 网络。

## 3.3 联机中心 Hostname

联机中心的 Hostname (https://easytier.cn/guide/network/configurations.html#其他设置) 应为：
scaffolding-mc-server-{port: uint16}。
其中，port 必须是小于等于 65535 且大于 1024 的整数。例如：

- scaffolding-mc-server-33768：合法；
- scaffolding-mc-server-77844：不合法，77844 大于 65535；
- scaffolding-mc-server-n8：不合法，n8 不是整数。

## 3.4 联机客户端 machine_id

联机客户端应根据硬件信息生成 machine_id 字符串，作为玩家硬件设备的标识符。
machine_id 应足够长，不易在不同设备间发生碰撞，并对相同网络终端能产出稳定的结果。

# 4.通信定义

该协议的通信部分是建立于 TCP 协议之上的应用层协议，联机中心应在 EasyTier 上的虚拟 IP 地址和联机中心 Hostname 中 port 参数指示的端口开放 TCP 服务器，接受联机房客符合如下标准的请求（大端序）：

| 代号     | 偏移量        | 类型     | 解释                                                       |
|--------|------------|--------|----------------------------------------------------------|
| 动作类型长度 | 0          | uint8  | 字段*动作类型*的字节长度                                            |
| 动作类型   | 1          | byte[] | 此请求所表示的动作的动作类型名称，应符合 {namespace: ascii string}:{value: ascii string} |
| 请求体长度  | 1 + 动作类型长度 | uint32 | 字段*请求体*的字节长度                                             |
| 请求体    | 5 + 动作类型长度 | /      | 请求体，视每个*动作类型*的具体规范而定                                            |

其中，namespace 和 value 都应当是仅包含小写字母、下划线、数字的ASCII字符串。
联机中心应返回如下标准的请求：

| 代号    | 偏移量 | 类型     | 解释                  |
|-------|-----|--------|---------------------|
| 状态    | 0   | uint8  | 响应状态                |
| 响应体长度 | 1   | uint32 | 字段*响应体*的字节长度（可能为 0） |
| 响应体   | 5   | /      | 响应体（可能为空）           |

如果处理请求成功，状态应为 0，响应体将视*动作类型*而定；如果请求失败，则应返回以下状态：

| 状态       | 错误类型    | 响应体            | 响应体格式   |
|----------|---------|----------------|---------|
| [32, 64) | *动作类型*定义的错误 | 若未由*动作类型*明确指出，默认为空 | /       |
| 255      | 未知错误    | 错误详细内容         | UTF8字符串 |

## 4.1 动作类型

请求体和响应体的具体格式和语义由*动作类型*规定。动作类型分为基本动作、标准动作和扩展动作

- 基本动作：所有联机客户端都必须完整实现的最小动作类型集合，包括：`c:ping`, `c:supported_actions`,` c:server_port`, `c:heartbeat`, `c:player_profiles_list`；
- 标准动作：由 Scaffolding-MC 规范直接规定的动作类型，以 c (community) 作为命名空间的全部动作类型；
- 扩展动作：社区自发约定实现的其他动作类型。

所有基本动作都是标准动作。
扩展动作的制定者可递交 SEP 来将特定扩展动作添加为标准动作。

### 4.1.1 标准动作

#### c:ping [基本动作]

测试联机中心是否正常运行

- 请求体：任意内容
- 请求体格式：二进制内容，长度应小于 32 字节
- 响应体：与请求体内容一致
- 响应体格式：二进制内容，长度应与请求体一致

##### c:supported_actions [基本动作]

与联机中心协商支持的动作类型列表

- 请求体：联机房客支持的动作类型列表；
- 请求体格式：由 `\0` 分割的多个 ASCII String，如 c:supported_actions\0c:server_addresses\0c:player_name；
- 响应体：联机中心支持的动作类型列表；
- 响应体格式：由 `\0` 分割的多个 ASCII String，如 c:supported_actions\0c:server_addresses\0c:player_name。

#### c:server_port [基本动作]

获取 Minecraft 服务器的地址

- 请求体：空；
- 响应体：Minecraft服务器的在联机中心上的端口；
- 响应体格式：大端序编码的 2 字节 u16 数据，FF FF。
- 错误状态：
  * 32：服务器未启动

##### c:heartbeat [心跳/5秒] [基本动作]

- 请求体：玩家 ID 和联机客户端生成的 machine_id
- 请求体格式（JSON）：{ name: string, machine_id: string, vendor: string }
- 响应体：空

#### c:player_profiles_list [基本动作]

- 请求体：空
- 响应体：玩家列表（包括联机中心）
- 响应体格式（JSON）：[{ name: string, machine_id: string, vendor: string, kind: 'HOST' | 'GUEST' }]

### 4.1.2 扩展动作

各联机客户端可以自行约定或实现动作类型，选用自己的命名空间（不能使用c作为命名空间）。
请尽量选用联机客户端的名称作为命名空间，以避免命名空间冲突！

# 6.标准流程定义（未完成）

1. 联机客户端加入对应 EasyTier 网络；
2. 联机客户端遍历完整 peer 列表，查找合法的联机中心 Hostname。应该有且只有一个。

    - 如果网络中找不到合法的联机中心 Hostname，则这不是一个联机房间，联机客户端应提示玩家联机中心发现失败，并终止联机。
    - 网络中有多个合法的联机中心 Hostname 是协议未定义行为。

3. 立刻发送 `c:heartbeat` 心跳包（在动作类型协商之前）；
4. （如果客户端仅支持基本动作，可跳过此步）在联机信息交换协议上发送 `c:supported_actions`，获取联机中心支持的动作类型，并与当前联机客户端支持的动作类型取交集，确定本次联机可用的全部动作类型；
5. 在联机信息交换协议上发送 `c:server_port`，获取Minecraft服务器地址；
6. 每隔 5s 发送一次 `c:heartbeat` 心跳包。
