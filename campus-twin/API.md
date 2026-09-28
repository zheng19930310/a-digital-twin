# 瑞晟数字孪生平台 · 交互 JS API 说明

> 单文件页面（`index.html`），无任何外部依赖。所有对外能力挂在 `window.DT` 上。
> 页面脚本在 `<body>` 末尾同步执行，因此 **`DT` 在 DOM ready 时即可用**；若在 `<head>` 里写集成代码，请包一层 `window.addEventListener('DOMContentLoaded', ...)`。
>
> 版本：`DT.version === '1.4.0'`

---

## 0. 30 秒上手

```html
<iframe id="twin" src="index.html" width="100%" height="800" frameborder="0"></iframe>

<script>
  var twin = document.getElementById('twin');
  twin.addEventListener('load', function () {
    var DT = twin.contentWindow.DT;      // 同源 iframe 直接取
    DT.report.open('b-main');            // 打开主厂房报表
    DT.scene.setRotation(45);            // 转到 45°
    DT.onSelect = function (p) { console.log('用户点了', p.name); };
  });
</script>
```

如果页面本身就是孪生页（非 iframe），直接 `window.DT` 即可。

---

## 1. 命名空间总览

| 命名空间 | 作用 |
| --- | --- |
| `DT.scene` | 三维场景：旋转、缩放、平移、对象清单、定位、坐标换算 |
| `DT.report` | 统计报表：打开/关闭、数据读取与热更新、双翼折叠、倾斜、体感 |
| `DT.tip` | 数据浮框：外部主动弹出 / 收起 |
| `DT.onXxx` | 生命周期回调（赋值覆盖），同时以 `dt:xxx` CustomEvent 派发 |

约定：
- **角度一律用「度」**（`0 ~ 360`），内部自动转弧度。
- **坐标**分两套：`世界坐标 (x, y, z)` 是厂区网格坐标（约 `x:0~48, y:0~44, z:高度`）；`屏幕坐标` 是浏览器 CSS 像素。
- 所有方法都有返回值或明确副作用，无隐藏状态；调用失败返回 `false` / `null`，不抛异常。

---

## 2. `DT.scene` — 场景控制

### 2.1 视角

| 方法 | 参数 | 返回 | 说明 |
| --- | --- | --- | --- |
| `rotate(deg)` | `deg` 数值，正=顺时针 | — | 在当前角度上叠加旋转量，带缓动 |
| `setRotation(deg)` | `deg` 数值 | — | 直接设定绝对角度 |
| `getRotation()` | — | `Number` | 读取当前角度，`0~360`，保留 1 位小数 |
| `snapRotation()` | — | — | 吸附到最近的 45° 标准视角 |
| `resetView()` | — | — | 视角 + 缩放 + 平移全部复位 |

```js
DT.scene.rotate(30);            // 向右再转 30°
DT.scene.setRotation(135);      // 直接跳到 135°
var d = DT.scene.getRotation(); // 135
DT.scene.snapRotation();        // 吸附到 135°（已是标准角，不变）
```

> 旋转带缓动：`getRotation()` 返回的是**当前实际渲染角度**，设定后约 0.5~1 秒才收敛到目标值。若需要立刻读取目标值或做后续坐标换算，请 `await` 一帧以上（约 300ms）再取值。

### 2.2 缩放与平移

| 方法 | 参数 | 返回 | 说明 |
| --- | --- | --- | --- |
| `zoom(k)` | `k` 倍率，如 `1.2` / `0.8` | — | 相对缩放，自动夹在 `0.6 ~ 3.2` |
| `setZoom(z)` | `z` 绝对值 | — | 绝对缩放 |
| `getZoom()` | — | `Number` | 当前缩放 |
| `pan(dx, dy)` | 像素 | — | 按屏幕像素平移视点 |
| `center()` | — | — | 平移归零，场景回到屏幕中心 |

```js
DT.scene.zoom(1.35);
DT.scene.pan(-120, 40);      // 向左上挪一点
DT.scene.center();
```

### 2.3 对象与定位

| 方法 | 参数 | 返回 | 说明 |
| --- | --- | --- | --- |
| `listObjects()` | — | `Array<Object>` | 全部可点击对象：`{ key, name, type, x, y, w?, d?, h? }` |
| `focus(x, y, z)` | 世界坐标 | — | 把该世界坐标移到屏幕正中 |
| `focusKey(key, zoomTo?)` | 对象 key、可选缩放 | `Boolean` | 按 key 定位对象，找不到返回 `false` |
| `screenOf(x, y, z)` | 世界坐标 | `{x, y}` | 世界坐标 → 屏幕坐标（做外部标记层用） |
| `hover(key)` | 对象 key | — | 设置悬停对象（高亮跟随） |

