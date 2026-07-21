---
title: "[Note] Qt/C++로 MPEG-TS 암호화 상태 진단 도구를 만든 기록"
date: 2025-03-11 20:00:00 +0900
categories: [note]
tags: [qt, cpp, mpeg-ts, udp, multicast, rtp]
layout: single
---

영상 스트림이 들어오는데 재생이 안 되거나 암호화 상태를 확인해야 할 때, 원본 TS 파일과 실시간 멀티캐스트 패킷을 직접 살펴볼 수 있는 Qt/C++ 도구를 만들었다.

목표는 복잡한 미디어 플레이어를 만드는 것이 아니라 다음 정보를 빠르게 보는 것이었다.

- 188바이트 TS 패킷이 정상적으로 정렬돼 있는가
- PAT와 PMT에서 프로그램과 스트림 PID를 찾을 수 있는가
- 영상/음성 PID의 scrambling bit가 설정돼 있는가
- UDP 멀티캐스트와 RTP로 들어오는 스트림도 같은 기준으로 분석할 수 있는가

## TS 패킷의 시작점을 먼저 확인했다

MPEG-TS 패킷은 일반적으로 188바이트이고 첫 바이트는 `0x47`이다.

```cpp
constexpr int TS_PACKET_SIZE = 188;
constexpr uint8_t TS_SYNC_BYTE = 0x47;

if (packet[0] != TS_SYNC_BYTE) {
    return ParseResult::InvalidSyncByte;
}
```

파일 크기만 188로 나눠 떨어진다고 정상 TS라고 볼 수 없다. 중간 바이트가 유실되면 이후 패킷 경계가 모두 어긋난다. 그래서 매 패킷의 sync byte를 확인하고, 맞지 않으면 그 위치부터 분석을 중단해 잘못된 통계를 만들지 않게 했다.

## 헤더에서 PID와 scrambling bit를 분리했다

TS 헤더의 필요한 비트를 마스크 연산으로 꺼냈다.

```cpp
struct TsHeader {
    bool transportError;
    bool payloadStart;
    uint16_t pid;
    uint8_t scramblingControl;
    uint8_t adaptationFieldControl;
    uint8_t continuityCounter;
};

TsHeader parseHeader(const uint8_t* packet) {
    return {
        (packet[1] & 0x80) != 0,
        (packet[1] & 0x40) != 0,
        static_cast<uint16_t>(((packet[1] & 0x1F) << 8) | packet[2]),
        static_cast<uint8_t>((packet[3] & 0xC0) >> 6),
        static_cast<uint8_t>((packet[3] & 0x30) >> 4),
        static_cast<uint8_t>(packet[3] & 0x0F)
    };
}
```

`transport_scrambling_control`은 2비트 값이다.

| 값 | 의미 |
| --- | --- |
| `00` | scrambling 없음 |
| `01` | 예약 값 |
| `10` | even key로 scrambled |
| `11` | odd key로 scrambled |

도구에서는 PID별 전체 패킷 수와 scrambling 값이 0이 아닌 패킷 수를 같이 집계했다. 일부 패킷만 보고 전체 스트림이 암호화됐다고 단정하지 않기 위해서다.

## PAT에서 PMT를 찾고 PMT에서 실제 스트림을 찾았다

PID만 나열하면 어떤 것이 영상이고 음성인지 알기 어렵다. 그래서 먼저 PAT와 PMT를 수집했다.

```text
PAT (PID 0x0000)
  프로그램 번호 → PMT PID

PMT
  stream_type + elementary PID
  예: H.264/H.265 영상, AAC 음성
```

파일 분석은 두 번 읽도록 구성했다.

```text
1차: PAT/PMT를 읽어 프로그램과 스트림 PID 수집
2차: 수집한 PID별 패킷 수와 scrambling 상태 집계
```

