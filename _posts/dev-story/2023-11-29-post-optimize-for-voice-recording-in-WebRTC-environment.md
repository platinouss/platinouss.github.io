---
title:  "WebRTC 기반 환경에서 최소한의 메모리 사용으로 녹음하기"
excerpt: "WebRTC로 실시간 음성을 송신하는 상황 속에서, 메모리 사용률을 고려하여 음성 샘플 데이터를 오디오 파일로 변환해보자."

categories:
  - dev-story
tags:
  - WebRTC
  - Node.js Stream

toc: true

date: 2023-11-29
last_modified_at: 2023-11-29
---

## 개요
다시보기 서비스를 제공하기 위해서, WebRTC를 통해 실시간으로 수신되는 발표자의 음성 원시 데이터를 병합하여 오디오 파일로 변환해야했다.
변환 작업은 음성을 중계하고 있는 미디어 서버에서 진행해야했고, 미디어 서버는 Node.js 환경에 TypeScript 언어로 구현하고 있었다.

위 환경에서 녹음 및 변환 기능을 **구현한 과정**과 **진행하며 겪었던 문제**를 기술해보려고 한다.

> 현재 node-webrtc 모듈이 TypeScript를 지원하고 있지 않아서, 필자가 해당 모듈 레포지토리 issue를 참고하여 타입 정의 파일을 만들었습니다. [**node-webrtc 모듈 Typescript 언어로 빌드하기**](https://github.com/boostcampwm2023/web13_Boarlog/issues/28)를 참고하여 타입 정의 파일을 추가해주면 TypeScript로도 진행할 수 있습니다.

먼저 발표자가 송신한 실시간 음성 원시 데이터를 가져오는 방법을 살펴보자.

## 실시간으로 수신된 음성 샘플 데이터에 접근하기

> [**node-webrtc - RTCAudioSink**](https://github.com/node-webrtc/node-webrtc/blob/develop/docs/nonstandard-apis.md#rtcaudiosink)

## 수신된 음성 샘플 데이터 저장하기


## 오디오 파일로 변환하기


## 참고
- [**https://github.com/node-webrtc/node-webrtc/blob/develop/docs/nonstandard-apis.md**](https://github.com/node-webrtc/node-webrtc/blob/develop/docs/nonstandard-apis.md)  
- [**https://nodejs.org/api/stream.html**](https://nodejs.org/api/stream.html)  
- [**https://github.com/fluent-ffmpeg/node-fluent-ffmpeg/blob/master/README.md**](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg/blob/master/README.md)  