`type` 取值：`building`（建筑）、`tank` / `pump` / `cam` / `lamp` / `agv` / `solar`（设备）。
`dev:agv` 的 `x/y` 是**巡逻中的实时坐标**，每次调用都会变，因此 `focusKey('dev:agv')` 会跟到车当前位置。

```js
DT.scene.listObjects().forEach(function (o) {
  console.log(o.key, o.name, o.type, o.x, o.y);
});
DT.scene.focusKey('b-warehouse', 1.6);   // 飞到成品仓库并放大
var p = DT.scene.screenOf(27, 6.5, 4.2); // 主厂房屋顶的屏幕位置
```

---

## 3. `DT.report` — 统计报表

### 3.1 打开 / 关闭

| 方法 | 参数 | 返回 | 说明 |
| --- | --- | --- | --- |
| `open(key, quiet?)` | 对象 key；`quiet=true` 静默刷新 | `Boolean` | 打开该对象报表，key 无效返回 `false` 并 `console.warn`。`quiet` 模式下不重放弹出动画、不重置折叠状态、不触发 `onOpen` |
| `close()` | — | — | 关闭报表（带动画） |
| `isOpen()` | — | `Boolean` | 当前是否展开 |
| `current()` | — | `String\|null` | 当前报表 key |

### 3.2 数据读取与热更新

| 方法 | 参数 | 返回 | 说明 |
| --- | --- | --- | --- |
| `keys()` | — | `Array<String>` | 全部有报表的 key |
| `data(key)` | 省略则返回全量 | `Object` | 读取报表数据结构 |
| `update(key, patch)` | `patch` 为字段子集 | `Boolean` | 浅合并字段；若该报表正打开则**静默重绘**（不打断当前折叠状态与动画） |

报表数据结构：

```js
{
  name: '主厂房 · 生产运行报表',
  sub : '建筑面积 26,000 ㎡ · 8 条吊挂产线',
  kpis: [ ['指标名', '数值', '单位', '同比描述', 'up|down|flat'], ... ],
  bar : { labels: [...], values: [...], color: '#19c2b4', unit: '件' },
  pie : { title: '能耗构成', items: [ ['名称', 百分比], ... ] },
  table: {
    title: '产线明细',
    cols : ['产线','款式','计划(件)','完成(件)','达成率','状态'],
    rows : [ ['L1 智能吊挂','外套',2200,2150,'97.7%','ok'], ... ]
    // 最后一列写 ok / warn / err / info 会自动渲染成状态徽章
  }
}
```

```js
// 只换柱状图数据，其余保持不变
DT.report.update('b-main', {
  bar: { labels: ['1月','2月','3月','4月'], values: [8200, 9100, 10400, 11200],
         color: '#19c2b4', unit: '件' }
});

// 轮询刷新表格 + KPI
setInterval(function () {
  fetch('/api/line/L1').then(r => r.json()).then(function (d) {
    var cur = DT.report.data('b-main');
    cur.table.rows[0][3] = d.done;          // 完成(件)
    cur.kpis[0][1] = d.output.toLocaleString();
    DT.report.update('b-main', { table: cur.table, kpis: cur.kpis });
  });
}, 5000);
```

### 3.3 立体效果

| 方法 | 参数 | 返回 | 说明 |
| --- | --- | --- | --- |
| `fold(side, folded)` | `side`: `'L'`/`'R'`/`'both'`；`folded`: 默认 `true`，传 `false` 展开 | — | 折叠 / 展开左右侧翼 |
| `tilt(rx, ry)` | 度数 | — | 手动设定面板倾角（调试或外部联动用） |
| `gyro(on)` | `Boolean` | `Boolean` 生效状态 | 开关手机设备方向驱动的晃动 |

```js
DT.report.fold('both');        // 两侧全折起来，只留主面板
DT.report.fold('R', false);    // 右翼重新展开
DT.report.tilt(0, 6);          // 向右倾 6°
```

> 说明：指针进入面板范围时，面板会自动「回正」并停止写 transform，以保证图表悬停、滚轮、拖动的命中精度；离开面板后恢复跟手晃动。这是刻意为之的交互策略，不是 bug。

---

## 4. `DT.tip` — 数据浮框

