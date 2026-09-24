# WebForms 项目中 Android 平板 PDF 展示方案

> 适用场景：ASP.NET WebForms / .NET Framework 4、医院内网、Android 平板、厂商定制 Chromium 浏览器。  
> 当前问题：`<embed>` 和 `<iframe src="xxx.pdf">` 在 Windows Chrome 可以显示，但 Android 平板/Chrome 移动模式下只能看到 iframe 容器，PDF 内容为空。  
> 目标：**PDF 在当前报告页面直接展示，不跳转、不依赖 Google Docs Viewer 等外部在线服务。**

---

## 1. 结论

### 第一备选：自部署 PDF.js

推荐优先采用：

```text
报告页面
   │
   ├── 图片 → <img>
   │
   └── PDF → <iframe>
             │
             └── /pdfjs/web/viewer.html?file=/xxx.pdf
                         │
                         └── PDF.js
                              │
                              └── HTML5 Canvas 渲染 PDF
```

PDF.js 是浏览器端 PDF 解析/渲染方案，本身不需要调用外部在线 PDF 服务，可以把完整的 PDF.js 文件放在医院内网服务器上。Mozilla 官方文档也提供预构建的 viewer，可直接部署到网站中。

官方资料：

- PDF.js GitHub：https://github.com/mozilla/pdf.js/
- PDF.js 网站部署说明：https://github.com/mozilla/pdf.js/wiki/Setup-pdf.js-in-a-website
- PDF.js 浏览器兼容性 FAQ：https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions

---

# 2. 为什么不继续使用 embed / iframe 直接打开 PDF

当前已经验证：

```html
<embed src="/report/test.pdf">
```

Windows Chrome：

```text
正常
```

Android/移动环境：

```text
无法显示
```

进一步测试：

```html
<iframe
    src="/report/test.pdf"
    style="width:100%;height:1000px;border:5px solid blue;">
</iframe>
```

可以看到：

```text
蓝色 iframe 边框
红色背景
```

但是 PDF 内容为空。

这说明：

> iframe 本身已经成功创建，问题不是 iframe 高度、宽度或者 CSS，而是移动浏览器没有在 iframe 中提供可嵌入的 PDF 原生查看器。

因此下面这些方案都不应该作为主要解决方案：

```html
<embed src="xxx.pdf">
<object data="xxx.pdf"></object>
<iframe src="xxx.pdf"></iframe>
```

它们最终仍然依赖浏览器自身的 PDF 能力。

---

# 3. 第一备选：PDF.js

## 3.1 PDF.js 的核心原理

浏览器不是直接打开：

```text
xxx.pdf
```

而是先打开：

```text
/pdfjs/web/viewer.html
```

然后 viewer 使用 JavaScript 下载 PDF：

```text
viewer.html
    ↓
PDF.js JavaScript
    ↓
读取 PDF 二进制
    ↓
解析 PDF
    ↓
Canvas 渲染
    ↓
用户看到 PDF
```

因此：

```text
Android 浏览器是否自带 PDF Viewer
```

已经不重要。

因为真正负责显示 PDF 的是：

```text
PDF.js
```

而不是：

```text
Android 浏览器原生 PDF Viewer
```

---

# 4. PDF.js 不需要互联网

这是这个项目非常重要的一点。

医院内网完全可以部署：

```text
http://hospital-server/
    ├── Report.aspx
    ├── ReportFiles/
    │      └── 12345.pdf
    │
    └── pdfjs/
           ├── build/
           ├── web/
           └── ...
```

用户访问：

```text
http://hospital-server/Report.aspx
```

PDF.js 文件和 PDF 文件全部从医院内网服务器加载。

不需要：

```text
Google Docs Viewer
Google Drive
第三方 PDF 在线预览服务
公网 API
CDN
```

所以它符合医院内网场景。

PDF.js 官方文档说明可以使用预构建版本，并直接部署到自己的网站。

---

# 5. PDF.js 版本选择

## 5.1 不建议直接把“最新版本”作为第一测试版本

当前 PDF.js 官方最新版本已经是较新的版本，官方也明确区分 modern build 和 legacy build。

对于普通现代浏览器，优先使用官方最新版本。

