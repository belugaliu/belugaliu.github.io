---
title: Aspose 操作集合
date: 2024-12-09 00:36:46
tags:
  - aspose
  - 工具
categories:
  - code
---
aspose 在搜索引擎上可搜索且正确的内容太少或者说正确的不多。用自己的经验总结帖子，随时会增加内容。

### 通用部分

#### word 单位与 px 之间的转换

word 的单位是 pt（point）css 中 `font-size`  单位 px，之间的转换与分辨率相关。默认的分辨率为 96ppi，分辨率 96dpi 是量度单位，表示每 1 英寸上的点数是 96 点（图像 ppi 值越高，画面的细节就越丰富）。所以，转换公式

```
pt * 72f/92f = px
```

#### 给图片增加透明度

图片增加透明度，需要给新画布上设置透明度后将图片内容画至画布上。其中，`BufferedImage.TYPE_INT_ARGB` 至带有透明度的 RGB。

```java
// imageInputStream 图片的输入流
BufferedImage originImageData = ImageIO.read(imageInputStream);
BufferedImage newImageData = new BufferedImage(originImageData.getWidth(),
originImageData.getHeight(), BufferedImage.TYPE_INT_ARGB);
Graphics2D g2d = newImageData.createGraphics();
g2d.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
g2d.setComposite(AlphaComposite.getInstance(AlphaComposite.DST_OVER, 不透明度));
g2d.drawImage(originImageData, 0, 0, null);
g2d.dispose();
ImageIO.write(newImageData, "jpg", 本地文件);
```

#### java 端计算字体字号的文字宽和高度（单位 px）

借由文字转图片计算文字写入 word 后应有的宽度和高度。当然，如果设置了最大宽度，只能一个字一个字加上超过最大宽度再回车换行，n 行计算 `高度 = n * line-height`。

```java
int maxLineWidth = 最大行宽度;
BufferedImage image = new BufferedImage(maxLineWidth, 3000, BufferedImage.TYPE_INT_ARGB);
Graphics2D graphics = image.createGraphics();
graphics.setFont(new Font(字体, Font.PLAIN, 字号));
FontMetrics fontMetrics = graphics.getFontMetrics();
List<String> lines = new ArrayList();
String[] words = 文本.split("")
int currentWidth = 0;
int start = 0;
int realLineMaxWidth = 0;
for (int i = 0; i < words.length; i++) {
    String word = words[i];
    int wordWidth = fontMetrics.stringWidth(word);
    if (currentWidth + wordWidth > lineMaxWidth) {
        lines.add(文本.substring(start, i));
        start = i;
        realLineMaxWidth = Math.max(realLineMaxWidth, currentWidth);
        currentWidth = wordWith;
    } else {
        currentWidth += wordWidth;
    }
}
realLineMaxWidth = Math.max(realLineMaxWidth, currentWidth);
if (start < words.length - 1) {
   lines.add(content.substring(start));
}
int lineHeight = fontMetrics.getHeight();
int totalHeight = lines.size() * lineHeight;
int lineWidth = realLineMaxWidth;
```

### aspose.word

#### 找到 word 每页页首并跳转

aspose.word 没有能够直接跳转到页首的办法，所以只能从段落集合中找出每页的首段并跳转。

```java
// fileInputStream word的输入流
Document document = new Document(fileInputStream);
DocumentBuilder docBuilder = new DocumentBuilder(document);
NodeCollection paragraphs = document.getChildNodes(NodeType.PARAGRAPH, true);
LayoutCollector layoutCollector = new LayoutCollector(document);
int pageNum = 0;
for (Paragraph paragraph : (Iterable<Paragraph>) paragraphs) {
    if (paragraph.getParentNode().getNodeType() != NodeType.BODY || pageNum == layoutCollector.getStartPageIndex(paragraph)) {
       continue;
    }
    pageNum = layoutCollector.getStartPageIndex(paragraph);
    docBuilder.moveTo(paragraph);
}
```

#### 文本伪水印效果

使用文本框搭配透明度、旋转角度并设置文本框悬浮在文字之下，来伪装成水印效果。<font color='red'>其中，注意各种间距的设置以及文本对齐方式。</font>

