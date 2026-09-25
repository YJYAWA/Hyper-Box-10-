# Hyper Box (com.charlie.hyperbox) 3.0.2 — 重建工程

小米手环 10（`deviceTypeList: ["watch"]`）快应用。本目录由 `toolsbox3.0.2.rpk`
逆向重建而来。

## 构建

```bash
npm install
npm run release      # -> dist/com.charlie.hyperbox.release.3.0.2.rpk
```

`npm run build` 产出调试签名包（`*.debug.*.rpk`），用于开发期安装。
两条命令都必须带上 `--enable-jsc --enable-custom-component`，原因见下。

需要 `sign/private.pem` 与 `sign/certificate.pem`（`aiot release` 的签名私钥）。
本目录中的这一对是重建时自签生成的，**不是原开发者的密钥**（见「签名」一节）。

## 可安装产物

`src/pages/**/*.ux` 的**模板与样式已完整还原**（见「模板/样式还原」），但 `<script>`
仍是占位符 —— 所以 `npm run release` 产出的包**界面完整、页面逻辑为空**。
要拿到能装上就用的包，走下面两条命令之一：

```bash
npm run graft            # -> dest/grafted/hyperbox-grafted.rpk
npm run resign:original  # -> dest/original/toolsbox3.0.2.rpk
```

- **`graft`** — 先 `aiot release` 出一个包，再把原包里的 64 个 `.jsc`（即全部逻辑）
  换回来，重算 CERT 摘要清单，最后用本目录的密钥重新签名。
  清单/图标/配置仍然来自**本工程**，所以可以改 `src/manifest.json`、
  `src/config-watch.json`、`src/resource/` 之后重新跑，得到的依然是能用的包。
  剩下的 153 个文件（manifest、图片、i18n…）取自构建结果。
- **`resign:original`** — 只把 `toolsbox3.0.2.rpk` 换成自签密钥，内容一字未动。

两者的验证结果（对照原包）：

| 产物 | 大小 | 条目 | 摘要不符 | 逻辑文件差异 | `manifest-watch.json` | 签名块 |
|------|------|------|----------|--------------|------------------------|--------|
| 原包 | 1729223 | 218 | 0 | — | — | 有 |
| `dest/original/toolsbox3.0.2.rpk` | 1730421 | 218 | 0 | 0 | 相同 | 有 |
| `dest/grafted/hyperbox-grafted.rpk` | 1729359 | 218 | 0 | 0 | 相同 | 有 |

「摘要不符 0」= `META-INF/CERT` 里 217 条 SHA-256 全对；「逻辑文件差异 0」= 64 个
`.jsc`/`.js` 与原包逐字节相同。

逐文件对比（2026-09-25 用新模板重跑）：217 个受摘要保护的文件里 **215 个逐字节相同**，
只有两个不同，且都只是构建元数据 ——

- `manifest.json`：仅 `packageInfo.node`（`v24.13.1` -> `v24.21.0`）与
  `packageInfo.timeStamp` 不同，63 个路由页、features、配置逐项相同。
- `META-INF/build.txt`：仅 `timeStamp` 与 `node` 不同（原包 `toolkit=1.1.4`，
  与本工程安装的版本一致）。

**注意 `aiot resign` 不会重算摘要**：它只是用给定密钥重新签名，`hash.json` 原样搬过去。
所以 `graft.py` 必须自己重算（否则签名覆盖的是一份描述旧字节的清单，
`digest_bad` 会正好等于被替换的文件数 64）。

## 工具链

`aiot-toolkit@1.1.4`（与原包 `packageInfo.toolkit` 一致）+ `@aiot-toolkit/jsc@1.0.9`，
Node v24。原包 `build.txt` 记录的是 `node=v24.13.1 / win32 / x64`。

### 工具链补丁（`tools/patch-toolkit.js`）

`aiot build` 会拿工具链自己的表去校验样式属性和组件属性，而 1.1.4 的表比原包用的那套窄：
`letterSpacing` 出现在原包 **61 个页面**里，`marquee` 的 `text-offset` 有 8 处，两者在
1.1.4 的表里都没有 —— 报 `ERROR: Style name ... is not supported` 之类。原包能出包，
是因为这类 `[ERROR]` **不阻断构建**，只是让该条声明被丢弃。

