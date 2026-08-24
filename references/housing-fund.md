# 住房公积金（北京 · 单位侧）

平台：**单位网上业务系统** `https://dwwsyw.gjj.beijing.gov.cn/`
登录页 `/ish/gjjloginstyle/tysfgjj.jsp`，按钮 `goTysfpt()` 跳北京统一身份认证 `yzt.beijing.gov.cn`
→ **法人登录 → 电子营业执照**小程序扫码（和社保 yltc 同一套 yzt，公积金中心的 `client_id` 与社保不同）。
企业给办事人授权时，授权事项选「电子政务-一体化政务服务平台-政务服务网」。

首页 `/ish/home` · 全部功能只有 7 个：

| 功能 | 菜单码 |
|---|---|
| 增减员（含确认申报、缴款入口） | `UPLGJJ0202` |
| 单位信息变更（**比例、委托收款都在这里**） | `UPLGJJ0102` |
| 个人缴存基数修改 | `UPLGJJ0802` |
| 企业缴存报告 | `UPLGJJ0605` |
| 委托收款缴款 | `UPLGJJ0201` |
| 补缴 | `UPLGJJ0203` |
| 常用表格下载 | `PPLGG2001` |

服务记录（查自己办过什么）`/ish/flow/menu/SPFWJL0002` —— 要先填日期范围再点 `b_query1`。

## 0. 单位户可能早就开好了

**e窗通 企业开办时会同步完成公积金单位登记开户**，开户日期 = 公司成立日。
此时首页会显示：已开户 + 职工 0 人 + 缴存比例 **0%** + 缴至年月 `--` + 托收日 `-`。

⇒ 看着像「完全没办」，但**登记义务已经履行，不存在逾期**。
真实缺口只有三项：**职工个人账户未设立（增员）· 缴存比例未设 · 从未缴存 / 无托收**。

⚠️ 不要拿另一个平台的记录反推这里的状态。新社保平台的三合一登记（社保 + 就业 + 劳动用工备案）
**没有公积金字段**，社保办完不代表公积金顺带办了 —— 公积金是 e窗通 那次带的。

### 开户四条途径（官方《住房公积金单位登记开户操作详解》）

1. **e窗通**（企业开办时同步）← 绝大多数新企业实际走的就是这条
2. 公积金网上业务系统
3. 柜台（《单位柜台办理住房公积金登记开户申请表》公积金表 202，公章 + 法人签字）
4. 老社保平台 `rsj.beijing.gov.cn/csibiz/` 单位预登记，连带录 15 项公积金信息

事项「1.1.3 住房公积金单位登记开户」是**即办件**，法定办结时限 **0 工作日**，到现场 **0 次**。
⚠️ 但网上办完**仍有一份纸质件**：《单位网上办理住房公积金登记开户申请表》一式两份，公章 + 法人签字。

## 1. 🔴 依赖链：委托收款 ⇒ 比例 ⇒ 增员 ⇒ 确认申报 ⇒ 缴款

一条直线，**绕不过去**：

1. 增减员页点 `btn_hjzjy_add` ⇒ 报「单位住房公积金缴存比例应在 5%-12% 中选择，请予以变更」
2. **比例没有独立菜单**，只能走「单位信息变更」`UPLGJJ0102`
3. 该表提交 ⇒ 报「您单位信息有缺项，请补充红框缺项信息后，再次点击【提交】!」
   红框字段实测正是**委托收款整块**：`wtskdwzhmc wtskdwzhhm zfxth wtskr myhjxyqr zhlb zt`

⚠️ 源码里 `var valid_ids = "#c_gjjzhxx,#ct_form1" + (show_wtsk_form ? ",#wtsk_form" : "")`
且 `var show_wtsk_form = true;` **写死** ⇒ **委托收款不可跳过**。
`checkWtsk()` 里虽有「全空或全填都算通过」的逻辑（`return !(c_nu && c_not)`），
但它是闭包且在 `formValidate` **之后**才跑，页面加载时 `required` 已经挂上了 ⇒ 实际拦死。
（能用 JS 抹掉 `required` 绕过，但这是政务实名申报，不要这么干。）

