---
title: Aspose 如何去掉官方水印（crack）
date: 2024-12-14 23:19:55
tags:
  - aspose
  - 工具
categories:
  - code
---

如果你资金不紧张的话，还是支持正版比较好。因为支持原创才是王道，做原创且用且珍惜。仅针对资金不足也不用于商业用途的情况下，可以看向这里。

### 基本思路

aspose 代码做了混淆，基本属于看不懂具体的实现。因其使用 SDK 单机授权校验，所以还是有迹可循。通过各个组件的 `new License().setLicense("licenseText")` 方式来找到入口。读通 `setLicense` 逻辑看清楚最后设置了哪些地方的哪些值。看完后会发现，大都会是某些类的 static 属性值对象，具体的请靠我的实现参（si）透（kao）。

### aspose 各组件去官印

#### aspose.words

```java
public static void registerWords_21_11() {
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
            log.error("crack Aspose.word 21.11 失败", e);
        }
    }
```

#### aspose.pdf

```java
public static void registerPdf_21_8_1() {
        try {
            Class<?> l9nClass = Class.forName("com.aspose.pdf.l9n");
            Constructor l9nConstructor = l9nClass.getDeclaredConstructors()[0];
            l9nConstructor.setAccessible(true);
            Object l9n = l9nConstructor.newInstance();

            Date maxDate = new Date(Long.MAX_VALUE);
            Field lcField = l9nClass.getDeclaredField("lc");
            lcField.setAccessible(true);
            lcField.set(l9n, maxDate);
            Field lyField = l9nClass.getDeclaredField("ly");
            lyField.setAccessible(true);
            lyField.set(l9n, maxDate);

            Field l0ifField = l9nClass.getDeclaredField("l0if");
            l0ifField.setAccessible(true);
            Class<?> l9kClass = Class.forName("com.aspose.pdf.l9k");
            Field lfField = l9kClass.getDeclaredField("lf");
            lfField.setAccessible(true);
            Object lf = lfField.get(null);
            l0ifField.set(l9n, lf);

            Class<?> l9nLfClass = Class.forName("com.aspose.pdf.l9n$lf");
            Field l9nliField = l9nLfClass.getDeclaredField("lI");
            l9nliField.setAccessible(true);
            l9nliField.set(null, l9n);

            Class<?> l19hClass = Class.forName("com.aspose.pdf.l19h");
            Field liField = l19hClass.getDeclaredField("lI");
            liField.setAccessible(true);
            liField.set(null, 128);
            Field l19hLfField = l19hClass.getDeclaredField("lf");
            l19hLfField.setAccessible(true);
            l19hLfField.set(null, false);
        } catch(Exception e) {
            log.error("crack Aspose.pdf 21.8.1 失败", e);
        }
    }
```

#### aspose.slides

```java
public static void registerPPT_21_11() {
        try {
            Class<?> publicClass = Class.forName("com.aspose.slides.internal.oh.public");
            Constructor<?> publicConsurctor = publicClass.getDeclaredConstructors()[0];
            publicConsurctor.setAccessible(true);
            Object publicObject = publicConsurctor.newInstance();

            Date maxDate = new Date(Long.MAX_VALUE);
            Field intField = publicClass.getDeclaredField("int");
            intField.setAccessible(true);
            intField.set(publicObject, maxDate);
            Field newField = publicClass.getDeclaredField("new");
            newField.setAccessible(true);
            newField.set(publicObject, maxDate);

            Field tryField = publicClass.getDeclaredField("try");
            tryField.setAccessible(true);
            tryField.set(null, publicObject);

            Field ifField = publicClass.getDeclaredField("if");
            ifField.setAccessible(true);
            ifField.set(publicObject, 2);

            Class<?> nativeClass = Class.forName("com.aspose.slides.internal.oh.native");
            Field nativeDo = nativeClass.getDeclaredField("do");
            nativeDo.setAccessible(true);
            nativeDo.set(null, publicObject);
        } catch(Exception e) {
            log.error("crack Aspose.slide 21.12 失败", e);
        }

    }
```