한 번에 모두 처리하려고 하면 PMT를 만나기 전에 나온 패킷의 종류를 알 수 없다. 파일을 두 번 읽는 비용보다 결과를 단순하고 정확하게 만드는 쪽을 선택했다.

## Qt 화면에서는 트리와 상세 정보를 나눴다

분석 결과는 Qt의 model/view 구조로 표시했다.

```text
PAT
  PMT PID / 프로그램 번호
    stream type / elementary PID
```

왼쪽 트리에서 PMT나 스트림을 선택하면 오른쪽 목록에 다음 정보를 보여줬다.

- PID
- stream type
- 전체 패킷 수
- scrambled 패킷 수
- scrambling 비율

파일을 다시 열 때 이전 분석 결과가 남지 않도록 PAT/PMT 배열, 패킷 목록, UI model을 같이 초기화했다.

## UDP 멀티캐스트와 RTP도 같은 파서로 연결했다

파일뿐 아니라 실시간 스트림도 확인할 수 있게 `QUdpSocket`으로 멀티캐스트 그룹에 가입했다.

```cpp
socket.bind(QHostAddress::AnyIPv4,
            port,
            QUdpSocket::ShareAddress |
            QUdpSocket::ReuseAddressHint);

socket.joinMulticastGroup(groupAddress, networkInterface);
```

RTP로 전달되는 경우에는 기본 헤더를 분리한 뒤 payload를 188바이트씩 TS 파서에 넘겼다.

```cpp
QByteArray tsPayload = datagram.mid(rtpHeaderSize);

for (int offset = 0;
     offset + TS_PACKET_SIZE <= tsPayload.size();
     offset += TS_PACKET_SIZE) {
    analyzePacket(tsPayload.constData() + offset);
}
```

수신한 payload를 파일로 저장하는 기능도 넣어 실시간 문제를 나중에 다시 재현할 수 있게 했다.

실제 RTP 헤더는 CSRC, extension, padding에 따라 12바이트보다 길어질 수 있다. 처음 구현은 고정 12바이트 헤더를 가정했기 때문에, 범용 도구로 확장하려면 RTP 첫 바이트의 CC/X/P 값을 읽어 동적으로 payload 위치를 계산해야 한다는 한계도 남겼다.

## UI가 멈추지 않도록 나눈 부분

UDP 수신은 Qt signal/slot으로 처리하고, 반복 연산이 큰 보조 데이터 분석은 `QThread` 작업으로 분리했다. worker에서는 UI 위젯을 직접 바꾸지 않고 결과를 signal로 전달하도록 구성했다.

```cpp
connect(worker, &AnalyzerWorker::finished,
        this, &MainWindow::showResult);
```

네트워크 수신과 파일 분석이 같은 UI thread에서 오래 돌면 창이 응답하지 않는 것처럼 보일 수 있기 때문이다.

## 만들면서 확인한 한계

- adaptation field가 있는 패킷은 payload 시작 위치를 별도로 계산해야 한다.
- PAT/PMT section이 여러 패킷에 나뉘면 section 조립이 필요하다.
- RTP 헤더 길이를 12바이트로 고정하면 extension이 있는 스트림을 잘못 읽을 수 있다.
- scrambling bit는 암호화 여부를 보여주지만 실제 복호화 가능 여부까지 보장하지 않는다.
- 실시간 UI 갱신을 너무 자주 하면 분석보다 렌더링 비용이 커질 수 있다.

## 정리

이 도구에서 구현한 흐름은 다음과 같다.

```text
TS sync 확인
→ 헤더 비트 파싱
→ PAT/PMT로 스트림 PID 분류
→ PID별 scrambling 통계
→ Qt 트리/상세 화면 표시
→ UDP multicast와 RTP 입력 연결
→ 필요 시 원본 TS 저장
```

영상이 “안 나온다”는 결과만 보는 대신 패킷 경계, 프로그램 정보, PID, 암호화 상태를 단계별로 확인할 수 있는 진단 도구를 직접 만든 경험이었다.
