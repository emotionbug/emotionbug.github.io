---
title: 'Wine 7+ 에서의 카카오톡 설정'
author: Alex Kwak
date: '2022-03-19 04:46 +0900'
categories: [Application]
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


### 상단 포커싱 알림은 발생하나, 메세지 알림이 발생하지 않는 경우

`설정` -> `실험실` -> `전체화면에서 알림 받기` 비활성화

- 해당 기능으로 알림 도착 시, 소리 활성화
- 알림 메세지 활성화

API 상 전체 화면 로직이 Wine 과의 문제가 있는 것으로 보임

### 남은 이슈
- `KakaoTalkShadowWnd`, `KakaoTalkEdgeWnd` 와 같은 쓰레기 창들이 발생
  - `WM_SKIP_TASKBAR` 와 같은 플래그들은 제대로 설정된 것으로 보아 Parent 프로세스가 활성화된 상태라면 하위 프로세스도 활성화 되는
  것 같다. 카카오 측에서 해당 기능을 비활성화 하는 기능일 제공하지 않는 이상 해결은 어려울 듯.
  - 채팅방을 눌렀을 때, 창이 멋대로 활성화되거나 활성화되지 않거나 하는 등의 문제도 이와 같은 문제로 보임.
- 파일 첨부 기능
  - `WinHttp` 라이브러리를 사용하는 듯으로 보이는데, API 단을 확인해서 Http 의 `Response` 를 확인하거나 하는 등의 시도를 해보면,
  원인 파악이 가능할 듯.
