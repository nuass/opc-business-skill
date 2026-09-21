# 政务站浏览器自动化通用铁律

统一走 **Playwright over CDP**（连已登录的 Chrome，端口见项目约定），
**永远 `browser.contexts[0]`** —— `browser.new_page()` 会开一个空 context，登录态全丢。

## 0. 换站必换姿势（最贵的一课）

同一条业务链上的两个政务站，点击方式可能**互斥**：

| 站点类型 | 可用 | 不可用 |
|---|---|---|
| 北京 e窗通 | trusted `Input.dispatchMouseEvent` + `Input.insertText` | 合成 `.click()` 完全惰性 |
| dj.samr.gov.cn | 合成 `el.click()` | trusted 鼠标事件毫无反应 |
| 爱企查 | 裸 CDP，**不开任何 domain** | 一开 `Runtime.enable` 整页只剩报错文案 |
| 人社 yltc（Element UI） | Playwright 常规 locator + 点击 | — |
| 公积金 dwwsyw（jQuery + Bootstrap） | 模态框按钮用 `evaluate` 里 `el.click()` | 同一站的**菜单**却必须用真点击（见 housing-fund.md） |

**别把上一个站验证过的姿势直接搬过去**，同一个站内不同控件也可能互斥。
一次静默 no-op 会浪费很多轮。先合成 `.click()` 试一次、不动再上 trusted（或反之），
**两种都试过再怀疑坐标**。

## 0.5 翻译浮层会吞掉所有点击（浏览器装了翻译扩展时）

页面上叠了一层 `.trans-text` / `.textTranslate` / `.textTranslate-box` 之后，
**坐标算得再准也点不到真元素** —— 事件全被浮层收走，且不报任何错。

```js
// 三个 class 全量置，只置 .trans-text 不够
['.trans-text', '.textTranslate', '.textTranslate-box'].forEach(sel =>
  document.querySelectorAll(sel).forEach(e => e.style.pointerEvents = 'none'));
// 验证：这里返回的第一个必须是你要点的元素
document.elementsFromPoint(x, y).map(e => e.className);
```

⇒ **trusted 点击前先 `elementsFromPoint` 自检**，别等点完了才发现打空。


## 1. 会话铁律

1. **同一门户下不同模块的会话可能相互独立。** 人社的 `yltc` 模块和 `ggfw/unit-center` 就是两套：
   yltc 可能挂着「个人」会话而 unit-center 是「单位」会话，此时访问 `company/*` 返回
   `访问页面失败:用户信息错误，请重新登录`。
2. **会话短（约 10 分钟）且必须原地操作。** `sessionStorage` 是 per-tab，
   **新开标签页会得到「未登录」的假阴性**。
   正确姿势：枚举 `ctx.pages` 找内容含目标关键词的那个标签，在**同一个标签里**导航；
   `pg.reload()` 可以安全保活。
3. **不可逆提交可能连带杀掉上游 SSO 会话** ⇒ 提交后的 401 不等于提交失败，
   先让用户重新扫码登录再判定结果。

## 2. 找活标签页（多标签同 URL 时的必备）

同一个 URL 常有多个标签，**死标签没有 token 且表单元素为 0**。
在死标签上用 Playwright locator 会**永远等待**，表现得像页面 bug。用谓词筛：

```python
async def live_page(ctx, url_part, key='token1', min_items=5):
    for pg in ctx.pages:
        if url_part not in pg.url:
            continue
        st = await pg.evaluate(
            "()=>({tok:!!sessionStorage.getItem('%s'),"
            " items:document.querySelectorAll('.el-form-item').length})" % key)
        if st['tok'] and st['items'] > min_items:
            return pg
    return None
```

⚠️ **`ctx.pages` 的下标会在两次调用之间漂移** —— 永远按 URL 子串 + 存活谓词选标签，**绝不按下标**。

## 3. Vue SPA 路由发现（可复用）

**HTTP 状态码探测无效** —— SPA 对任意路径都返回 index HTML，瞎猜的路径全 200 是假阳性。

正确姿势：
```bash
curl <站点>/<模块>/index | grep -oE 'src="[^"]*\.js"'     # 拿入口 bundle
# 在 bundle 里正则抓三元组
#   path:"..."  +  redirect:"..."  +  meta:{...bannerTitle:"..."}
```

⚠️ **深链 goto 常渲染空白页**（路由守卫拦），必须扫码登录后**在应用内点进去**。

## 4. 路由守卫可能依赖 query 参数

人社 yltc 的实例：守卫源码里只有 URL 带 `?resource=xxx` 才走 `getUnitInfo()` 填
`sessionStorage.unitInfo`；否则 `"dw"===身份 && "company"===path.split("/")[1]` 直接 `next()` 放行。
⇒ 想让 `unitInfo` 有值，**在同一标签页内导航到 `<路由>?resource=1`**，光进路由没用。