但是本项目存在一个特殊问题：

> Lenovo Android 平板使用的是厂商浏览器，Chromium 版本目前未知。

因此实际项目中不能假设它一定是最新 Chromium。

---

## 5.2 当前项目建议

第一轮建议测试：

```text
PDF.js 3.11.174
```

原因不是它是 PDF.js 官方“最佳版本”，而是：

1. 比当前 6.x 新版老很多，兼容老浏览器的实际概率更高；
2. 3.x 的 Web Viewer 结构比较成熟；
3. 对传统 WebForms + IIS 项目比较容易直接部署；
4. 不需要把整个前端工程引入现有 WebForms 项目；
5. 可以先用现成 viewer 验证 Android 平板兼容性。

如果真实 Lenovo 平板仍然无法运行，再考虑：

```text
PDF.js 2.x
```

尤其可以进一步测试带 ES5/legacy 能力的版本。

---

# 6. PDF.js 部署方式

建议不要在项目中重新编译 PDF.js。

直接使用官方预构建版本。

项目结构可以做成：

```text
WebFormsProject
│
├── Report.aspx
├── Report.aspx.cs
│
├── ReportFiles
│     ├── 10001.pdf
│     └── 10002.pdf
│
└── pdfjs
      ├── build
      ├── web
      └── ...
```

核心是保证：

```text
/pdfjs/web/viewer.html
```

可以被浏览器访问。

例如：

```text
http://192.168.1.100/pdfjs/web/viewer.html
```

---

# 7. WebForms 中修改 PDF 展示代码

当前代码大概是：

```csharp
if (image.ImageType.ToUpper().Equals(".PDF"))
{
    fileDiv += string.Format("<embed src='{0}'></embed>", imagePath);
}
else
{
    fileDiv += string.Format("<img src='{0}' />", imagePath);
}
```

修改为：

```csharp
if (image.ImageType.ToUpper().Equals(".PDF"))
{
    string pdfUrl = "/" + imagePath;

    pdfUrl = HttpUtility.UrlEncode(pdfUrl);

    fileDiv += string.Format(
        "<iframe " +
        "src='/pdfjs/web/viewer.html?file={0}' " +
        "style='display:block;width:100%;height:1000px;border:none;'>" +
        "</iframe>",
        pdfUrl);
}
else
{
    fileDiv += string.Format(
        "<img src='{0}' />",
        imagePath);
}
```

完整一点：

```csharp
if (rImages.Count > 0)
{
    sb.Append("<h3 class='help_tit'>报告文件</h3>");

    string fileDiv = string.Empty;

    foreach (var image in rImages)
    {
        if (string.IsNullOrWhiteSpace(image.ImagePath))
            continue;

        string imagePath =
            image.ImagePath.Substring(
                0,
                image.ImagePath.LastIndexOf('.'))
            + image.ImageType;

        imagePath = imagePath.TrimStart('/');

        if (image.ImageType.ToUpper().Equals(".PDF"))
        {
            string pdfUrl = "/" + imagePath;

            pdfUrl = HttpUtility.UrlEncode(pdfUrl);

            fileDiv += string.Format(
                "<iframe " +
                "src='/pdfjs/web/viewer.html?file={0}' " +
                "style='display:block;width:100%;height:1000px;border:none;'>" +
                "</iframe>",
                pdfUrl);
        }
        else
        {
            fileDiv += string.Format(
                "<img src='{0}' />",
                imagePath);
        }
    }

    sb.Append(
        string.Format(
            "<div class='help_box'>{0}</div>",
            fileDiv));
}
else
{
    sb.Append("<h3 class='help_tit'>无报告文件</h3>");
}
```

---

# 8. 为什么一定要 UrlEncode

例如 PDF 地址：

```text
/ReportFiles/2026/09/test report.pdf
```

直接拼接：

```text
viewer.html?file=/ReportFiles/2026/09/test report.pdf
```

可能出现 URL 参数解析问题。

所以：

```csharp
string pdfUrl = HttpUtility.UrlEncode("/" + imagePath);
```

然后：

```text
/pdfjs/web/viewer.html?file=...
```

更安全。

---

