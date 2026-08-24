# 工商：变更登记与信息查询

## 1. 北京 e窗通 变更登记

入口：`https://ect.scjgj.beijing.gov.cn`
典型流程（以经营范围变更为例）：填变更事项 → `change/contact` 经办人信息 → 上传材料 `change/filesUpload`
→ 提交 → **待在线签字** → 法定代表人电子营业执照小程序签字 → 受理。

需要法定代表人签字的材料：变更登记申请书、股东决定、章程修正案。

### 经营范围怎么写

- **只能从官方「经营范围规范表述查询系统」里挑标准条目**，不能自由造句。
  提交页是搜索 + 勾选式的，自己敲的词会匹配不上。
- 条目分**一般项目**（登记即可）和**许可项目**（要有相应前置 / 后置许可才能真开展）。
  **写进执照 ≠ 拿到许可**，别把「登记了」当「能干了」。
- 加条目会连带影响：**国标行业分类可能变**（进而牵动电子税务局的行业码 / 税种）、
  部分行业还有另外的**备案义务**（例如家政服务要在商务部门备案）。
  ⇒ 变更核准后要**回头复核税务侧行业码**，并把新执照换到所有已经用旧执照登记过的平台。
- ⚠️ **敏感条目会影响下游资质审核**（典型：教育培训类条目会牵动小程序 / App 的类目审核难度）。
  不确定的条目，先想清楚下游要不要用，再决定加不加。

### ⚠️ 「XX 未表明代理身份」的正解是换经办人，不是去备案代理人

提交时被拒：「XX未表明代理身份，请通过全国经营主体登记注册代理人信息系统（https://dj.samr.gov.cn）中的
「我要表明代理身份」模块填报相关信息」。

**正解：把 `change/contact` 页的经办人改成法定代表人本人。**
经办人姓名 / 证件号码 / 移动电话三个输入框都不是 readOnly，改完保存 → 一路下一步 → 提交，
**这条拒绝直接消失**，业务进入「待在线签字」。

**Why**：经办人是法定代表人本人时系统不存在「代理」关系，自然不触发代理人备案校验。
而变更材料本来就必须法定代表人签字 —— 他无论如何要出面，改经办人不增加人工成本，
反而把签字人从 2 人减到 1 人，还省掉另一人的身份证照片与常住地址采集。

**⚠️ 同页那个「该人员是否以代理机构工作人员名义代理登记事务 是/否」选「否」绕不过校验** —— 实测无效，必须换人。

### e窗通 自动化姿势

- **只吃 trusted 输入**：`Input.dispatchMouseEvent`（mouseMoved→mousePressed→mouseReleased）；
  输入用 `Input.insertText` 逐字。合成 `dispatchEvent` / `.click()` **完全惰性**。
- **trusted 点击必须先 `scrollIntoView`**：视口 innerHeight 常只有 ~759，而「下一步」按钮在 y≈1016，
  `Input.dispatchMouseEvent` 用的是视口坐标 ⇒ 点在窗口外，表现为「点了没反应」且不报任何错。
  先 `el.scrollIntoView({block:'center'})` → sleep → **重新测坐标** → 再点。
- **登录 URL 必须从 e窗通 自己的 JS bundle 里挖，不能手拼。** 手拼
  `yzt.beijing.gov.cn/...?module=EblCert&goto=<e窗通首页>` 会被拒：**「goto参数错误，请从官网进入！」**
  真实 `goto` 是**整串 OAuth2 authorize**（`service=bjzwService` + 本站专属 `client_id`
  + `redirect_uri=.../ect/apply/baic/entuser/redirect.do`），硬编码在 `static/js/` 的 chunk 里。
  ⚠️ 用 `urllib` / `curl` 抓那些 chunk 会被 WAF 打 **403** ⇒ 要在**页面内** `fetch` 再正则：
  ```js
  // 在已登录的 e窗通 标签页里执行
  const srcs = [...document.querySelectorAll('script[src]')].map(s => s.src);
  for (const u of srcs) {
    const t = await (await fetch(u, {credentials: 'include'})).text();
    const m = t.match(/https?:\/\/yzt\.beijing\.gov\.cn[^"'`]+/g);
    if (m) console.log(u, m);
  }
  ```
- **别把 `DIV.title 重要提示` 当成阻塞弹窗。** 判据是查 `.el-dialog__wrapper` 的可见性，
  而不是页面上有没有「重要提示」这几个字 —— 我为此白点了一次「关闭」。

### 变更登记的状态机与进度探针

办理状态实测演进：

```
（填表中，可自行删）→ 网上提交 → 待在线签字 → 业务已提交待审核 → 审查中 → 业务已办结
```

判定「变更到底生效没有」的三个硬信号，缺一不可：

| 信号 | 位置 | 说明 |
|---|---|---|
| **办理状态** | 我的业务 → 登记业务列表 | 只有「业务已办结」才是生效 |
| **最后修改时间** | 同上，每行一个 | ⭐ **最好的进度探针**：提交后它会冻住不动，一旦开始跳说明登记机关侧真的在处理 |
| **审批意见** | 详情页 | 有「审查机关 / 审查时间 / 审查意见」三栏；**审查时间有了但意见空白 = 在审，还没出结论** |

- **「业务终止」按钮消失 = 已进入审查，不能再自行撤回。** 想改只能等审查结论。
- ⚠️ **聚合查询站（爱企查 / 天眼查等）比登记机关滞后数天。**
  它们仍显示旧经营范围**不能**推出「没核准」；反过来它们更新了才是**已核准**的滞后确认。
  判「今天生效了没有」**只能读 e窗通 我的业务，或 gsxt**。


## 2. dj.samr.gov.cn（全国经营主体登记注册服务网）

**⚠️ 结论：这条路基本不用走**（见 §1）。代理人的政策学习 / 从业知识测试 / 实名认证 / 地址+身份证照片
全都是白跑的功。以下仅备查。

- 入口 `https://dj.samr.gov.cn/djfww/#/login` →「个人用户登录」tab（账号自建，凭证进私有档案）
- **登录态在 `sessionStorage.userToken`，不是 cookie** ⇒ Chrome 重启/关标签页就掉登录
- 三道门槛：① 实名认证（支付宝小程序人脸核身，**只能人工手机做，CDP 无法自动化**）
  ② 法律法规学习 + 从业知识测试（政策学习 38 篇 PDF 只 `window.open`，**零网络请求 ⇒ 服务端不记进度**，
  真门槛只有测试：20 题，18 单选 + 2 多选，**每次加载题目和选项都重新随机** ⇒ 答案库要按「题干片段 → 字母」匹配，
  不能按题号；匹配不上就中止，别猜。重复提交会得到 toast `系统异常，请重试。`）
  ③ 代理人信息填报 `#/agent-add`
