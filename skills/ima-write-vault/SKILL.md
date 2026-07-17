---
name: ima-write-vault
description: IMA 知识库写入与凭证排错专用 Skill。当需要通过 IMA OpenAPI 将笔记/文件写入知识库、或遇到 IMA
  写入类报错（尤其是 code 200002 skill auth failed、知识库字段名不匹配、库名找不到、 凭证
  BOM/换行导致鉴权失败）时使用。覆盖「创建笔记→查找知识库→关联入库→校验」标准四步流程， 并固化已知坑点（kb_id/kb_name 字段名、实际库名以
  search 返回为准、凭证落盘校验、OpenAPI 凭证 与 ima-mcp OAuth 两套独立凭证）。优先规避 shell 中文转义问题，改用
  Node require 调用 ima_api。
disable: true
---

# IMA 知识库写入与凭证排错

本 Skill 沉淀通过 IMA OpenAPI 写笔记/文件到知识库的标准流程，以及一组反复出现的鉴权与字段坑点。
触发场景：用户要求「把内容存到 ima 知识库」「创建笔记并关联到某库」「轮换 IMA 凭证后重连」，
或写入时返回 `200002` 等鉴权错误。

## 一、环境前置

- **凭证文件**（OpenAPI 体系，非 OAuth）：
  - `~/.config/ima/client_id`
  - `~/.config/ima/api_key`
  - 由 `ima-skills/ima_api.cjs` 在运行时自动读取，无需在调用命令里显式传参。
- **两套独立凭证（极易混淆，务必分清）**：
  - **OpenAPI clientId / apiKey**：用于 `ima_api.cjs` 程序化读写。失效仅影响 API 调用，不影响连接器。
  - **ima-mcp OAuth 授权**：WorkBuddy 连接器侧 `connected` 状态。轮换 OpenAPI 凭证**不会**改变它。
  - 结论：连接器显示 `connected` ≠ OpenAPI 凭证有效。

## 二、调用方式（规避中文转义）

不要把中文/多行 JSON 塞进 shell 命令参数（易因引号、换行、编码出错）。改用 Node 直接 `require` 模块：

```js
const ima = require('/绝对路径/ima-skills/ima_api.cjs');
// ima.<apiName>(bodyObject) 返回 Promise<响应JSON>
```

脚本模板见 `scripts/write_note_to_kb.js`，可直接复制改写。

## 三、标准四步写入流程

1. **创建笔记**：调用创建接口（如 `import_doc`）得到 `note_id`。若笔记已存在，复用旧 `note_id`，不要重复创建。
2. **查找目标知识库**：调用搜索接口（如 `search_knowledge_base`）按库名检索。
   - ⚠️ **字段名是 `kb_id` / `kb_name`**，不是文档示例里的 `id` / `name`。
   - ⚠️ **实际库名以 search 返回为准**：例如界面显示「【关注】」，真实 `kb_name` 可能是「关注」。**不要凭记忆硬编码库名**，先 search 取真实 `kb_id`。
3. **关联入库**：调用关联接口（如 `add_knowledge`），`media_type=11` 表示笔记（其他类型按 API 文档），传入 `kb_id` 与 `media_id`（即笔记的 `note_id` 派生 id）。
   - 校验返回 `ADD_CODE === 0`（或对应成功码）。
4. **校验**：在目标库内再次检索该笔记，确认 `media_id` 一致、`media_type` 正确，方为真正入库成功。

## 四、已知坑点与排错

### 坑 1：200002 skill auth failed
旧 OpenAPI 凭证失效。**第一步永远先核对本地文件是否真的更新了**：
```bash
cd ~/.config/ima && ls -la --time-style=+%H:%M:%S client_id api_key
```
- 若 `mtime`/字节数与上次失败一致 → 新密钥**未真正落盘**（见坑 3），不是服务端问题。
- 若已更新仍 200002 → 到 ima.qq.com/agent-interface 确认该 OpenAPI 应用处于「启用」且密钥对来自同一应用。

### 坑 2：库名/字段名凭记忆
search 返回 `kb_id`/`kb_name`；真实库名可能无方括号等装饰符。始终以 search 实时返回为准。

### 坑 3：凭证落盘失败（BOM / 换行 / 部分命令未执行）
- 用 `printf '%s' "值" > 文件`（不加换行）；或让助手用 Write 工具直写（无 BOM、无多余空白）。
- **切勿用 Windows 记事本「另存为 UTF-8」**——会写入 UTF-8 BOM（`\uFEFF`），使 `clientId` 变成 `\uFEFFxxx` 而鉴权失败。
- 多条 `printf` 命令可能只有部分成功（如第二条静默失败）。写入后**逐一**核对两个文件的 `mtime` 与字节数。

### 坑 4：重复创建笔记
关联失败时若已拿到 `note_id`，后续应复用而非重新 `import_doc`，避免库内出现重复笔记。

详细排错命令与判定逻辑见 `references/troubleshooting.md`。

## 五、轮换凭证后的重连（用户侧）
OpenAPI 凭证在 agent-interface 重置后，只需把新 `clientId`/`apiKey` 写入 `~/.config/ima/` 两个文件（见坑 3），脚本即可恢复。ima-mcp 连接器 OAuth 无需重连。

## 六、验证清单
- [ ] 两个凭证文件 mtime 已更新、字节数符合新密钥长度（无 BOM）
- [ ] search 取到真实 `kb_id`（非凭记忆）
- [ ] add_knowledge 返回成功码
- [ ] 入库后 search 校验 media_id / media_type 一致
- [ ] 本地临时脚本/中间文件已清理