# 9. 最容易遇到的问题：PDF.js 和 PDF 必须尽量同源

建议：

```text
页面：
http://hospital-server/Report.aspx

PDF.js：
http://hospital-server/pdfjs/web/viewer.html

PDF：
http://hospital-server/ReportFiles/123.pdf
```

也就是：

```text
协议相同
域名相同
端口相同
```

这样可以尽量避免浏览器的：

```text
Same-Origin Policy
CORS
```

问题。

如果 PDF 放在：

```text
http://file-server/123.pdf
```

而 PDF.js 在：

```text
http://web-server/pdfjs/
```

就需要额外处理跨域。

**本项目第一阶段不要这样做。**

---

# 10. IIS / Web.config 注意事项

如果使用较新的 PDF.js，可能会遇到：

```text
.mjs
```

文件 MIME Type 问题。

可以检查 IIS 是否支持：

```xml
<configuration>
  <system.webServer>
    <staticContent>
      <mimeMap
          fileExtension=".mjs"
          mimeType="text/javascript" />
    </staticContent>
  </system.webServer>
</configuration>
```

注意：

> 如果 IIS 已经配置 `.mjs`，不要重复添加，否则可能出现 MIME 映射重复错误。

对于 3.x 版本，具体是否需要这一步取决于你下载的 PDF.js 包结构。

---

# 11. 第一阶段测试方法

不要一上来就把整个项目改完。

建议按以下顺序。

## 第一步：确认 PDF.js 自己能打开

浏览器访问：

```text
http://服务器地址/pdfjs/web/viewer.html
```

确认 Viewer 可以正常出现。

---

## 第二步：直接给 viewer 一个 PDF

例如：

```text
http://服务器地址/pdfjs/web/viewer.html?file=%2FReportFiles%2Ftest.pdf
```

如果电脑可以打开：

```text
PDF.js 正常
```

---

## 第三步：Chrome DevTools 移动设备模式

测试：

```text
Desktop
Mobile
```

---

## 第四步：真实 Lenovo Android 平板

这是最重要的一步。

因为：

```text
Chrome DevTools Mobile
```

只是模拟移动环境。

它不能完全模拟：

```text
厂商浏览器
Android WebView
Chromium 内核版本
厂商定制限制
```

所以最终必须用真实设备测试。

---

# 12. PDF.js 的优点

### ① 不依赖浏览器原生 PDF Viewer

这是本项目最大的价值。

---

### ② 不依赖互联网

全部文件可以放在：

```text
医院内网
```

---

### ③ 不需要服务器把 PDF 转成图片

PDF 本身仍然保持：

```text
PDF
```

---

### ④ 支持多页

PDF.js 可以处理：

```text
1页
10页
50页
100页
```

---

### ⑤ 支持缩放

用户可以：

```text
放大
缩小
旋转
翻页
```

---

### ⑥ 支持文本搜索等 PDF 功能

比单纯转换成图片更接近真正的 PDF 阅读器。

---

# 13. PDF.js 的缺点

主要问题是：

```text
PDF 的解析和渲染发生在 Android 平板上
```

因此如果：

```text
PDF 很大
PDF 很复杂
PDF 包含大量高清图片
PDF 页数很多
```

低性能平板可能：

```text
CPU 占用高
内存占用高
打开速度慢
滚动卡顿
```

因此需要实际设备测试。

---

# 14. 第二备选：服务器端 PDF → 图片

如果 PDF.js 在某些 Lenovo 平板上仍然不稳定，那么第二个值得做的是：

```text
PDF
 ↓
服务器解析
 ↓
转换成 PNG/JPEG
 ↓
HTML <img>
 ↓
Android 浏览器
```

例如：

```text
test.pdf
```

转换为：

```text
test_001.jpg
test_002.jpg
test_003.jpg
```

然后页面：

```html
<img src="/ReportImages/test_001.jpg">
<img src="/ReportImages/test_002.jpg">
<img src="/ReportImages/test_003.jpg">
```

此时 Android 浏览器根本不知道这是 PDF。

它只是在显示：

```text
图片
```

所以兼容性通常非常高。

---

# 15. PDF → 图片方案的优点

对于医院报告这种：