## 5. SPA 内路由跳转会丢字典（下拉渲染成空白）

store 的 `SET_DICE_LIST` 往往是整体覆盖（`state.dictList = t`）。
从 A 页 `router.push` 到 B 页后，`dictList` 里会缺 B 页需要的字段 ⇒
`f-select-dict` 类组件的下拉**渲染成空白**（点了没选项，且不报错）。

- **修法：整页 `reload`**（会重新拉 `dict/queryList?aaa100=...` 全量）。
- **判据**：`sessionStorage.dictList` 的 key 里有没有该字段的 `dictType`。
- **查某个下拉的 dictType**：从 `el-form-item` 里的 `.el-select.__vue__` 往上爬 `$parent`，
  直到 `$options.name === 'f-select-dict'`，读 `$props.dictType`。
- ⚠️ 组件的 `optionsSource` / `options` 属性**恒为 0**，**不能**用它判断下拉有没有值（能用的下拉也是 0）。

## 6. 不点开下拉也能读选项 / 直调 React·Vue handler

- **Vue（Element UI）**：`el.__vue__` 往上爬 `$parent` 找到目标组件，直接读 props / 调 methods。
- **React（arco 等）**：从 DOM 上的 `__reactProps$*` / `__reactFiber$*` key 拿 fiber，直调 `onClick`；
  ⚠️ React 版本不同 key 名不同，要遍历 `Object.keys(el)` 匹配前缀。
  select 类控件要先触发 `onVisibleChange(true)` 再选。
- 兜底再考虑真鼠标点击。

## 6.5 React 受控表单（Ant Design 2 / React 15，税务社保费客户端）

- 用原生 value setter + dispatch `input/change` 合成事件**不会更新 React state**：
  值看着填进去了，【保存/提交】按钮仍保持 disabled。**必须真鼠标点进输入框聚焦，
  再用 CDP `Input.insertText` 写值**。
- 覆盖已填值：先**三击（`clickCount:3`）全选**再 insertText；直接 insertText 会追加，
  Cmd+A 在部分输入框里不生效（曾把同一个数叠写成三段）。
- 弹框里同名按钮在 DOM 里可能有**隐藏副本**（`getBoundingClientRect` 全 0），
  选择器命中隐藏那个 ⇒ 点击 no-op 且不报错。先按 `offsetParent !== null`
  过滤出可见按钮再取坐标点。

## 7. 弹窗与验证

- **自定义确认框不一定是 `.el-message-box`**。找不到就**直接 dump `document.body.innerText`**，
  比猜选择器快得多 —— 多次靠这招才发现弹窗原文。
- **提交类操作可能观察不到 XHR**（同步跳转/beacon），**不要用「没抓到请求」判定失败**。
- **唯一可靠验证 = 去列表页/查询接口看这条记录在不在**，核对每个关键字段。

## 8. 截图与 OCR

- 有的站 `Page.captureScreenshot` 渲染空白 ⇒ 改读 `<img>` 的 `src`/data-URI。
- 远程 Windows 客户端截图：**scp 回本地跑 OCR**，
  不要用多模态直接读 1920x1080 远程截图（token 贵且小字看不清）。

## 9. 扫码登录：二维码要程序解码，不要靠眼看

政务站的实名登录几乎全是「电子营业执照小程序扫码」。二维码通常是页面里的一张 data-URI `<img>`。
**把它解码成 URL 再发给用户**，比让用户对着截图扫可靠得多（也能确认拿到的是登录码不是别的码）。

⚠️ **原尺寸（常见 200×200）直接 decode 大概率失败**，放大也没用 —— 缺的是**白色静默区**：

```python
import cv2, numpy as np, base64
img = cv2.imdecode(np.frombuffer(base64.b64decode(b64), np.uint8), cv2.IMREAD_GRAYSCALE)
img = cv2.copyMakeBorder(img, 20, 20, 20, 20, cv2.BORDER_CONSTANT, value=255)  # ← 关键
data, pts, _ = cv2.QRCodeDetector().detectAndDecode(img)
```

补白边后原尺寸就能解出来。**decode 失败 ≠ 二维码坏了**，先补白边再放大再换灰度。

## 10. 内联 JS 用文件传，不要塞进 `python -c`

多层引号 + 换行嵌套极易炸成 `SyntaxError: Invalid or unexpected token`，
而且报错位置指向 shell 而不是 JS，很难看。
**每段 JS 写成 `.js` 文件，Python 里 `open(path).read()` 读进来再送 `Runtime.evaluate`。**