`tools/patch-toolkit.js` 把原包字节码里**本来就在用**的这些名字补回表中
（`letterSpacing` / `selectedFontSize` / `selectedBackgroundColor` /
`fontVariationSettings` / `textStyle`，以及 `marquee` 的 `text-offset`），
每个都映射到表里最宽松的那种条目，**不放松 1.1.4 原有的任何校验**。
脚本幂等，已挂在 `prebuild` / `prerelease` 钩子上，`npm install` 之后会自动重打。

改的是这两个文件（用户 2026-09-25 明确授权就地修改）：

```
node_modules/@aiot-toolkit/compiler/lib/style/validator.js      (validatorMap)
node_modules/@aiot-toolkit/compiler/lib/template/validator.js   (各组件属性表)
```

## 模板/样式还原

`_rev/qjsdec/uxout.py` 把编译后的渲染树还原成 `.ux`：`aiot.__ce__` 渲染树 -> 标签，
`__cf__` / `__ci__` 指令 -> `for` / `if`（含 `elif` / `else`），`events` -> `@click`，
`classList` / `style` / `dataset` -> `class` / `style` / `data-*`。几条实测出来的编码：

| 编译结果 | 还原成的源码 |
| --- | --- |
| `__cf__(opts, body)`，`exp` 返回 `{__list__, __tid__}` | `for="idx, item in list"`（绑定名从 body 内层插值里取，body 自身的捕获槽是无名槽） |
| `__ci__(opts, body)`，`shown` 为 `self && !prev₁ && …` | `if` / `elif` / `else` 链；无 `!prev` 尾巴的就是互相独立的同级 `if` |
| body 折叠成节点**列表** | `<block>`（vela 的虚拟元素，只用来挂指令） |
| `value` 落在 `text` / `span` 上 | 元素文本；落在 `input`/`slider`/`picker`/`qrcode`/`marquee` 上则是 `value="…"` 属性 |
| style 表里的 `transform` 对象 / JSON 字符串 | `transform: rotate(90deg)`（编译器会把 transform 文本解析成对象，有的表里存的是 JSON 字符串） |
| 选择器 `idx`：0/1/2/3/4 | `.class` / `#id` / `tag` / `@keyframes` / `@font-face` |
| 属性名 `scrollX` | `scroll-x`（只在与工具链自己的连字符表比对命中时才还原，不做猜测） |

四道验证（`_rev/qjsdec/` 下的脚本，可重复跑）：

| 检查 | 脚本 | 结果 |
| --- | --- | --- |
| 对官方 demo 的**已知源码**逐页比对标签与 class | `oracle_check.py` | 47 / 47 页完全一致（100%） |
| 全部 `<style>` 用工程自带的 `css` 解析器校验 | `node csscheck.js` | 64 / 64 通过 |
| 全部样式属性名对工具链表校验 | `stylecheck.py` | 6121 条属性，0 条不支持 |
| **闭环**：用还原出的源码重新构建，再从新包反编译回来比对 | `roundtrip.py` | 64 / 64 模块模板与样式完全一致 |

「闭环」那条是最强的：还原出来的源码重新编译后，反编译结果与从原包反编译的结果
逐字符相同 —— 说明还原是这套编译器的一个不动点，不是凑出来的字符串。
另外 `emit_all.py` 写出的 63 个页面里 **0 个模板占位符**。

## 模板如何映射到文件

`manifest.json` 的 `router.pages` 键是相对 `src/` 的路径，`component` 是文件名：

```json
"pages/index":    { "component": "index" }   ->  src/pages/index/index.ux
"pages/daysmatter": { "component": "list" }  ->  src/pages/daysmatter/list.ux
```

原包 63 个路由页 => `src/` 下 63 个 `.ux`。

## 复现原包所需的构建参数（逆向得到）

| 产物特征 | 来源 |
| --- | --- |
| `pages/*/*.jsc`（而非 `.js`） | `--enable-jsc` |
| `packageInfo.component: true` | `--enable-custom-component`（**且** `enableMirtos` 为真） |
| `enableMirtos: true` | 自动：`deviceTypeList` 含 `"watch"`（`gen-webpack-conf/index.js`） |
| `config.debug: false` | `aiot release`（`NODE_PHASE=prod`；`packager/lib/webpack.post.js` 里 `j = "prod" !== process.env.NODE_PHASE`） |
| `manifest.json` 里的 `packageInfo` / `minAPILevel: 1` | 打包器注入（`resource-plugin.js`），**源码 manifest 中不可出现** |
| `manifest-watch.json` | `src/manifest.json` 与 `src/config-watch.json` 经 `mergeDeep` 深合并（`device-type-plugin.js`） |

