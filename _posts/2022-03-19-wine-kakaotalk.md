---
title: 'Wine 7+ 에서의 카카오톡 설정'
author: Alex Kwak
date: '2022-03-19 04:46 +0900'
categories:
  - dev_application
tags: [wine]
published: true
---
### 설치

Ubuntu에 카카오톡을 설치를 하는 과정을 정리해보고자 함.

```shell
sudo dpkg --add-architecture i386
wget -nc https://dl.winehq.org/wine-builds/winehq.key
sudo apt-key add winehq.key

# For ubuntu 21.10
sudo add-apt-repository 'deb https://dl.winehq.org/wine-builds/ubuntu/ impish main'

sudo apt install --install-recommends winehq-devel

wget https://app-pc.kakaocdn.net/talk/win32/KakaoTalk_Setup.exe
wine ./KakaoTalk_Setup.exe

# Wine Mono 설치 하지 않음.
```

### 한글 입력 밀림

**Option 1. `inputStyle=root` 설정**

`input-style-root.reg`

```text
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\Wine\X11 Driver]
"InputStyle"="root"
```

**Option 2. Patch 적용하여 직접 빌드 및 설치**

https://github.com/korean-input/issues/issues/14


### 알림 및 파일 전송

`설정` -> `실험실` -> `전체화면에서 알림 받기` 활성화

`설정` -> `실험실` -> `카카오톡 속도 개선 (beta)` 활성화

![스크린샷](/assets/posts/2022-03-19-wine-kakaotalk/스크린샷, 2022-03-19 09-41-57.png)

- 해당 기능으로 알림 도착 시, 소리 활성화
- 알림 메세지 활성화

API 상 전체 화면 로직이 Wine 과의 문제가 있는 것으로 보임

### 남은 이슈
- `KakaoTalkShadowWnd`, `KakaoTalkEdgeWnd` 와 같은 쓰레기 창들이 발생 ( 아래 패치 사용하여 해결 )
  - `WM_SKIP_TASKBAR` 와 같은 플래그들은 제대로 설정된 것으로 보아 Parent 프로세스가 활성화된 상태라면 하위 프로세스도 활성화 되는
  것 같다. 카카오 측에서 해당 기능을 비활성화 하는 기능일 제공하지 않는 이상 해결은 어려울 듯.
  - 채팅방을 눌렀을 때, 창이 멋대로 활성화되거나 활성화되지 않거나 하는 등의 문제도 이와 같은 문제로 보임.
  - https://github.com/emotionbug/wine/commit/5f0b111529bdc6d6b7b2c78317651233ca7f54b2
  - https://github.com/emotionbug/wine/commit/f8774551750fd15c113bb464da6958d6bf919f95
  - https://github.com/emotionbug/wine/commit/91c6e4458a5660d4cca8a6f01ab6b767e5fb6b29