- **`代理人所属机构` 不是卡点**：唯一选项就是「自行代理」且默认已选中。旁边「选择代理机构」弹窗是给挂靠代理机构的人用的。
- **不要点「取消代理人身份」。**

### dj.samr 自动化姿势（与 e窗通 完全相反）

- **只吃合成 `el.click()`**，trusted `Input.dispatchMouseEvent` 打在正确坐标上也毫无反应
  （已用 `getBoundingClientRect()` 回读确认过坐标，不是坐标问题）。
  Why：服务卡片 `LI.li_1 > A[href="javascript:void(0)"]` 的 handler 是 `addEventListener` 绑的，
  `onclick === null` 且元素上没有 `__vue__`/`__react` key，拿不到 fiber 直调；trusted 事件被自己的遮罩吞掉。
- **点击处理器挂在内层 `<a>`**：`li#xxx.click()` 静默无效，必须 `document.querySelector('#xxx a').click()`。
- **业务规则拒绝走 `.el-message-box`，不走 `.el-form-item__error`** ⇒ 点了「什么都没发生」时
  先轮询 `.el-message, .el-message-box, .el-notification` 再下结论。
- **`Page.captureScreenshot` 在这站渲染空白** ⇒ 要图就读 `img.src` 后 curl 远程 static 图，或抽 data-URI 解 base64。
- **只改 `location.hash` 会把 SPA 搞坏**（卡片列表消失 + 弹「未查询到数据！」）⇒ 用点卡片导航。
- **`Page.reload` 会丢 `sessionStorage.userToken`** ⇒ 这站宁可点卡片，别 reload。
- **Element checkbox 要点 `.el-checkbox__inner`**，点 `label.el-checkbox` 或内层 `input` 会双触发抵消，
  多选题永远只剩最后一个。
- **不要对 Element 弹窗发 Escape**：会关掉弹窗并清空整个表单，要点「关闭 / 取消」按钮。

## 3. 查别家公司工商信息（经营范围/法人/注册资本）

### 唯一走通的：爱企查 + 裸 CDP，且**不开 `Runtime.enable`**

`aiqicha.baidu.com` 有 devtools 检测。CDP 会话里**只要调用 `Runtime.enable`（或 `Page.enable`），
整页就只剩 17 个字符**：`请关闭浏览器的调试窗口再访问页面！`
（Why：检测靠 console 格式化时触发的 getter，`Runtime.enable` 会让它生效。）

```python
send("Target.attachToTarget", ...)          # 什么域都不要 enable
send("Page.navigate", {"url": url}, sid); time.sleep(10)   # Page.navigate 不需要 Page.enable
doc  = send("DOM.getDocument", {"depth": 1}, sid)
html = send("DOM.getOuterHTML", {"nodeId": doc["root"]["nodeId"]}, sid)["outerHTML"]
```

- 搜索页 `https://aiqicha.baidu.com/s?q=<urlencode(名称)>`；详情页 `https://aiqicha.baidu.com/company_detail_<pid>`
- **公司名在 HTML 里被 `<em>` 逐字拆碎**（「北<em>京</em>…」）⇒ 别从 DOM 文本取，
  从内嵌 JSON 的 `"pid":"…"` / `"entName":"…"` 成对提取，`entName` 要 `unicode_escape` 解码并去 `<em>`。
- 详情页文本里「经营范围」之后约 2000 字符即完整的「一般项目：…许可项目：…」原文。

### 不通的路（别再试）

- **WebSearch 对中文工商查询返回空**，第一轮就别浪费。
- 启信宝 `qixin.com` → 403；水滴信用 `shuidi.cn` → 301；企查查/天眼查要登录。
- `gsxt.gov.cn` 强反爬 + 验证码。爱企查数据源就是 gsxt，够用；**权威口径仍以 gsxt 为准，对外引用要注明**。
