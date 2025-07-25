# PrivHunterAI   
一款通过被动代理方式，利用主流 AI（如 Kimi、DeepSeek、GPT 等）检测越权漏洞的工具。其核心检测功能依托相关 AI 引擎的开放 API 构建，支持 HTTPS 协议的数据传输与交互。

## 工作流程
<img src="./img/%E6%B5%81%E7%A8%8B%E5%9B%BE.png" width="800px" />

## Prompt
```json
{
  "role": "你是一个专注于HTTP语义分析的越权漏洞检测专家，负责通过对比HTTP数据包，精准检测潜在的越权行为，并给出基于请求性质、身份字段、响应差异的系统性分析结论。",
  "input_params": {
    "reqA": "原始请求对象（包括URL、方法和参数）",
    "responseA": "账号A发起请求的响应数据",
    "responseB": "将账号A凭证替换为账号B凭证后的响应数据",
    "statusB": "账号B请求的HTTP状态码（优先级排序：403 > 500 > 200）"
  },
  "analysis_flow": {
    "preprocessing": [
      "STEP 0. **请求类型识别（读/写）**：通过请求方法（GET/POST/PUT/DELETE）、URL特征、参数内容、请求体结构等判断请求是否为写操作，若包含典型身份字段（如 user_id/account_id），优先判断为写操作。",
      "STEP 1. **接口属性判断**：识别接口是否为公共接口（如验证码获取、公告类资源），结合路径命名、是否要求认证等进行判断。",
      "STEP 2. **动态字段过滤**：自动忽略影响判断的动态字段（如 timestamp、request_id、trace_id、nonce、session_id 等），支持后续通过配置扩展字段。",
      "STEP 3. **身份字段提取**：分析请求参数及Body中是否存在账号身份字段（如 user_id、account_id、email 等），用于辅助判断操作行为目标。"
    ],
    "core_logic": {
      "快速判定通道（优先级从高到低）": [
        "1. **越权行为（Result返回True）**：若为写操作，且 responseA 与 responseB 核心字段一致，均表示写入/修改成功（如 writeStatus = success），视为写操作型越权（true）。",
        "2. **越权行为（Result返回True）**：若 responseB 与 responseA 关键字段（如 data.id、user_id、account_number）完全一致（不包含动态字段），判断为读操作型越权（true）。",
        "3. **越权行为（Result返回True）**：若 responseB 与 responseA 完全一致，判断为越权行为（true）。",
        "4. **越权行为（Result返回True）**：若 responseB 中包含 responseA 的敏感字段（如 user_id、email、balance），但无账号B相关数据，判断为越权行为（true）。",
        "5. **非越权行为（Result返回false）**：若 responseB.status_code 为403或401，判断为无越权行为（false）。",
        "6. **非越权行为（Result返回false）**：若 responseB 为空（如 null、[]、{}），且 responseA 有数据，判断为非越权行为（false）。",
        "7. **非越权行为（Result返回false）**：若 responseB 与 responseA 在关键业务字段值或结构上显著不一致，判断为非越权行为（false）。",
        "8. **无法判断（Result返回Unknown）**：若不满足明确的越权或非越权标准，且字段相似度处于模糊区间，返回 unknown。",
        "9. **无法判断（Result返回Unknown）**：若 responseB 为500、乱码或格式异常时，返回 unknown。"
      ],
      "深度分析模式（快速通道未触发时执行）": {
        "字段值对比": [
          "a. **结构层级分析**：采用JSON Path对比字段层级结构和字段覆盖率，评估字段匹配相似度。",
          "b. **关键字段匹配**：识别如 user_id、order_id、account_number 等字段，分析命名、路径、值的一致性。"
        ],
        "语义分析": [
          "i. **数值型字段检查**：判断是否存在账户余额、积分、金额等关键字段泄露。",
          "ii. **格式与模式分析**：分析如手机号、邮箱、身份证等字段格式是否对应账号A。",
          "iii. **敏感字段泄露检测**：自动识别 password、token、email、phone 等字段，判定是否为账号A的数据。"
        ]
      }
    }
  },
  "decision_tree": {
    "true": [
      "1. 若写操作响应中，账号B执行后获得与账号A一致的 success 响应，判定为越权（res: true）。",
      "2. 若为读操作且 responseB 返回账号A的敏感数据，判定为越权（res: true）。",
      "3. 若 responseB 与 responseA 字段完全一致，未包含账号B自身信息，判定为越权（res: true）。",
      "4. 若关键字段（如 order_id、user_id、phone）结构和值完全一致，判定为越权（res: true）。"
    ],
    "false": [
      "1. responseB.status_code 为 403/401 → 非越权（res: false）。",
      "2. responseB 数据为空但 responseA 有内容 → 非越权（res: false）。",
      "3. responseB 与 responseA 在关键字段值或结构上差异显著 → 非越权（res: false）。",
      "4. 若接口为公共资源接口，无需鉴权 → 非越权（res: false）。"
    ],
    "unknown": [
      "1. 相似度处于中间地带（50%-80%），字段结构部分匹配 → 无法判断（res: unknown）。",
      "2. 响应为乱码、加密格式、异常格式 → 无法判断（res: unknown）。",
      "3. 无法判断操作目标是账号A还是账号B（如无身份字段） → 无法判断（res: unknown）。"
    ]
  },
  "output_spec": {
    "json": {
      "res": "结果为 true、false 或 unknown。",
      "reason": "提供详细的分析过程和判断依据。",
      "confidence": "结果的可信度（百分比,string类型,需要加百分号）。"
    }
  },
  "notes": [
    "1. 判断为越权时，res 返回 true；非越权时，返回 false；无法判断时，返回 unknown。",
    "2. 保持输出为 JSON 格式，不添加任何额外文本。",
    "3. 确保 JSON 格式正确，便于后续处理。",
    "4. 保持客观，仅基于请求及响应内容进行判断。",
    "5. 支持用户提供动态字段或解密方式，以提高分析准确性。",
    "6. 若请求方法无法识别为明确的写/读操作，默认保守处理为 unknown。"
  ],
  "advanced_config": {
    "similarity_threshold": {
      "structure": 0.8,
      "content": 0.7
    },
    "sensitive_fields": [
      "password",
      "token",
      "phone",
      "email",
      "user_id",
      "account_id",
      "id_card"
    ],
    "dynamic_fields_default": [
      "timestamp",
      "request_id",
      "trace_id",
      "nonce",
      "session_id"
    ],
    "auto_retry": {
      "when": "检测到加密数据、乱码、或格式异常时",
      "action": "建议用户提供解密方式后重新检测"
    }
  }
}
```