⇒ **结论：拿到对公户四要素（开户行 / 账号 / 户名 / 12 位联行号）之前，公积金一步都推不动。**
排期时把「开对公户」放在公积金前面。

## 2. 参数怎么定

| 项 | 怎么定 |
|---|---|
| 单位 / 个人缴存比例 | **5%**（详见下） |
| 缴存基数 | 与社保基数、劳动合同月工资取同一个数（三数对齐） |
| 月缴存额 | 基数 × 5% × 2，**北京按元取整**（不是四舍五入到分） |
| 首次汇缴年月 | 改成**实际起薪那个月**，不要留系统默认的开户月 |
| 托收日 | `wtskr` 是 input，>28 会提示「当月无此日则按当月最后一日」⇒ 建议 ≤28 |

### 比例选 5% 是真省钱，和社保逻辑不同

**公积金比例可在 5%~12% 自选，没有社保那种「低于下限按下限核定」的地板。**
且事项「1.1.5 单位申请住房公积金降低缴存比例、缓缴」意味着：
**低于 5% 或缓缴要走审批，5% 是免审批地板；调高随时可以，调低要审批。**

⚠️ 基数改动在**一个公积金年度内线上只能改一次**（首页【个人缴存基数修改】）。

### 首次汇缴年月一定要改

进增减员页**必弹**：「您单位首次汇缴年月为 YYYY-MM，是否要变更首次汇缴年月？
点击【是】将首次汇缴年月变更为当前月(YYYY-MM)；点击【否】不变更」

**点【是】。** 按钮是 `#confirm_modal` 内 `button.btn.btn-primary`，「是」的 `value="0"`。

**Why**：留着开户月（= 公司成立月）会被要求补缴那几个月，而那几个月**无职工、无工资、个税是零申报**，
补缴等于自己造出一段与社保 / 个税口径打架的记录。

## 3. 单位信息变更表单机制（59 字段）

- **全字段默认只读，每个字段一个独立【变更】按钮，id = `<字段id>_bg`**（如 `dwjcbl_bg`）
- **单值型**弹框：`xxx` = 原值(只读) / `yyy` = 新值，确认按钮 **`b_bg`**
- **下拉型**弹框：`aaa` = 原值(只读) / `bbb` = 新值(select)，确认按钮 **`b_bgselect`**
- **option value 是小数**：5% = `0.05`、12% = `0.12`
- ✅ **只需设单位比例 `dwjcbl`，个人比例 `grjcbl` 会自动跟着变成同值**
  （没有 `grjcbl_bg`，个人比例不能单独设 ⇒「单位 5% + 个人 5%」天然满足）
- 联系人：点 `select_lxrxx` 弹「选择一个经办人」，选中 radio + 确认 ⇒ 自动回填
  `lxr_xingming / lxr_zjlx / lxr_zjhm / lxr_sjhm`
- 校验硬规则（源码）：企业性质 ⇒ 资金来源只能「单位自筹」；统一社会信用代码为空要去柜台补录

### 委托收款字段取值

| 字段 | 含义 | 取值 |
|---|---|---|
| `zhlb` | 账户类别 | 对公 = `2` |
| `zt` | 状态 | 启用 = `10` |
| `myhjxyqr` | 每月汇缴需要确认 | 是 = `10` |
| `wtskr` | 托收日 | input，>28 有提示 |
| `wtskdwkhh` | 开户行 | **总行级** select（工行 102000 / 农行 103000 / 中行 104000 / 建行 105000 / 招行 308000 …） |
| `zfxth` | 联行号 | 12 位，**网点靠它定位**，不在开户行 select 里选 |

填完要点 **`btn_wtskzhxxjy`【校验有效性】**验账户真实性。
`valid_yhzhxx`：若 `#btn_wtskzhxxjy` 已有 class `isvalided` 就跳过重校验；
改动 `ZHJY_ITEMS`（`wtskdwkhh / wtskdwzhhm / wtskdwzhmc / zfxth`）任一项都会剥掉 `isvalided`。

