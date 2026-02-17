# 抓火鸡游戏开发经验总结

> 记录从V1到V10.4的完整开发过程中踩过的坑，供后续游戏项目参考。

---

## 一、中文字体渲染（最大的坑）

### 问题
英文游戏字体（Lilita One）不支持中文，中文回退到系统默认细体，视觉不统一。

### 尝试过的方案

| 方案 | 结果 |
|------|------|
| 直接用Google Fonts全量加载ZCOOL KuaiLe | 文件太大(2MB+)，加载慢 |
| woff2子集化（只含游戏用到的212个汉字） | ✅ 40KB，完美 |
| TTF格式嵌入 | ❌ Chromium不渲染，必须用woff2 |

### 最终方案
```bash
# 用pyftsubset提取子集
pyftsubset ZCOOLKuaiLe-Regular.ttf \
  --text="游戏中用到的所有汉字" \
  --output-file=zcool-subset.woff2 \
  --flavor=woff2
```
然后base64嵌入HTML的`@font-face`。

### 🔴 铁律
- **中文游戏必须内嵌字体**，不能依赖系统字体
- **只用woff2格式**，TTF在某些浏览器不渲染
- **子集化**：只包含实际用到的字符，控制在50KB以内

---

## 二、中文文字描边（第二大坑）

### 问题
Supercell/Candy Crush风格需要文字描边。英文字体笔画简单，`-webkit-text-stroke: 3px #000` 效果很好。但**中文笔画密集**，3px描边会填满笔画间隙，变成黑色方块。

### 失败历程

| 版本 | 做法 | 手机效果 |
|------|------|----------|
| V10 | text-stroke: 2-4px #000 | ❌ 中文变黑块 |
| V10.1 | text-stroke: 1px + 8方向2px shadow | ❌ 手机上仍然太重 |
| V10.2 | text-stroke: 0px + 8方向2px shadow | ❌ 2px shadow在小屏也太粗 |
| V10.4 | text-stroke: 0px + 8方向1-1.5px shadow | ✅ 清晰可读 |

### 最终方案
**完全不用text-stroke，用多方向text-shadow模拟描边：**

```css
/* 重要标题（大字号28px+） - 1.5px 8方向 */
.title {
  -webkit-text-stroke: 0px;
  text-shadow:
    -1.5px -1.5px 0 #000, 1.5px -1.5px 0 #000,
    -1.5px 1.5px 0 #000, 1.5px 1.5px 0 #000,
    0 -1.5px 0 #000, 0 1.5px 0 #000,
    -1.5px 0 0 #000, 1.5px 0 0 #000,
    0 3px 0 rgba(0,0,0,.6);  /* 底部3D投影 */
}

/* 正文/标签（14-18px） - 1px 4方向 */
.label {
  -webkit-text-stroke: 0px;
  text-shadow:
    -1px -1px 0 #000, 1px -1px 0 #000,
    -1px 1px 0 #000, 1px 1px 0 #000,
    0 2px 0 rgba(0,0,0,.4);
}

/* 按钮文字（有彩色背景） - 只需投影 */
.btn {
  -webkit-text-stroke: 0px;
  text-shadow: 0 2px 0 rgba(0,0,0,.4), 0 1px 3px rgba(0,0,0,.3);
}
```

### 🔴 铁律
- **中文永远不用 `-webkit-text-stroke`**，哪怕1px在小屏上也可能出问题
- **用text-shadow的多方向偏移模拟描边**，不侵入笔画内部
- **按钮上的文字不需要描边**，彩色按钮背景本身就提供了对比度
- **shadow像素值要分级**：大标题1.5px，正文1px，按钮0px只用投影
- **桌面看着OK不代表手机OK**：手机渲染像素密度不同，描边会显得更粗

---

## 三、CSS自动化修改的教训（最惨痛的坑）

### 问题
用Python脚本批量修改CSS时，正则匹配不够精确，导致：
1. `.level-card .num`、`.level-card .name`、`.level-card .stars` 三个规则被合并成一个
2. 合并后CSS只取最后一个同名属性，前面的规则全部失效
3. 关卡选择页面排版完全崩溃（grid变成单列列表）

### 根因分析
```python
# 错误做法：用regex搜`.level-card .num{...}`时，
# 匹配到了从.num开头一直到.stars结尾的整段CSS
# 因为}字符在中间出现但regex贪婪匹配跳过了

# 危险的regex
old = re.search(r'\.level-card \.num\{[^}]+\}', html)
# 如果CSS是压缩的单行，[^}]+ 会正确停在第一个}
# 但如果脚本对结果做了替换再搜索，可能产生意外合并
```

