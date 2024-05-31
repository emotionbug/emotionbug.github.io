---
title: 'Wine 한국어 버그 수정'
author: Alex Kwak
date: '2022-03-16 05:30 +0900'
categories: [Application]
tags: [wine]
published: true
---
### 개요

Wine의 한국어 버그는 상당히 오래된 이슈이다. 물론, `fcitx` 와 같은 입력기에서 `inputStyle = root`을 사용하여 입력하는 경우, 문제는
발생하지 않는다.

하지만, 일반적으로 `inputStyle = root` 방식의 입력기나 설정을 사용하지 않는 이용자가 대다수이며이기 때문에 이를 수정하고자 한다.

버그는 총 두가지로 나뉘어 볼 수 있다.
- `notepad`를 실행하고 텍스트를 입력했을 때 `caret`이 보이지 않는 문제
- `wordpad`( `riched20` )를 사용했을 때 글자가 밀리고, 중복입력 되는 현상
  - `notepad`의 경우와 중복되는 현상도 발생

위의 문제를 바탕으로 `riched20` 부터 문제가 있을 것으로 판단하여, `riched20`으로 파고 내려가면서 시작했다.

### 중복 문자 버그

`riched20`은 `WM_IME_STARTCOMPOSITION`, `WM_IME_COMPOSITION`, `WM_IME_ENDCOMPOSITION` 세 가지 문자 조합 이벤트를 처리하는데,
이는 각 문장 조합의 시작, 중간 단계, 끝의 세 단계로 나뉘어진다.

하지만, 리뷰 결과 아무런 문제를 찾지 못하였고 `riched20` 단계에서 이벤트를 발생하는 드라이버 측에서 문제가 발생한 게 틀림 없지만,
컴포넌트의 루틴이 달라 문제가 발생하지 않는 것 이라 판단했다.

`한글글`이라는 케이스와 같이, 한글의 조합이 발생한 이후 기존의 `Compostion`이 덮어써지는 것이라 판단하여 `WM_IME_ENDCOMPOSITION`
이벤트가 발생하는 로직을 찾았다.

`X11DRV_XIMLookupChars` -> `IME_SetResultString` 에서 `WM_IME_ENDCOMPOSITION` 이라는 이벤트를 발생시키도록 로직이 구성되어
있다. 하지만, `WM_IME_ENDCOMPOSITION` 이벤트 발생 이후 `myPrivate->bInComposition`을 변경해주지 않아 발생하는 문제로 판단하여
이를 수정하였다.

```diff
Index: dlls/winex11.drv/ime.c
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/dlls/winex11.drv/ime.c b/dlls/winex11.drv/ime.c
--- a/dlls/winex11.drv/ime.c	(revision 842452d4e79b20c42d2b7e279063b2feabeb31de)
+++ b/dlls/winex11.drv/ime.c	(date 1647366402111)
@@ -1049,10 +1049,9 @@
     GenerateIMEMessage(imc, WM_IME_COMPOSITION, 0, GCS_COMPSTR);
     GenerateIMEMessage(imc, WM_IME_COMPOSITION, lpResult[0], GCS_RESULTSTR|GCS_RESULTCLAUSE);
     GenerateIMEMessage(imc, WM_IME_ENDCOMPOSITION, 0, 0);
+    myPrivate->bInComposition = FALSE;

-    if (!inComp)
-        ImmSetOpenStatus(imc, FALSE);
-
+    ImmSetOpenStatus(imc, FALSE);
     ImmUnlockIMC(imc);
 }
```

### 이전 글자 완성 이후 다음 글자가 보이지 않는 버그

다음으로 존재하는 버그의 케이스는 아래와 같다.

**Failed**
```text
입력기 상태 = 한
사용자 ㄱ 입력
입력기 상태 = 한
사용자 ㅡ 입력
입력기 상태 = 한그
```

**Expected**
```text
입력기 상태 = 한
사용자 ㄱ 입력
입력기 상태 = 한ㄱ
사용자 ㅡ 입력
입력기 상태 = 한그
```