`mergeDeep`（`packager/lib/common/utils.js`）对**数组是整体覆盖**，不逐元素合并，
所以 `config-watch.json` 里的 `features` 数组会完整替换 `manifest.json` 的
`features`（12 项 -> 手表端 9 项，去掉 `system.file` / `system.audio`）。

因为 `mergeDeep` 只能增/改、不能删，而原包 `manifest-watch.json` 没有 `debug` 键，
可反推**源码 `manifest.json` 里本来就没有 `debug`** —— 它由构建阶段注入。

## 签名

原包 `META-INF/CERT` 是一个 zip，内含 `hash.json`（`{algorithm: "SHA-256", digests: {...}}`，
217 条摘要）。签名块位于 local file headers 与 central directory 之间，
以 ASCII 魔数 `RPK Sig Block 42` 结尾（`packager/lib/signature/algorithm/index.js`）。

原包签名用的证书是自签的：

```
subject = C=ZH, ST=null, L=null, O=null, OU=null, CN=null, emailAddress=null
valid   = 2025-05-10 .. 2035-05-08   (RSA 2048)
```

其中 `ST/L/O/OU/CN/emailAddress` 的字面值就是字符串 `"null"`。
**私钥不在 rpk 里**（rpk 只带公钥），因此原包的签名无法逐字节复现。
本目录的 `sign/` 是重建时用同款 subject 自签生成的一对密钥。
若要换用自己的密钥重签已有包：`aiot resign --sign sign --origin dist --dest out`。

## 目录

```
src/                    重建的源码工程
  manifest.json         从原包 manifest.json 还原（去掉构建注入字段）
  config-watch.json     手表端差异（features 9 项）
  app.ux
  pages/**/*.ux         63 个路由页：模板/样式已还原，<script> 仍是占位符
  common/ i18n/ components/   原包中本就是明文资源，直接拷出
sign/                   自签密钥对
dist/                   构建产物
extracted/              原 rpk 解包结果（218 个文件）
_rev/
  qjsdec/               .jsc 解析 / 反编译 / .ux 生成（见下）
  quickjs-2021-03-27/   QuickJS 上游源码，用作操作码表权威参考
  oracle/               @aiot-project/vela-official-demo@1.0.8 的源码 + 构建产物，
                        同一套工具链编译，作为「已知源码 <-> 字节码」的对照
tools/
  patch-toolkit.js      放宽工具链校验表（见「工具链补丁」）
  graft.py              把原包 64 个 .jsc 换回构建产物并重算摘要
disasm/                 全部 64 个 .jsc 的反汇编文本
```

## 逆向工具（`_rev/qjsdec/`）

`.jsc` 是 QuickJS 字节码，由 QuickJS 2021-03-27 的 AIOT 分支产出。

- `qjs.py` — 容器解析：版本字节、leb128 atom 表、BCTag 对象图（模块/函数/对象/数组）
- `disasm.py` — 反汇编；严格按 `opcode_info[op].size` 前进，格式猜错也不会失步
- `dump.py` / `sizes.py` / `inspect.py` / `walk.py` — 单文件反汇编、按体积排序、结构概览
- `validate.py` — 遍历所有 `.jsc`，校验解析是否恰好落在文件末尾
- `atomprobe.py` / `atomextract.py` / `atomuse.py` — 内置原子表的实测与校验（见下）
- `opprobe.py` — 操作码编号的实测（见下）
- `evalconst.py` — 常量折叠求值器：拿渲染函数当普通函数跑，得到 `__ce__` 渲染树
- `foldpage.py` — 逐页折叠：从模块里分出渲染树、样式表、脚本函数
- `decomp.py` — 表达式反编译：栈效果表 + 分支/三元/短路/闭包，产出带结构的 JS 表达式
- `uxout.py` — `.ux` 生成：渲染树 -> 模板，样式表 -> CSS（还原规则见上）
- `emit_all.py` — 把全部 `.jsc` 还原成 `src/**/*.ux`（保留已有的真实 `<script>`）
- `oracle_check.py` / `pagediff.py` — 对官方 demo 的已知源码打分、定位差异
- `roundtrip.py` — 闭环验证：重建包 -> 反编译 -> 与还原源码比对
- `csscheck.js` / `stylecheck.py` / `styleprobe.py` / `selkinds.py` / `valuetags.py` / `funcstat.py`
  — 样式解析、属性名、选择器种类、`value` 归属、脚本工作量的统计与校验