### 正确做法
```python
def safe_replace_in_rule(html, selector_pattern, prop_changes):
    """只修改CSS规则内的特定属性值，不动结构"""
    pattern = selector_pattern + r'\{([^}]+)\}'
    m = re.search(pattern, html)
    if not m:
        return html
    inner = m.group(1)
    
    for prop, new_val in prop_changes.items():
        # 只替换属性值，不增不删属性
        prop_re = re.compile(re.escape(prop) + r':[^;]+')
        if prop_re.search(inner):
            inner = prop_re.sub(f'{prop}:{new_val}', inner)
    
    return html[:m.start()] + f'{selector_pattern}{{{inner}}}' + html[m.end():]
```

### 🔴 铁律
- **永远不要用脚本重写整个CSS规则**，只修改具体属性值
- **修改后必须验证CSS规则数量没变**：
  ```python
  for sel in ['.level-card .name{', '.level-card .stars{', '.level-card .num{']:
      assert html.count(sel) == 1, f"CSS rule {sel} missing or duplicated!"
  ```
- **改完CSS后必须浏览器截图测试每个界面**，不能只看代码
- **保留好的基础版本**，出问题时能快速回退：
  ```bash
  git show <good-commit>:index.html > index_base.html
  ```

---

## 四、单文件HTML游戏的资源打包

### 问题
Telegram发送HTML文件时，外部资源（图片/字体）无法加载。必须所有内容内嵌。

### 方案
```python
# 图片 → base64 data URI
# 字体 → base64 @font-face
# 所有内容打包进一个HTML文件

# 打包脚本必须处理两种引号：
html = html.replace(f"'{orig}'", f"'{data_uri}'")   # JS字符串
html = html.replace(f'"{orig}"', f'"{data_uri}"')   # HTML属性
```

### 🔴 铁律
- 替换资源路径时**必须同时处理单引号和双引号**
- 图片需要**resize压缩**再base64（火鸡220x220，数字64x64）
- 总文件大小控制在**1.5MB以内**（Telegram限制）
- 打包后立即**验证文件能正常打开**

---

## 五、AI生图 + 背景移除

### 问题
用ComfyUI生成游戏素材后，RMBG背景移除效果取决于背景色选择。

### 失败案例
- 绿色火鸡 + 绿色背景 → RMBG无法分离（颜色太接近）
- 结果：`v3_green.png` 有91.8%绿色残留

### 🔴 铁律
- **AI生图的背景色必须和主体颜色差异大** → 最安全用**纯白背景**
- RMBG-2.0不稳定，**用RMBG-1.4**
- 生成后**自动QC检查**：计算非透明区域中残留背景色的占比

---

## 六、GitHub Pages部署

### 流程
```bash
# 1. 打包standalone
python3 build_standalone.py

# 2. 复制为index.html（Pages要求）
cp index_v9.html index.html

# 3. 提交部署
git add -f index.html
git commit -m "描述"
git push
```

### 🔴 铁律
- GitHub Pages部署有**1-2分钟延迟**
- 用户可能看到**浏览器缓存的旧版本** → 提醒强刷
- 手机Safari的缓存特别顽固，有时需要**清除网站数据**

---

## 七、开发流程规范（血泪总结）

### 每次修改必须遵循的流程：

```
1. 基于已知好的版本开始修改
2. 修改范围最小化（只改需要改的属性）
3. 修改后验证CSS结构完整性
4. 打包standalone
5. 浏览器打开测试（至少4个界面）：
   - 首页
   - 关卡选择（检查grid布局）
   - 游戏中（检查HUD + 道具标签）
   - 暂停弹窗（检查popup居中 + 按钮）
6. 全部通过后才push部署
7. 部署后curl验证线上版本是最新的
```

### 🔴 绝不能做的事
- ❌ 用脚本一次性重写大段CSS
- ❌ 改完不测试就push
- ❌ 在手机上没验证就告诉用户"已修复"
- ❌ 把多个CSS规则合并成一个（哪怕看起来是对的）
- ❌ 信任"桌面浏览器看着OK"就等于"手机OK"

---

## 八、Supercell风格设计规范（最终版）

### 色彩
```
背景：深蓝 #0D1B2A
面板：深蓝紫渐变 #1A237E → #283593 → #1565C0
按钮-主要：绿 #5CBF2A（渐变到#3D8B22）
按钮-次要：蓝 #29B6F6（渐变到#1565C0）
按钮-危险：红 #F44336（渐变到#C62828）
金色强调：#FFD740（标题、关卡名、分数）
边框：#000 黑色（Supercell标志性粗边框）
面板外框：金色 #FFD740（box-shadow实现）
```

### 文字
```
标题：白色#FFF 或 金色#FFD740，font-weight:900
正文：白色#FFF
描边：只用text-shadow，不用text-stroke
按钮文字：白色，只用投影不用描边
```

### 按钮
```css
.btn {
  border: 3px solid #000;
  border-bottom: 6px solid #000;  /* 3D厚度 */
  border-radius: 16px;            /* 比Candy Crush更硬朗 */
  box-shadow: inset 0 3px 0 rgba(255,255,255,.3);  /* 顶部高光 */
}
```

---

*最后更新：2026-02-17 V10.4*