와인의 코드를 리딩하면서 확인한 텍스트 입력 로직은 아래와 같다.
```text
1. 조합 글자를 입력한다.
2. 입력 중, 완성형 글자가 발생한다.
3. 완성형 글자를 입력한다. ( 이는, 조합되는 중인 글자를 초기화 하고 결과만을 남긴다. )
```

`Failed` 케이스와 같은 현상이 일어나는 이유는 Wine의 로직이 조합중인 글자를 초기화 하고 결과만을 남기기 때문인데, 조합형 글자가 입력된
후 다시 `CompositionString`에 남은 미완된 조합 글자를 다시 입력해주도록 하는 로직을 작성하면 해결된다.

( 다른 CJK IME의 올바르지 않은 동작 방지를 위하여 Korean에 한해 이 동작을 제한해야 할 필요가 있는 것 같다. )

```diff
Index: dlls/winex11.drv/xim.c
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/dlls/winex11.drv/xim.c b/dlls/winex11.drv/xim.c
--- a/dlls/winex11.drv/xim.c	(revision 842452d4e79b20c42d2b7e279063b2feabeb31de)
+++ b/dlls/winex11.drv/xim.c	(date 1647366170601)
@@ -117,6 +117,13 @@

     IME_SetResultString(wcOutput, dwOutput);
     HeapFree(GetProcessHeap(), 0, wcOutput);
+
+    // After then if `CompositionString` is remaining, flushing it. i.e., Korean
+    if (CompositionString)
+    {
+        IME_SetCompositionString(SCS_SETSTR, CompositionString,
+                                 dwCompStringLength, NULL, 0);
+    }
 }
 ```

### riched20 의 버그

아래의 패치는 `Composition` 상태 중 `Selection` 을 지우는 버그와 `Composition` 초기화 중, 선택된 범위가 있을 경우 글자가 깨지는 문제를 수정한다.

```diff
Index: dlls/riched20/editor.c
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/dlls/riched20/editor.c b/dlls/riched20/editor.c
--- a/dlls/riched20/editor.c	(revision 842452d4e79b20c42d2b7e279063b2feabeb31de)
+++ b/dlls/riched20/editor.c	(date 1647365594802)
@@ -4097,8 +4097,8 @@
     return 0;
   case WM_IME_STARTCOMPOSITION:
   {
-    editor->imeStartIndex=ME_GetCursorOfs(&editor->pCursors[0]);
     ME_DeleteSelection(editor);
+    editor->imeStartIndex=ME_GetCursorOfs(&editor->pCursors[0]);
     ME_CommitUndo(editor);
     ME_UpdateRepaint(editor, FALSE);
     return 0;
@@ -4109,7 +4109,6 @@

     ME_Style *style = style_get_insert_style( editor, editor->pCursors );
     hIMC = ITextHost_TxImmGetContext(editor->texthost);
-    ME_DeleteSelection(editor);
     ME_SaveTempStyle(editor, style);
     if (lParam & (GCS_RESULTSTR|GCS_COMPSTR))
     {
```

### 후기

위의 패치로 한글 입력 문제가 정상적으로 수정된 것이라 보였다. 딱히, 디버깅이나 코드를 읽으면서 느끼는 감상은 없었다.

이후 버전에서 계속 이를 패치하여 사용할 수는 없는 노릇이니 wine-devel 에 패치를 반영해보고자 regression test 진행 후,
이를 wine-devel@winehq.org 로 Submitting 하였다.

https://www.winehq.org/pipermail/wine-devel/2022-March/211149.html

간단한 Code convention 과 양식을 지켜서 TestBot 통과와 Review 단계를 거쳐, Commit 으로 이루어지는 형태였는데 오픈 소스에 내 이름을
세긴다는 것이 상당히 나에게 흥분되는 일이었는지 오히려 이 과정이 정말 재밌게 느껴졌다.

~~(Mail을 카카오 웹으로 전송했더니 HTML 깨지는 일/C++ Style의 주석을 썼다고 빠꾸 때문에 v3가 되긴 했지만)~~