告知单口径：**户名与单位名称一致时无需去银行柜台**（结果页那句「请您尽快到单位的开户银行办理相关手续」
是通用文案）。多数国有 / 股份行在数据共享银行名单内。

## 4. 🔴 提交按钮「像死的」= `form_validate` 拒绝，不是点击 no-op

这一条花了 6 个脚本才定位，是本平台**最贵的坑**。

`单位信息变更` 的提交按钮**有两层**：
- 可见的 `b_flow_A` 是页面级包装，跑完业务预检后调 `$("#b_flow_a").click()`
- 隐藏的 `b_flow_a` 带 `class="submit-button" data-apply="0" data-validate="true"`，
  由 `#page_flow_buttons` 上的 `.submit-button` 委托监听处理，里面跑
  `ydl.formValidate(pageTabs[i].dom)` + `pageTabs[i].form_validate()` 再 `doSubmit(...)`

**根因**：`pageTabs[0].form_validate` 返回 `false` —— 源码里

```js
// 强制阅读委托收款告知书
if (notNullCount == $(WTSK_CHECK_IDS).length) {
  if (changeCount > 0 && $("#agree-box").prop("checked") === false) {
    ydl.toast('...请勾选已阅读委托收款告知单！'); return false; }}
```

它用 **`ydl.toast`（瞬时浮层）报错，不是 `.modal`** ⇒ 所有轮询 `.modal` 的循环都读到「无弹窗」，
被误判成「点击是 no-op」。

**修法**：点 `#open-xys` 打开告知单 → 关掉（按钮 `sub_close` 文案「我知道了」）→
`document.getElementById('agree-box').click()` → 确认
`pageTabs[0].form_validate.call(jQuery('#b_flow_a')) === true`（顺带把 `#wtskxxbgflag` 置 `"true"`）
→ 再点 `b_flow_A`。**缴款页 `UPLGJJ0105` 同样有 `agree-box`，同样要勾。**

> ⚠️ **通用教训：这个平台上「提交按钮像死的」，第一嫌疑是 `form_validate` 拒绝。
> 去读 `pageTabs[i].form_validate.toString()`，而不是继续轮询 `.modal`。**

### 排障姿势备查

- 页面内联脚本是在函数里 `eval` 的 ⇒ 它的 `var`（`show_wtsk_form` / `valid_yhzhxx` / `WTSK_CHECK_IDS`
  / `ZHJY_ITEMS`）**不在 `window` 上**（`'show_wtsk_form' in window === false`）。
  探测必须用 `typeof` 兜，否则 `ReferenceError` 把整个脚本打断。
  **别据此误判「内联脚本没执行」。**
- 有效手段：`jQuery._data(el,'events')` dump handler 源码；沿祖先链 + `document` / `window` 找委托监听；
  CDP `DOMDebugger.getEventListeners`；monkeypatch `ydl.formValidate` / `ydl.alert` / `jQuery.ajax` 打点；
  直接 `h.handler.call(el, jQuery.Event('click', {...}))` 调委托 handler。

## 5. ⚠️ 委托收款缴款提交后「看起来没生效」是正常的

提交成功后你会看到：
- 服务记录里**不会新增一条缴款业务**
- 汇缴清册的「**缴款方式**」列仍**空**
- 「**缴款状态**」仍是**未缴款**
- 缴款清册队列里那张清册**仍然在**（再进 `B_jk` 还看得到）
- 首页「缴至年月」**没翻**

因为**委托收款只是登记托收指令，真正扣款由银行在托收日发起**，那时才写缴款状态 / 入账日期。

⇒ **判定成败的时点是托收日之后**，去看汇缴清册的「缴款方式 / 缴款状态 / 入账日期」三列
+ 首页「缴至年月」。银行托收当天批量处理，入账常落当天晚些或次日（T+1）
⇒ **托收日当天看到「未缴款」要推到次日再判。**

⛔ **当天不要重复提交，也不要改走「网上缴款」，会造成重复缴款。**

