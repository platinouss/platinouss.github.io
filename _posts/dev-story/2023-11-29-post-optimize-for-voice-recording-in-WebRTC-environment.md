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
변환 작업은 음성을 중계하고 있는 미디어 서버에서 진행해야했고, 현재 미디어 서버는 Node.js 환경에 TypeScript 언어로 구현되고 있다.

위 환경에서 녹음 및 변환 기능을 **구현한 과정**과 **진행하며 겪었던 문제**를 기술해보려고 한다.

> 현재 node-webrtc 모듈이 TypeScript를 지원하고 있지 않아서, 필자가 해당 모듈 레포지토리 issue를 참고하여 타입 정의 파일을 만들었습니다. [**node-webrtc 모듈 Typescript 언어로 빌드하기**](https://github.com/boostcampwm2023/web13_Boarlog/issues/28)를 참고하여 타입 정의 파일을 추가해주면 TypeScript로도 진행할 수 있습니다.

먼저 발표자가 송신한 실시간 음성 원시 데이터를 가져오는 방법을 살펴보자.

## 실시간으로 수신된 음성 샘플 데이터에 접근하기
현재 미디어 서버는 Chromium WebRTC를 직접 사용하는 [**node-webrtc 모듈**](https://github.com/node-webrtc/node-webrtc)을 기반으로 구동된다.  
이 모듈에서는 몇 가지 [**비표준 API**](https://github.com/node-webrtc/node-webrtc/blob/develop/docs/nonstandard-apis.md)를 제공하는데, 그 중 하나는 실시간으로 수신되고 있는 peer의 음성 원시 데이터를 샘플 단위로 가져올 수 있는 API이다.  

```typescript
[constructor(MediaStreamTrack track)]
interface RTCAudioSink: EventTarget {
  void stop();
  readonly attribute boolean stopped;
  attribute EventHandler ondata;
};
``` 
음성 샘플 데이터를 얻기 위해서는 node-webrtc 모듈에서 제공하는 RTCAudioSink를 활용하면 되는데, RTCPeerConnection을 통해 얻은 MediaStreamTrack을 인자로 넘겨 생성할 수 있다. (비디오는 RTCVideoSink로 가능하다)  

> RTCAudioSink가 어떤 로직으로 WebRTC를 통해 수신되는 음성 샘플 데이터를 가로채오는지 분석 예정입니다.

RTCAudioSink는 샘플 단위의 음성 데이터를 수신 받을 때마다, RTCAudioData 타입의 데이터와 함께 'data' 이벤트가 발생한다. 이 이벤트를 핸들링하기 위해서는 RTCAudioSink의 ondata 필드에 이벤트 핸들러 함수를 설정하면 된다.

그럼 RTCAudioData에는 어떤 정보들이 담기는지 알아보자.

```typescript
export interface RTCAudioData {
  samples: Int16Array;
  sampleRate: number;
  bitsPerSample?: 16;
  channelCount?: 1;
  numberOfFrames?: number;
}
```
RTCAudioData에는 Int16Array 타입의 **샘플 단위의 음성 데이터(samples)**와 **sample rate 정보(sampleRate)**가 담긴다.

다음으로는 샘플 단위의 음성 데이터를 병합하는 과정을 살펴보자.

## 수신된 음성 원시 데이터들을 병합하기
병합 시에 유의해야 할 점은 지속적인 메모리 점유에 유의해야한다. 만약 음성 원시 데이터의 총 크기가 1GB라고 가정한다면, 자칫하다가는 병합 과정에서 1GB의 메모리를 음성 녹음에 사용하게 될 수 있다.  

따라서 버퍼 같은 곳에 음성 원시 데이터를 저장해두고, 이후 버퍼가 특정 크기에 도달하면, 담겨 있던 데이터를 디스크의 파일에 쓰고 해당 버퍼를 비워주는 식으로 메모리 점유율을 줄여야한다.

Node.js에서는 [**Stream**](https://nodejs.org/api/stream.html)을 제공하는데, Stream은 작은 버퍼를 사용하여 데이터를 일부분씩 처리하기 때문에 전체 데이터를 한 번에 메모리에 로드하지 않게된다. (버퍼의 기본 크기는 16KiB 이지만, highWaterMark 옵션으로 버퍼의 크기를 설정할 수 있다)

하지만 이 버퍼를 비우기 위해서는 버퍼에 담긴 데이터를 어딘가에 써야하는데, 이때 사용하는 것이 pipeline이다.
```typescript
pipeline(
  fs.createReadStream('archive.tar'),
  zlib.createGzip(),
  fs.createWriteStream('archive.tar.gz'),
  (err) => {
    if (err) {
      console.error('Pipeline failed.', err);
    } else {
      console.log('Pipeline succeeded.');
    }
  },
); 
```
위 코드는 pipeline에 ReadStream과 WriteStream을 각각 첫 번째와 마지막 인자로 넘겨, WriteStream이 ReadStream의 데이터를 받아 처리하도록 연결하는 부분이다.

> Read Stream은 일정 단위로 특정 데이터를 읽을 때 사용하며, Write Stream은 특정 대상에 데이터를 쓸 때 사용합니다. 또한 Transform Stream(PassThrough)은 읽기와 쓰기가 모두 가능한 스트림을 뜻합니다.

이렇게 되면 ReadStream에 있던 데이터가 WriteStream에 전달되면서 버퍼가 비워지게되고, ReadStream에 지속적으로 데이터가 추가되더라도 메모리보다 큰 파일을 읽을 수 있게된다. 

다시 이 글의 주제로 돌아와서, 어떤식으로 음성 원시 데이터를 병합 해야 최소한의 메모리를 사용할 수 있을까? 위에서 언급했던 Node.js의 Stream을 사용한다면 된다. 

```typescript
const passThrough = new PassThrough();
pipeline(
  passThrough,
  fs.createWriteStream("병합 파일 경로"),
  (err) => {
    if (err) {
      console.log(err);
    }
  }
);
```
RTCAudioSink의 ondata 필드에 등록한 이벤트 핸들러를 통해 RTCAudioData를 받으면, RTCAudioData의 samples 필드 데이터만 Transform Stream인 PassThrough에 추가한다. 

이후 pipeline을 사용하여, Transform Stream을 Write Stream과 연결하면 메모리 사용률을 최소화하고 원시 데이터들을 하나의 파일로 병합할 수 있게된다.

## 음성 원시 데이터 파일을 오디오 파일로 변환하기
병합한 원시 데이터 파일을 mp3와 같은 오디오 파일로 변환하기 위해서는 ffmpeg 모듈이 필요하다.  
Node.js에는 ffmpeg를 사용할 수 있는 [**node-fluent-ffmpeg**](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg) 라이브러리가 존재한다.  


```typescript
ffmpeg(fs.createReadStream("병합된 원시 데이터 파일 경로"))
  .addInputOptions('-f s16le', '-ar 48k', '-ac 1') // 16비트로 인코딩 된 오디오 데이터, 샘플링 레이트, 오디오 채널 수
  .format('mp3')
  .audioCodec('libmp3lame')
  .on('start', () => {
    ...
  })
  .on('error', (err) => {
    ...
  })
  .on('end', async () => {
    ...
  })
  .pipe(fs.createWriteStream("변환된 mp3 파일 저장 경로"), { end: true });
```
변환 과정에서도 원시 파일을 읽고 변환된 파일을 저장해야 하기 때문에, 메모리를 효율적으로 사용하기 위해서는 Read Stream과 Write Stream을 사용해야 한다. 

addInputOptions은 Read Stream을 통해 읽는 음성 원시 데이터의 오디오 정보를 설정하는데 사용하는 옵션이다. format과 audioCodec은 단어 의미 그대로 출력 format과 audio 코덱을 설정하는 옵션이다.

위와 같이 ffmpeg command를 설정한다면, WebRTC 기반으로 수신된 음성 샘플 데이터를 오디오 파일로 변환할 수 있게된다.

> ffmpeg가 어떻게 오디오 파일로 변환시키는지 조사해볼 예정입니다.

## 참고
- [**https://github.com/node-webrtc/node-webrtc/blob/develop/docs/nonstandard-apis.md**](https://github.com/node-webrtc/node-webrtc/blob/develop/docs/nonstandard-apis.md)  
- [**https://nodejs.org/api/stream.html**](https://nodejs.org/api/stream.html)  
- [**https://github.com/fluent-ffmpeg/node-fluent-ffmpeg/blob/master/README.md**](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg/blob/master/README.md)  