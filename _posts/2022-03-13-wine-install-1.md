---
title: 'Wine 빌드 및 설치'
author: Alex Kwak
date: '2022-03-13 23:30 +0900'
categories: [Application]
tags: [wine]
published: true
---
### 참고

- 와인 빌드 방법 및 종속성에 대한 정리 글
  - https://wiki.winehq.org/Building_Wine

### 소스 다운로드
```
mkdir -p ~/wine-build/wine32
mkdir -p ~/wine-build/wine64
mkdir -p ~/wine-build/wine32-tools
mkdir -p ~/wine-build/deploy
cd ~/wine-build
sudo apt-get install git

git clone git://source.winehq.org/git/wine.git wine-emotionbug
```


### 32 bit 빌드를 위한 Docker 준비

Wine 공식 홈페이지에서는 LXC를 사용하라고 권장하고 있지만, 여기서는 Docker를 사용하여 빌드한다.

* `emotionbug`는 필자가 쓰는 닉네임 및 계정 이름.
* $HOME 을 서로 Mount 하여, 같은 $HOME 을 사용하도록 설정하며, 같은 계정 이름 및 그룹을 생성하여 권한 문제를 방지한다.

```shell
docker run -it -v /home:/home --restart=always --name ubuntu32bit i386/ubuntu:bionic
adduser emotionbug
usermod -aG sudo emotionbug
apt-get -y install sudo ubuntu-minimal
exit
```

### 빌드 종속성 설치

```shell
sudo apt-get install build-essential gcc-multilib g++-multilib gcc-multilib flex bison libx11-dev libfreetype6-dev libfontconfig1-dev gcc-mingw-w64 libxcursor-dev libxi-dev libxext-dev libxxf86vm-dev libxrandr-dev  libxinerama-dev libxcomposite-dev libosmesa-dev ocl-icd-opencl-dev libpcap-dev libdbus-1-dev libsane-dev libusb-1.0-0-dev libv4l-dev libgphoto2-dev libpulse-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev oss4-dev libudev-dev libsdl2-dev libcapi20-dev libcups2-dev libkrb5-dev libopenal-dev samba-dev libvulkan-dev libldap2-dev libgnutls28-dev gettext libgettextpo-dev
docker exec -it ubuntu32bit su - emotionbug
sudo apt-get install build-essential gcc-multilib g++-multilib gcc-multilib flex bison libx11-dev libfreetype6-dev libfontconfig1-dev gcc-mingw-w64 libxcursor-dev libxi-dev libxext-dev libxxf86vm-dev libxrandr-dev  libxinerama-dev libxcomposite-dev libosmesa-dev ocl-icd-opencl-dev libpcap-dev libdbus-1-dev libsane-dev libusb-1.0-0-dev libv4l-dev libgphoto2-dev libpulse-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev oss4-dev libudev-dev libsdl2-dev libcapi20-dev libcups2-dev libkrb5-dev libopenal-dev samba-dev libvulkan-dev libldap2-dev libgnutls28-dev gettext libgettextpo-dev
exit
```

## 64-bit version 빌드 ( Host )

```shell
cd ~/wine-build/wine64
../wine-emotionbug/configure --enable-win64 --prefix=$HOME/wine-build/deploy
make -j16
```

## 32-bit wine tools 빌드 ( Container )

```shell
docker exec -it ubuntu32bit su - emotionbug
cd ~/wine-build/wine32-tools
../wine-emotionbug/configure --prefix=$HOME/wine-build/deploy
make -j16
exit
```

## 64-bit 와 32-bit 빌드 결합 ( Container )

```shell
docker exec -it ubuntu32bit su - emotionbug
cd ~/wine-build/wine32
../wine-emotionbug/configure --with-wine64=$HOME/wine-build/wine64 --with-wine-tools=$HOME/wine-build/wine32-tools --prefix=$HOME/wine-build/deploy
make -j16
exit
```

## 설치 ( Host )

* prefix로 지정한 `$HOME/wine-build/deploy` 경로에 설치(실행에 필요한 바이너리들이 위치하게) 된다.

```shell
cd ~/wine-build/wine32
make install
cd ~/wine-build/wine64
make install
```