```text
只读
主要查看
不需要编辑 PDF
不需要复制文字
页数通常有限
```

这个方案其实非常值得保留。

尤其是：

```text
低配置 Android 平板
老旧 Chromium
厂商定制浏览器
```

情况下，最终渲染出来的只是：

```html
<img>
```

浏览器支持非常成熟。

---

# 16. PDF → 图片方案的缺点

服务器需要增加 PDF 渲染组件。

例如：

```text
PDFium
Docnet / PDFium
Ghostscript
ImageMagick + PDF 渲染后端
商业 PDF SDK
```

也就是说：

```text
PDF.js
```

主要消耗：

```text
Android 平板 CPU / 内存
```

而：

```text
PDF → 图片
```

主要消耗：

```text
服务器 CPU
服务器磁盘
服务器缓存
```

因此如果使用 PDF → 图片：

> **一定要做缓存。**

不能每次用户打开报告都重新把 PDF 转成图片。

---

# 17. 推荐的 PDF → 图片缓存结构

例如：

```text
ReportFiles
    └── 12345.pdf

ReportImageCache
    └── 12345
          ├── 001.jpg
          ├── 002.jpg
          ├── 003.jpg
          └── ...
```

第一次打开：

```text
PDF
 ↓
检测缓存
 ↓
没有
 ↓
转换
 ↓
保存 JPG
 ↓
返回页面
```

第二次打开：

```text
PDF
 ↓
检测缓存
 ↓
已经存在
 ↓
直接显示 JPG
```

这样服务器压力会小很多。

---

# 18. 第三种思路：PDF 转 HTML

理论上还可以：

```text
PDF
 ↓
服务器解析
 ↓
HTML
```

然后页面直接显示 HTML。

但是这个方案不建议作为主要路线。

原因是 PDF 和 HTML 的排版模型差异很大。

特别是：

```text
字体
表格
图片
定位
分页
中文字体
印章
线条
复杂报告布局
```

很容易出现：

```text
网页显示和 PDF 原始效果不一致
```

医院报告尤其不适合拿这种方案作为第一选择。

---

# 19. 第四种思路：直接让浏览器打开 PDF

就是：

```html
<a href="/test.pdf">
```

或者：

```javascript
window.location.href = "/test.pdf";
```

这个方案本身没有问题。

但是它解决的是：

```text
打开 PDF
```

不是：

```text
在当前报告页面内嵌显示 PDF
```

而且最终还是依赖：

```text
Android 浏览器自己的 PDF Viewer
```

当前项目已经验证移动环境存在兼容问题，因此不作为方案。

---

# 20. 第五种思路：Object 标签

例如：

```html
<object
    data="/test.pdf"
    type="application/pdf"
    width="100%"
    height="1000">
</object>
```

它和：

```html
<embed>
<iframe>
```

本质上仍然依赖浏览器处理：

```text
application/pdf
```

所以：

```text
Windows Chrome
```

可能正常。

但是：

```text
Android 厂商浏览器
```

仍然可能失败。

因此不建议继续投入大量时间研究 `<object>`。

---

# 21. 第六种思路：浏览器插件 / 第三方在线 PDF Viewer

例如：

```text
Google Docs Viewer
其他在线 PDF Viewer
第三方 SaaS PDF Viewer
```

本项目不考虑。

原因：

```text
医院内网
数据安全
患者报告
网络隔离
第三方依赖
```

都不适合。

---

# 22. 方案对比

| 方案 | Android兼容性 | 是否依赖浏览器原生PDF | 是否需要外网 | 服务器压力 | 平板压力 | 推荐程度 |
|---|---:|---:|---:|---:|---:|---:|
| PDF.js 自部署 | 高 | 否 | 否 | 低 | 中 | **第一备选** |
| PDF → 图片 | 很高 | 否 | 否 | 中/高 | 低 | **第二备选** |
| `<iframe>` PDF | 不确定 | 是 | 否 | 低 | 低 | 不推荐 |
| `<embed>` | 不确定 | 是 | 否 | 低 | 低 | 不推荐 |
| `<object>` | 不确定 | 是 | 否 | 低 | 低 | 不推荐 |
| PDF → HTML | 中 | 否 | 否 | 中/高 | 中 | 不推荐 |
| Google/在线 Viewer | 取决于服务 | 否 | **是** | 外部 | 中 | 不考虑 |
| 商业 PDF SDK | 高 | 否 | 否 | 视产品而定 | 视产品而定 | 可作为后续商业方案 |