| 方法 | 参数 | 说明 |
| --- | --- | --- |
| `show(html, x, y)` | HTML 字符串 + 屏幕坐标 | 在指定位置弹浮框，自动贴边避让、上方放不下则翻到下方 |
| `hide()` | — | 收起 |

```js
DT.tip.show('<span class="tt">告警</span><span>3# 变压器温升 82℃</span>', 640, 300);
setTimeout(function(){ DT.tip.hide(); }, 3000);
```

浮框内可用的样式类：`.tt`（标题）、`.tv`（数值）、`.tu`（单位）、`.up`（绿）、`.down`（红）。

---

## 5. 回调与事件

两种消费方式，任选或并用。

### 5.1 赋值回调

```js
DT.onOpen     = function (p, DT) {};   // p = { key, name }        报表打开
DT.onClose    = function (p, DT) {};   // p = { key }              报表关闭
DT.onSelect   = function (p, DT) {};   // p = { key, name }        场景内点击对象
DT.onHover    = function (p, DT) {};   // p = { key|null }         场景内悬停对象
DT.onRotate   = function (p, DT) {};   // p = { deg }              视角变化（已节流）
DT.onTableRow = function (key, rowIndex, cells) {};  // 报表表格行点击
```

### 5.2 CustomEvent

每个回调同时派发 `dt:<事件名>`，数据在 `e.detail`：

```js
window.addEventListener('dt:select', function (e) {
  console.log(e.detail.key, e.detail.name);
});
```

| 事件 | detail |
| --- | --- |
| `dt:open` | `{ key, name }` |
| `dt:close` | `{ key }` |
| `dt:select` | `{ key, name }` |
| `dt:hover` | `{ key }`（离开为 `null`） |
| `dt:rotate` | `{ deg }` |

> `onTableRow` 只走回调、不发 CustomEvent（参数是三元组，不属于统一 payload 形态）。

---

## 6. 联动调用示例

### 6.1 点击表格行 → 场景飞到对应设备

```js
// 报表里的产线明细，点一行就让三维场景对准它
var LINE_POS = { 'L1': [18, 6], 'L2': [22, 6], 'L3': [26, 6], 'L4': [30, 6],
                 'L5': [34, 6], 'L6': [37, 6], 'L7': [24, 9], 'L8': [33, 9] };

DT.onTableRow = function (key, idx, cells) {
  var m = /^L(\d)/.exec(cells[0]);
  if (!m) return;
  var p = LINE_POS['L' + m[1]];
  if (!p) return;
  DT.scene.focus(p[0], p[1], 4.2);   // 平移到该产线上方
  DT.scene.setZoom(1.8);
  DT.scene.hover('b-main');          // 主厂房保持高亮
};
```

### 6.2 外部工具栏 → 控制孪生页

```html
<div class="toolbar">
  <button data-rot="-45">⟲ 左转</button>
  <button data-rot="45">⟳ 右转</button>
  <button data-obj="b-main">主厂房</button>
  <button data-obj="dev:agv">AGV</button>
</div>

<script>
document.querySelector('.toolbar').addEventListener('click', function (e) {
  var b = e.target.closest('button'); if (!b) return;
  if (b.dataset.rot) DT.scene.rotate(Number(b.dataset.rot));
  if (b.dataset.obj) {
    DT.scene.focusKey(b.dataset.obj, 1.7);
    DT.report.open(b.dataset.obj);
  }
});
</script>
```

### 6.3 报警列表 → 定位 + 打开报表 + 浮框提示（三段联动）

```js
function jumpToAlarm(a) {
  DT.scene.resetView();
  DT.scene.setRotation(a.view || 0);
  DT.scene.focusKey(a.key, 1.9);
  DT.report.open(a.key);
  var p = DT.scene.screenOf(a.x, a.y, a.z || 3);
  DT.tip.show('<span class="tt">' + a.title + '</span>' +
              '<span class="down">' + a.level + '</span> ' + a.msg, p.x, p.y);
  setTimeout(function () { DT.tip.hide(); }, 4000);
}

// 报警推送（WebSocket / SSE 均可）
socket.on('alarm', function (a) {
  if (a.auto) jumpToAlarm(a);
  else renderAlarmButton(a, function () { jumpToAlarm(a); });
});
```

### 6.4 视角随报表走（打开建筑自动转正，关闭后回到总览）

