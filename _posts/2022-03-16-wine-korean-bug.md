---
title: 'Wine 한국어 버그 수정'
author: Alex Kwak
date: '2022-03-16 05:30 +0900'
categories: [Application]
tags: [wine]
published: true
---
### 개요

Wine의 한국어 버그는 상당히 오래된 이슈이다. 물론, `fcitx` 와 같은 입력기에서 `root` inputStyle을 사용하여 입력하는 경우, 문제는 발생하지 않는다.

하지만, 대부분의 환경이 그러하지 않으므로 필자 역시 그러므로 수정을 해보고자 한다.

보통 Linux에서 Wine을 사용하는 일반적인 목적은 카카오톡이 대다수이지 않을까... 하는 생각에서 시작하여, `riched20` 컨트롤의 소스부터 파고 내려가고자 했다.

Wine에서 `notepad`를 실행했을 경우, 한글을 입력하면 `caret`이 보이지 않지만 한글을 덮어쓰는 문제는 발생하지 않는다.

Wine에서 `riched20`을 사용하는 `wordpad` 를 실행하고, 한글을 입력하면 한글을 덮어쓰는 문제가 발생한다.

그리하여, `riched20`와 관련된 문제라 판단하여, `riched20`으로 파고 내려가면서 시작했다.

### 중복 문자 버그

`riched20`은 `WM_IME_STARTCOMPOSITION`, `WM_IME_COMPOSITION`, `WM_IME_ENDCOMPOSITION` 세 가지 문자 조합 이벤트를 처리하는데,
이는 각 문장 조합의 시작, 중간 단계, 끝의 세 단계로 나뉘어진다.

하지만, 리뷰 결과 아무런 문제를 찾지 못하였고 `riched20` 단계에서 이벤트를 발생하는 드라이버 측에서 문제가 발생한 게 틀림 없지만, 컴포넌트의 루틴이 달라 문제가 발생하지 않는 것 이라 판단했다.

`한글글`이라는 케이스와 같이, 한글의 조합이 발생한 이후 기존의 `Compostion`이 덮어써지는 것이라 판단하여 `WM_IME_ENDCOMPOSITION` 이벤트가 발생하는 로직을 찾았다.

`X11DRV_XIMLookupChars` -> `IME_SetResultString` 에서 `WM_IME_ENDCOMPOSITION` 이라는 이벤트를 발생시키도록 로직이 구성되어 있다.
하지만, `WM_IME_ENDCOMPOSITION` 이벤트 발생 이후 `myPrivate->bInComposition`을 변경해주지 않아 발생하는 문제로 판단하여 이를 수정하였다.

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
3. 완성형 글자를 입력한다.
```

하지만, `Failed` 케이스에서 볼 수 있듯이, 완성형 글자를 입력한다고 해도 이후 남은 조합형 글자가 남을 수 있다.

이는 `CompositionString` 이라는 변수에 잔재하게 되는데, 이를 사용하여 조합형 글자 이후, `CompositionString`이 있을 경우, 비워주도록 하는 로직을 작성하면 해결된다.

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