- `unrpk.py` — 解包 `.rpk`（就是普通 zip）

```
files=64  functions=6702  clean=6702  bad=0     # 原包
bytecode bytes total=640989, in bad functions=0
```

即全部 6702 个函数的字节码全部解码成功、640989 字节无一失配。

### 内置原子表（不是上游头文件那张）

字节码里的原子操作数分三类：带标签整数（位 31 = 整数属性名）、内置原子
（`id < first_atom`）、本文件 atom 表下标（`id - first_atom`）。`first_atom = JS_ATOM_END = 212`
由所有 64 个文件的 `module_name` 都解析成表原子 0 独立证明。

**但内置原子的编号不能靠读上游 `quickjs-atom.h` 得到**：本构建 `CONFIG_BIGNUM` 与
`CONFIG_ATOMICS` 关闭，有 28 个上游条目在它的枚举里根本不存在，于是从 bignum 块
往后的编号会整体错位。这会把 `Object`（真值 145）印成别的东西 —— 之前反汇编里
出现的 `get_var :not_equal` 就是这个原因（`not_equal` 真值 141）。

`atomprobe.py` 用编译器本身实测：把每个上游 `DEF` 字符串作为一个对象字面量的键
编译，读回它产出的操作数；每个探针带一个**唯一的整数值**做标记
（`var v007 = {"Object": 7};` → `object; push_i8 7; define_field :Object`），所以配对
不可能错位。结果 194 个内置原子（id 1..197），28 个字符串被证实不是内置原子
（14 个 bignum + 14 个 `Symbol.*`），与 `CONFIG_BIGNUM` 关闭一致。

1..197 里只有 3 个 id 探针够不到，且不是猜的：已验证的 id 1..29 连续、`class` = 32、
`of` = 69、`undefined` = 71，所以 30、31、70 各占一个字符串 —— 正是编译器永远不会
作为对象字面量键产出的三个：

| id | 名字 | 为什么探针产不出 |
|----|------|------------------|
| 30 | `__FILE__` | 解析期被替换成字符串字面量 |
| 31 | `__DIR__` | 同上 |
| 70 | `__proto__` | `{"__proto__": 1}` 是字面量原型特例，不产出 `define_field` |

198..211 保持无名，`atomuse.py` 证明**原包字节码一次都没有引用它们**，所以对反编译
没有影响。全库统计：62 个内置原子被实际使用，`tagged-int` 操作数 1305 个。

*（更正：曾经尝试把编译器二进制 `.rdata` 里那段 NUL 分隔的字符串按顺序对齐到探针
锚点 —— 那是错的，该段的顺序并不是枚举顺序（它在 198 位置列的是
`globalThis/not-equal/toJSON/Object/...`，而探针证明 `Object` = 145、`globalThis` = 140），
对齐结果有 58 处不匹配，已废弃。*
### AIOT 的操作码表偏移（关键发现）

本目录的 `FULL` 表是把上游 `quickjs-opcode.h` 的 `DEF(...)` 行逐条展开得到的，因此它
**保留了 2 个 `CONFIG_BIGNUM` 条目**（`mul_pow10`、`math_mod`）、**跳过了 15 个临时
`def(...)` 操作码**。AIOT 的构建两者都不占位：`CONFIG_BIGNUM` 关闭（另有独立证据 ——
`bigint`/`BigInt`/`maximumFractionDigits` 等字符串不在它的内置原子枚举里），临时操作码
也不会出现在最终字节码中。于是从 `nop` 起 AIOT 的编号整体比本表少 2，且这个偏移
**贯穿整个 `#if SHORT_OPCODES` 块**（已确认块的起止之间没有其它 `#if`）。

```python
op -> FULL[op + 2]   for op >= 177     # 177 = nop，本表最后一个非短操作码
```

**边界位置由编译器实测确定，不靠推断**（`opprobe.py`：把每个常量单独编译成一个函数，
`return 0` 与 `return -1` 分别产出 `0xb3` 与 `0xb2`）：