---

# 23. 最终建议的实施顺序

不要同时开发多个方案。

按照下面顺序做：

```text
第一阶段
    ↓
PDF.js 自部署
    ↓
测试 PC
    ↓
测试 Chrome Mobile
    ↓
测试真实 Lenovo Android 平板
```

如果：

```text
PDF.js 正常
```

就结束。

---

如果：

```text
PDF.js 在真实设备上出现兼容问题
```

进入第二阶段：

```text
PDF → 图片
    ↓
服务器端转换
    ↓
缓存
    ↓
<img> 展示
```

---

# 24. 建议最终架构

推荐最终保留：

```text
                    报告文件
                       │
              ┌────────┴────────┐
              │                 │
             图片               PDF
              │                 │
           <img>           PDF.js Viewer
                                │
                                │
                         Android Tablet
```

如果未来发现某些非常老的设备：

```text
PDF.js
   ↓
无法运行
```

可以增加服务器端兜底：

```text
PDF
 │
 ├── 正常设备 → PDF.js
 │
 └── 老旧设备 → PDF → 图片
```

这样比继续依赖：

```text
embed
iframe
object
```

可靠得多。

---

# 25. 关于“完全不调用其他东西”的结论

如果这里的“其他东西”指：

> **不调用第三方在线服务、不跳出当前页面、不依赖浏览器自带 PDF Viewer。**

那么：

### PDF.js 是符合要求的。

因为：

```text
PDF.js 文件       → 自己服务器
PDF 文件          → 自己服务器
JavaScript        → 自己服务器
PDF 渲染          → Android 浏览器本地执行
网络              → 医院内网
```

没有第三方在线服务。

---

如果“完全不调用其他东西”进一步指：

> **连 PDF.js、PDFium、图片转换库等第三方代码都不使用，只使用浏览器自身 HTML 标签展示 PDF。**

那么在你目前这个 Android 厂商浏览器环境下，基本没有可靠的纯 HTML 方案。

因为：

```text
<iframe>
<embed>
<object>
```

最终都绕不开：

```text
浏览器自身 PDF 能力
```

而你已经实际验证了该能力在移动环境存在问题。

因此继续研究 `<embed>` / `<iframe>` / `<object>` 的意义不大。

---

# 26. 当前项目建议

最终优先级：

```text
★★★★★  PDF.js 自部署
    ↓
    第一备选

★★★★☆  PDF → 图片 + 缓存
    ↓
    第二备选

★★☆☆☆  Object / Embed / iframe
    ↓
    已经验证存在移动兼容问题，不建议继续投入

★☆☆☆☆  在线 PDF Viewer
    ↓
    医院内网环境不考虑
```

---

## 27. 下一步

建议现在不要先写 PDF → 图片。

先把：

```text
PDF.js 3.11.174
```

放进现有 WebForms 项目，然后只做一个最小测试：

```text
Report.aspx
    ↓
iframe
    ↓
/pdfjs/web/viewer.html?file=/ReportFiles/test.pdf
```

先在真实 Lenovo 平板上确认：

```text
① Viewer 能否打开
② PDF 能否加载
③ PDF 第一页能否显示
④ 多页能否滚动
⑤ 缩放是否正常
⑥ 中文字体是否正常
⑦ 图片/表格是否正常
```

确认这些之后，再把它接入你现在的：

```csharp
foreach (var image in rImages)
```

正式代码。

---

## 参考资料

1. Mozilla PDF.js 官方项目  
   https://github.com/mozilla/pdf.js/

2. Mozilla PDF.js 网站部署说明  
   https://github.com/mozilla/pdf.js/wiki/Setup-pdf.js-in-a-website

3. Mozilla PDF.js 浏览器兼容性 FAQ  
   https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions

4. Mozilla PDF.js 官方 README  
   https://github.com/mozilla/pdf.js/blob/master/README.md