#### aspose.imaging

```java
public static void registerImage_21_12() {
        try {
            Class<?> brClass = Class.forName("com.aspose.imaging.internal.at.bR");
            Constructor<?> brConstructor = brClass.getDeclaredConstructors()[0];
            brConstructor.setAccessible(true);
            Object brObject = brConstructor.newInstance();
            Class<?> anClass = Class.forName("com.aspose.imaging.internal.at.aN");
            Field aField = anClass.getDeclaredField("a");
            aField.setAccessible(true);
            aField.set(null, brObject);

            Field bField = brClass.getDeclaredField("b");
            bField.setAccessible(true);
            bField.set(null, brObject);


            Class<?> apClass = Class.forName("com.aspose.imaging.internal.at.aP");
            Constructor<?> apConstructor = apClass.getDeclaredConstructors()[0];
            apConstructor.setAccessible(true);

            Field eField = apClass.getDeclaredField("e");
            eField.setAccessible(true);
            Object apObject = eField.get(null);

            Class<?> aOClass = Class.forName("com.aspose.imaging.internal.at.aO");
            Field cField = aOClass.getDeclaredField("c");
            cField.setAccessible(true);
            aO[] aOs = (aO[])cField.get(null);

            Field apaField = apClass.getDeclaredField("a");
            apaField.setAccessible(true);
            apaField.set(apObject, aOs[1]);

            Date maxDate = new Date(Long.MAX_VALUE);
            Field dField = apClass.getDeclaredField("d");
            dField.setAccessible(true);
            dField.set(apObject, maxDate);

        } catch(Exception e) {
            log.error("crack Aspose.imaging 21.12 失败", e);
        }
    }
```

#### aspose.html

```java
public static void registerHtml_21_12() {
        try {
            Class<?> z17Class = Class.forName("com.aspose.html.z17");
            Constructor<?> constructor = z17Class.getDeclaredConstructors()[0];
            Object z17 = constructor.newInstance();

            Date maxDate = new Date(Long.MAX_VALUE);
            Field m195Field = z17Class.getDeclaredField("m195");
            m195Field.setAccessible(true);
            m195Field.set(z17, maxDate);
            Field m196Field = z17Class.getDeclaredField("m196");
            m196Field.setAccessible(true);
            m196Field.set(z17, maxDate);

            Field m197Field = z17Class.getDeclaredField("m197");
            m197Field.setAccessible(true);
            m197Field.set(z17, z18.m208);

            Class<?> z2Class = Class.forName("com.aspose.html.z17$z2");
            Field z2Field = z2Class.getDeclaredField("m201");
            z2Field.setAccessible(true);
            z2Field.set(null, z17);
        } catch(Exception e) {
            log.error("crack aspose.html 失败", e);
        }
    }
```

#### aspose.cells

```java
public static void registerExcel_21_12() {
        try {
            String licenseExpiry = "20991231";
            Class<?> licenseClass = Class.forName("com.aspose.cells.License");
            Field aField = licenseClass.getDeclaredField("a");
            aField.setAccessible(true);
            aField.set(null, licenseExpiry);

            Class<?> zaukClass = Class.forName("com.aspose.cells.zauk");
            Constructor<?> zaukConstructor = zaukClass.getDeclaredConstructors()[0];
            zaukConstructor.setAccessible(true);
            Object zauk = zaukConstructor.newInstance();
            Field zaField = zaukClass.getDeclaredField("a");
            zaField.setAccessible(true);
            zaField.set(null, zauk);

            Field zcField = zaukClass.getDeclaredField("c");
            zcField.setAccessible(true);
            zcField.set(zauk, licenseExpiry);

            Class<?> zbmgClass = Class.forName("com.aspose.cells.zbmg");
            Field zbaField = zbmgClass.getDeclaredField("a");
            zbaField.setAccessible(true);
            zbaField.set(null, false);
        } catch(Exception e) {
            log.error("crack Aspose.excel 21.12 失败", e);
        }
    }
```