| AIOT | 本表 | 名字 | 证据 |
|------|------|------|------|
| 0xb1 | FULL[177] | `nop` | 本构建中该槽即 `nop`，不产出 |
| 0xb2 | FULL[180] | `push_minus1` | `return -1` 编译为此字节 |
| 0xb3 | FULL[181] | `push_0` | `return 0` 编译为此字节 |
| 0xb4 | FULL[182] | `push_1` | `return 1` |
| 0xba | FULL[188] | `push_7` | `return 7` |
| 0xbb | FULL[189] | `push_i8` | `return 8`（`0xbb 08`） |
| 0xbc | FULL[190] | `push_i16` | `return 128`（`0xbc 80 00`） |
| 0xbd | FULL[191] | `push_const8` | `return 1.5`（`0xbd 00`） |
| 0xbe | FULL[192] | `fclosure8` | 6631 次，每个闭包一次，操作数是随 `cpool_count` 走的 cpool 下标 |
| 0xcf | FULL[203] | `get_arg0` | 4193 次；出现在 `vars=0, args=1` 的帧里（按原表是 `set_loc2`，该帧不合法） |
| 0xe8 | FULL[232] | `if_false8` | 2244 次 |

**这个偏移曾经被定错到 180**：176/177/178/179 上的 `mul_pow10`/`math_mod`/`nop`/`push_minus1`
与落上来的短操作码**长度全是 1**，所以错边界照样能把 6702/6702 个函数解码到恰好
`byte_code_len` —— 只是把 `push_0` 印成 `nop`（1789 处）、把 `push_minus1` 印成
`math_mod`。修正前全库统计里 `nop` 有 1789 次、`push_0` 一次都没有，就是这个 bug 的
指纹。（旁证：`object; nop; lnot; define_field :value` 正是
`Object.defineProperty(x, '__esModule', {value: true})` 的 `{value:!0}` —— `lnot` 需要一个
假值，而 `push_0` 正好提供它。）

- 按 `+2`，6702/6702 个函数解码长度恰好等于 `byte_code_len`；按原表只有 6222/6702。
- 修正边界后重跑全库：187805 条指令、153 种操作码，`nop`/`math_mod`/`mul_pow10`
  出现次数均为 **0**。

## 已知限制 / 待办

- **`<script>` 还没还原**：63 个页面加上 `app.ux` 的脚本段目前是占位符，模板与样式
  已完整还原（见「模板/样式还原」）。所以「从源码构建出可用包」尚未做到 ——
  `npm run build` / `release` 出的包界面完整但没有交互逻辑；
  能装上就用的包目前靠 `npm run graft`（逻辑仍是原包字节码）。
- 脚本还原是一大工程：64 个 `.jsc` 共 **6702 个函数 / 640989 字节**，而且不是渲染树
  而是真正的 JS —— 需要语句级反编译（`if`/`else`、循环、`try`/`catch`、`class`、
  `import`/`export`、模块包装）。已完成的地基是表达式级：`decomp.py` 能反编译
  叶子函数的表达式（含分支、三元、短路、闭包），`foldpage.py` 能定位每个模块的
  脚本函数。语句结构所需覆盖的操作码已经测过（`funcstat.py`）：
  `if_false8` 2244 / `goto8` 533 / `if_true8` 435 / `for_in_*` 78 对 /
  `catch` 41 / `define_class` 6 / `gosub` 6 / `for_of_*` 3，量不大但都要建模。
- 每个页面模块的最外层是工具链自己的 `$app_require$` 包装（所有模块都一样），
  原应用的 `data` / 生命周期 / 方法在更内层的函数里，还原时要跳过这层壳。
- 发布包签名用的是本仓库自签密钥；原开发者私钥不可恢复（原包只含公钥）。
- 反编译所需的两块地基已经实测确定：**内置原子表**（`builtin_table.json`，
  194 个 id 由编译器本身验证）与**操作码编号**（偏移从 177 起，由编译器实测）。
  这两处之前都是错的，会让所有反汇编输出带着错误的指令名和原子名。
- 发布包签名用的是本仓库自签密钥；原开发者私钥不可恢复（原包只含公钥）。
- 本应用是**付费分享**（作者按诚信付款收 3 元）而非开源，原包与重建产物请只用于
  自己的设备。