```js
var lastView = null;

DT.onOpen = function (p) {
  lastView = { deg: DT.scene.getRotation(), zoom: DT.scene.getZoom() };
  DT.scene.snapRotation();
  DT.scene.focusKey(p.key, 1.45);
  DT.report.fold('both');            // 大屏下先折起双翼，减少遮挡
};

DT.onClose = function () {
  if (!lastView) return;
  DT.scene.setRotation(lastView.deg);
  DT.scene.setZoom(lastView.zoom);
  DT.report.fold('both', false);
  lastView = null;
};
```

### 6.5 实时数据接入（KPI + 图表 + 表格一次刷新）

```js
function applyTelemetry(key, d) {
  var cur = DT.report.data(key);
  if (!cur) return;

  cur.kpis = cur.kpis.map(function (row, i) {
    var v = d.kpis && d.kpis[i];
    return v === undefined ? row : [row[0], String(v), row[2], row[3], row[4]];
  });

  if (d.bar)   cur.bar   = Object.assign({}, cur.bar,   d.bar);
  if (d.pie)   cur.pie   = Object.assign({}, cur.pie,   d.pie);
  if (d.rows)  cur.table = Object.assign({}, cur.table, { rows: d.rows });

  DT.report.update(key, {
    kpis: cur.kpis, bar: cur.bar, pie: cur.pie, table: cur.table
  });
}

// 后端返回：{ kpis:[13400, 86.4, 3.58, 0.21], bar:{values:[...]}, rows:[...] }
setInterval(function () {
  fetch('/api/twin/b-main').then(r => r.json())
    .then(d => applyTelemetry('b-main', d));
}, 3000);
```

### 6.6 悬停联动：场景对象 ↔ 外部侧栏

```js
var sideInfo = document.getElementById('sideInfo');

DT.onHover = function (p) {
  if (!p.key) { sideInfo.textContent = ''; return; }
  var o = DT.scene.listObjects().find(function (x) { return x.key === p.key; });
  sideInfo.textContent = o ? (o.name + ' · ' + o.type) : p.key;
};

// 反向：鼠标移到侧栏条目上，让场景对应对象亮起来
sideInfo.addEventListener('mouseover', function (e) {
  var li = e.target.closest('[data-key]');
  if (li) DT.scene.hover(li.dataset.key);
});
```

### 6.7 iframe 跨文档联动（父子页面）

```js
// 父页面
var frame = document.getElementById('twin');
function callTwin(fn) { frame.contentWindow.eval('(' + fn + ')()'); }

// 子页面（index.html 内）主动通知父页面
DT.onSelect = function (p) {
  if (window.parent !== window) {
    window.parent.postMessage({ source: 'digital-twin', type: 'select', payload: p }, '*');
  }
};
```

> 生产环境请把 `'*'` 换成明确的 `targetOrigin`，并对 `message` 事件校验 `event.origin` 与 `data.source`。

---

## 7. 可用 key 速查

| 分类 | key |
| --- | --- |
| 建筑 | `b-main` 主厂房 · `b-assemble` 装配车间 · `b-warehouse` 成品仓库 · `b-office` 研发办公楼 · `b-power` 变电站 · `b-gate` 园区门厅 |
| 设备 | `dev:tank` 消防水箱 · `dev:pump` 给水泵房 · `dev:cam` AI 摄像机 · `dev:agv` AGV 搬运车 · `dev:solar` 屋顶光伏 |
| 路灯 | `dev:lamp0` ~ `dev:lamp3` |

运行时以 `DT.report.keys()` 为准（新增对象后无需改文档，直接枚举）。

---

## 8. 集成注意事项

1. **不要覆盖 `window.DT` 整体**，只赋值需要的 `onXxx` 回调，方法对象保持原样。
2. **`update()` 是浅合并**：传入的字段整体替换，未传字段不动。改单个 KPI 时请先 `data(key)` 取全量再回写。
3. **`open()` 后请留 300~500ms 再做坐标类操作**（弹出动画 + 倾斜回正期间，面板几何仍在变化）。
4. **移动端（宽度 ≤ 900px）**：双翼改为覆盖式抽屉并默认合拢，自绘滚动条隐藏，用系统滚动。外部布局请预留这个断点。
5. **键盘**：`Q` / `E` / `←` `→` 旋转，`R` 复位，`Esc` 关闭报表。宿主页面占用这些键时，请在捕获阶段自行 `stopPropagation`。
6. **性能**：场景为单 canvas、`requestAnimationFrame` 驱动；报表打开时倾斜循环接管，关闭后回到场景渲染，不会双跑。