汇缴页原文佐证：「已办理委托收款缴款的单位，若申报月为当前自然月且无增减员，
可不点击确认申报按钮，托收日当日系统自动生成缴存人员名单。」
⇒ **稳定期每月零操作**；只有增减员的月份才走一遍
`btn_hjzjy_add` / `btn_hjzjy_remove` → `H_submit` 确认申报 → `B_jk` 缴款。

⚠️ 托收日前对公户余额要够。别忘了同月可能还要出实发工资、社保征期可能落在下月。

## 6. 自动化四个坑

1. **找 gjj 标签页必须用 `startswith`，不能用 `in`。**
   `"dwwsyw.gjj.beijing.gov.cn" in page.url` 会**误命中 yzt 登录页** —— 该域名被 URL-encode 进了
   登录页的 `goto` / `redirect_uri` 参数里。用
   `page.url.startswith("https://dwwsyw.gjj.beijing.gov.cn")` 再按路径片段细分：
   `/ish/flow/menu/<CODE>` · `/ish/flow/resources/<hash>`（缴款流）·
   `/ish/flow/tmptask/DATAPOOL_<uuid>`（缴款方式选择页）· `/ish//flow/disppage/DATAPOOL_<uuid>`（结果页）。
   ⚠️ 点 `B_jk` 是**在当前标签页内导航**，之前按 `UPLGJJ0202` 找页会突然找不到。
2. **深链直接 goto 会被判「未登录」或「系统繁忙」**，即使刚 SSO 成功。
   `/ish/index`、`/ish/servicerecord` 直接 goto 返回「系统繁忙！请稍后再试！」；
   `/ish/flow/menu/UPLGJJ0202` 直接 goto 被判未登录。
   ⇒ **SSO 落 `/ish/home` 后在应用内点菜单。**
3. **Bootstrap 模态框 `#confirm_modal` 的 `.modal-footer` 拦 pointer events**，
   Playwright `locator.click()` 会一直 retry 到超时。改用 `evaluate` 里 `button.click()`
   （jQuery 绑定吃合成事件）。
   ⚠️ **但反过来，`evaluate` 点【增减员】菜单无效**（页面不跳），那一步必须用真 `page.click("text=增减员")`。
   —— 又一次印证「同一个站内不同控件的点击姿势都可能互斥」。
4. **`wait_until="domcontentloaded"` 会被慢子资源卡到超时，用 `wait_until="commit"`。**

### 变更对话框的三条设值纪律

- 子页面是**异步载入**的，select 的 option 也是异步填的 ⇒ 要等到「弹窗内唯一可编辑字段」出现
  **且 `select.options.length > 1`** 才能设值，否则会撞到「没有 option」或设进一个空值。
- 设值统一姿势（jQuery 绑定要触发链）：
  ```js
  jQuery(e).val(v).trigger('input').trigger('change').trigger('keyup').trigger('blur');
  try { jQuery(e).combobox('refresh'); } catch (x) {}
  ```
- 弹窗操作**只作用于最顶层 modal**：
  ```js
  Array.from(document.querySelectorAll('.modal'))
    .filter(m => m.classList.contains('in') || getComputedStyle(m).display === 'block')
    .pop()
  ```
- 确认按钮文案正则要留**非锚定分支**：真实标签有「确认提交」，只写 `^(确定|确认|是)$` 会全部漏掉。

## 7. 罚则（只核实到这个程度，别当准数讲）

- 「不办登记逾期罚 1 万~5 万」出自《住房公积金管理条例》第三十七条 —— **未拿到原文**，别当准数引用。
  gjj 站的「行政处罚事项目录」「行政处罚裁量基准」「执法依据」页面正文全是空的（JS 渲染无内容）。
- 但该中心**在真执法**，公开的《责令限期缴存通知书》可查，依据是《住房公积金管理条例》**第三十八条**，
  责令 7 个工作日内补缴单位部分，「逾期不补缴，本中心将依法申请人民法院强制执行」。
- ⇒ **第三十八条管欠缴 / 不缴，不是管未登记。** 两条别混着讲。