```java
Shape textBoxShape = docBuilder.insertShape(ShapeType.TEXT_BOX, 宽度pt, 高度pt);
textBoxShape.setBehindText(true);
textBoxShape.setRelativeVerticalPosition(RelativeVerticalPosition.PAGE);
textBoxShape.setRelativeHorizontalPosition(RelativeHorizontalPosition.PAGE);
textBoxShape.setLeft(x偏移pt);
textBoxShape.setTop(y偏移pt);
textBoxShape.setWrapType(WrapType.NONE);
textBoxShape.setStroked(false);
textBoxShape.setZOrder(10);
textBoxShape.getFill().setTransparency(1f);
TextBox textBox = textBoxShape.getTextBox();
textBox.setTextBoxWrapMode(TextBoxWrapMode.SQUARE);
textBox.setInternalMarginBottom(0);
textBox.setInternalMarginLeft(0);
textBox.setInternalMarginRight(0);
textBox.setTextBoxWrapMode(TextBoxWrapMode.NONE);
textBox.setInternalMarginTop(0);
Paragraph lastParagraph = textBoxShape.getLastParagraph();
docBuilder.moveTo(lastParagraph);
ParagraphFormat paragraphFormat = lastParagraph.getParagraphFormat();
paragraphFormat.setCharacterUnitFirstLineIndent(0);
paragraphFormat.setCharacterUnitLeftIndent(0);
paragraphFormat.setCharacterUnitRightIndent(0);
paragraphFormat.setLineUnitAfter(0);
paragraphFormat.setAlignment(ParagraphAlignment.LEFT);
paragraphFormat.setLineUnitBefore(0);
// 这样能够让行间距达到最小
paragraphFormat.setLineSpacing(固定行距为字号pt);
paragraphFormat.setLineSpacingRule(LineSpacingRule.EXACTLY);
docBuilder.getFont().setSize(字号pt);
docBuilder.getFont().getFill().setColor(文字颜色);
docBuilder.getFont().getFill().setOpacity(文字透明度);
docBuilder.write(文本);
textBoxShape.setRotation(角度);
```

#### 图片伪水印效果

使用图片框搭配透明度、旋转角度并设置文本框悬浮在文字之下，来伪装成水印效果。

```java
BufferedImage imageData = 透明图片资源;
Shape shapBox = docBuilder.insertImage(imageData, RelativeHorizontalPosition.PAGE, x偏移pt,
                    RelativeVerticalPosition.PAGE, y偏移pt, 宽度pt, 高度pt, WrapType.NONE);
shapBox.setBehindText(true);
shapBox.setZOrder(1);
shapBox.getFill().setTransparency(1f);
shapBox.setRotation(config.getAngle());
```

#### aspose.word 21.11 版本破解程序

如果有条件的话，还是不要用破解，支持支持知识产权嘛。在使用 aspose.word 中任何对象之前 static 静态代码块中完成 license 验证逻辑修改。

```java
public class CrackAsposeWord {
    static {
        registerWord();
    }
    
    public static void registerWord() {
        try {
            Class<?> zzXDbClass = Class.forName("com.aspose.words.zzXDb");
            Constructor zzXDbConstructor = zzXDbClass.getDeclaredConstructors()[0];
            zzXDbConstructor.setAccessible(true);
            Object zzXDb = zzXDbConstructor.newInstance();
            Field zzZ31 = zzXDbClass.getDeclaredField("zzZ3l");
            zzZ31.setAccessible(true);
            zzZ31.set(zzXDb, new Date(Long.MAX_VALUE));
            Field zzWSL = zzXDbClass.getDeclaredField("zzWSL");
            zzWSL.setAccessible(true);
            Class<?> zzYeQ = Class.forName("com.aspose.words.zzYeQ");
            Field zzXgr = zzYeQ.getDeclaredField("zzXgr");
            zzXgr.setAccessible(true);
            zzWSL.set(zzXDb, zzXgr.get(null));
            Field zzWiV = zzXDbClass.getDeclaredField("zzWiV");
            zzWiV.setAccessible(true);
            zzWiV.set(null, zzXDb);
            Class<?> zzZjjClassB = Class.forName("com.aspose.words.zzYKk");
            Field zzYU8 = zzZjjClassB.getDeclaredField("zzYU8");
            zzYU8.setAccessible(true);
            zzYU8.set(null, 128);
            Field zzyS = zzZjjClassB.getDeclaredField("zzyS");
            zzyS.setAccessible(true);
            zzyS.set(null, false);
        } catch(Exception e) {
            log.error("crack Aspose.word 失败", e);
        }
    }
}
```