## 使用方法
1. 下载源代码 或 Releases；
2. 编辑根目录下的`config.json`文件，配置`AI`和对应的`apiKeys`（只需要配置一个即可）；（AI的值可配置qianwen、kimi、hunyuan、gpt、glm 或 deepseek） ；
3. 配置`headers2`（请求B对应的headers）；可按需配置`suffixes`、`allowedRespHeaders`（接口后缀白名单，如.js）；
4. 执行`go build`编译项目，并运行二进制文件（如果下载的是Releases可直接运行二进制文件）；
5. 首次启动程序后需安装证书以解析 HTTPS 流量，证书会在首次启动程序后自动生成，路径为 ~/.mitmproxy/mitmproxy-ca-cert.pem(Windows 路径为%USERPROFILE%\\.mitmproxy\mitmproxy-ca-cert.pem)。安装步骤可参考 Python mitmproxy 文档：[About Certificates](https://docs.mitmproxy.org/stable/concepts-certificates/)。
6. BurpSuite 挂下级代理 `127.0.0.1:9080`（端口可在`mitmproxy.go` 的`Addr:":9080",` 中配置）即可开始扫描；
7. 终端和web界面均可查看扫描结果，前端查看结果请访问`127.0.0.1:8222` 。

### 配置文件介绍（config.json）
| 字段             | 用途                                                                                   | 内容举例                                                                                                  |
|------------------|----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| `AI`             | 指定所使用的 AI 模型                                                                   | `qianwen`、`kimi`、`hunyuan` 、`gpt`、`glm` 或 `deepseek`                                                                                              |
| `apiKeys`        | 存储不同 AI 服务对应的 API 密钥 （填一个即可，与AI对应）                                                        | - `"kimi": "sk-xxxxxxx"`<br>- `"deepseek": "sk-yyyyyyy"`<br>- `"qianwen": "sk-zzzzzzz"`<br>- `"hunyuan": "sk-aaaaaaa"`                 |
| `headers2`       | 自定义请求B的 HTTP 请求头信息                                                           | - `"Cookie": "Cookie2"`<br>- `"User-Agent": "PrivHunterAI"`<br>- `"Custom-Header": "CustomValue"`    |
| `suffixes`       | 需要过滤的文件后缀名列表                                                     | `.js`、`.ico`、`.png`、`.jpg`、 `.jpeg`                                                |
| `allowedRespHeaders` | 需要过滤的 HTTP 响应头中的内容类型（`Content-Type`）                                       | `image/png`、`text/html`、`application/pdf`、`text/css`、`audio/mpeg`、`audio/wav`、`video/mp4`、`application/grpc`|
| `respBodyBWhiteList` | 鉴权关键字（如暂无查询权限、权限不足），用于初筛未越权的接口 | - `参数错误`<br>- `数据页数不正确`<br>- `文件不存在`<br>- `系统繁忙，请稍后再试`<br>- `请求参数格式不正确`<br>- `权限不足`<br>- `Token不可为空`<br>- `内部错误`|

## 输出效果
持续优化中，目前输出效果如下：

1. 终端输出：
<img src="./img/%E6%95%88%E6%9E%9C.png" width="800px" />

2. 前端输出（访问127.0.0.1:8222）：
<img src="./img/%E5%89%8D%E7%AB%AF%E7%BB%93%E6%9E%9C.png" width="800px" />

## 后续计划
1. 添加敏感信息的扫描，例如通过正则匹配+AI辅助识别的方式扫描js文件中泄露的秘钥；
2. 优化越权漏洞/未授权漏洞扫描流程，实现消耗更少的token、更准确的扫描。

## 更新时间线
- 2025.02.18
  1. ⭐️新增扫描失败重试机制，避免出现漏扫；
  2. ⭐️新增响应Content-Type白名单，静态文件不扫描；
  3. ⭐️新增限制每次扫描向AI请求的最大字节，避免因请求包过大导致扫描失败。
- 2025.02.25 - 02.27
  1. ⭐️新增对URL的分析（初步判断是否可能是无需数据鉴权的公共接口）；
  2. ⭐️新增前端结果展示功能。
  3. ⭐️新增针对请求B添加其他headers的功能（适配有些鉴权不在cookie中做的场景）。
- 2025.03.01
  1. 优化Prompt，降低误报率；
  2. 优化重试机制，重试会提示类似:`AI分析异常，重试中，异常原因： API returned 401: {"code":"InvalidApiKey","message":"Invalid API-key provided.","request_id":"xxxxx"}`，每10秒重试一次，重试5次失败后放弃重试（避免无限重试）。
- 2025.03.03
  1. 💰成本优化：在调用 AI 判断越权前，新增鉴权关键字（如 “暂无查询权限”“权限不足” 等）过滤环节，若匹配到关键字则直接输出未越权结果，节省 AI tokens 花销，提升资源利用效率。
- 2025.03.21
  1. ⭐️新增终端输出请求包记录。
- 2025.04.10
  1. ⭐️新增结果可信度（confidence）输出，并优化Prompt。
- 2025.04.22 - 04.23
  1. 优化前端样式，并引入分页查询功能，避免一次性加载全部数据，从而减轻浏览器的渲染压力，提升页面响应速度和用户体验。

# 注意
声明：仅用于技术交流，请勿用于非法用途。
