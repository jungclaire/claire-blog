---
title: "가상 면접 사례로 배우는 대규모 시스템 설계 기초 - 15장 : 구글 드라이브 설계"
date: 2022-04-17T18:04:00+09:00
draft: false
---

이 책은 일요일마다 스터디를 시작하게 되면서 읽어보게 되었다.
오늘 스터디에서는 15장을 읽었고, 이 방법을 이용해서 읽어보니 신기하게 읽었다. 삼색 볼펜으로 읽는 것이다.
이 책뿐 아니라 다른 책들도 이렇게 읽어봐야겠다는 생각이 든다. 내용이 꽤 기억에 많이 남게 되는 방법 같다.

http://m.egloos.zum.com/agile/v/3684946

오늘은 15장을 읽었는데, 읽은 내용을 한번 더 복귀하기 위해 글로 한번 더 정리해본다.

나는 구글 드라이브를 내가 설계해서 만든다는 느낌으로 읽어보면서 복기해봤다.



구글 드라이브는 파일 저장 및 동기화 서비스로 문서, 사진, 비디오, 기타 파일을 클라우드에 보관할 수 있도록 한다.

필요한 기능은 파일 업로드 및 다운로드, 파일 동기화, 그리고 알림이었다. 파일을 암호화하고 크기에는 10GB 제한이 있다. DAU 기준 천만명일 때 어떻게 설계해보면 좋을지를 생각해볼 수 있었다.



1단계 - 문제 이해 및 설계 범위 확정
[기능적 요구사항]

- 파일 추가

- 파일 다운로드

- 여러 단말에 파일 동기화

- 파일 갱신 이력 조회

- 파일 공유

- 파일 편집, 삭제, 새롭게 공유 시 알림으로 표시



[비기능적 요구사항]

- 안정성: 저장소 시스템에서 안정성은 아주 중요하다. 데이터 손실은 발생하면 안 된다.

- 빠른 동기화 속도: 파일 동기화에 시간이 너무 많이 걸리면 사용자는 인내심을 잃고 해당 제품을 더 이상 사용하지 않게 될 것이다.

- 네트워크 대역폭: 이 제품이 네트워크 대역폭을 불필요하게 많이 소모한다면 사용자는 좋아하지 않을 것이다. 모바일 데이터 플랜을 사용하는 경우라면 더욱 그렇다.

- 규모 확장성: 이 시스템은 아주 많은 양의 트래픽도 처리 가능해야 한다.

- 높은 가용성: 일부 서버에 장애가 발생하거나, 느려지거나, 네트워크 일부가 끊겨도 시스템은 계속 사용 가능해야 한다.



2단계 - 개략적 설계안 제시 및 동의 구하기
API

1. 파일 업로드 API

- 단순 업로드: 파일 크기가 작을 때 사용한다.

- 이어 올리기 (resumable upload): 파일 사이즈가 크고 네트워크 문제로 업로드가 중단될 가능성이 높다고 생각되면 사용한다.

*업로드에 장애가 발생하면 장애 발생 시점부터 업로드를 재시작



2. 파일 다운로드 API



3. 파일 갱신 히스토리 API



* 모든 API는 사용자 인증을 필요로 하고 HTTPS 프로토콜을 사용해야 한다. (클라이언트와 백엔드 서버가 주고받는 데이터 보호)



한 대 서버의 제약 극복

파일시스템이 가득 찰 경우 해결책: 데이터를 샤딩(sharding) 하여 여러 서버에 나누어 저장



S3 서비스

데이터를 다중화 할 때는 같은 지역 안에서만 할 수도 있고, 여러 지역에 걸쳐 할 수도 있다. 여러 지역에 걸쳐 다중화하면 데이터 손실을 막고 가용성을 최대한 보장할 수 있으므로 그렇게 하기로 한다.

S3 버킷은 마치 파일 시스템의 폴더와도 같은 것이다.



추가 개선점

- 로드밸런서: 네트워크 트래픽을 분산하기 위해 로드밸런서를 사용한다. 로드밸런서는 트래픽을 고르게 분산할 수 있을 뿐 아니라 특정 웹 서버에 장애가 발생하면 자동으로 해당 서버를 우회해준다.

- 웹 서버: 로드밸런서를 추가하고 나면 더 많은 웹 서버를 손쉽게 추가할 수 있다. 트래픽이 폭증해도 쉽게 대응이 가능하다.

- 메타 데이터 데이터베이스: 데이터베이스를 파일 저장 서버에서 분리하여 SPOF(Single Point of Failure)를 회피한다. 다중화 및 샤딩 정책을 적용하여 가용성과 규모 확장성 요구사항에 대응한다.

- 파일 저장소: S3를 파일 저장소로 사용하고 가용성과 데이터 무손실을 보장하기 위해 두 개 이상의 지역에 데이터를 다중화한다.



동기화 충돌

먼저 처리되는 변경은 성공한 것으로 보고, 나중에 처리되는 변경은 충돌이 발생한 것으로 표시한다.



개략적 설계안

- 블록 저장소 서버: 파일 블록을 클라우드 저장소에 업로드하는 서버다. 블록 저장소는 클라우드 환경에서 데이터 파일을 저장하는 기술이다. 이 저장소는 파일을 여러 개의 블록으로 나눠 저장하며 각 블록에는 고유한 해시값이 할당된다. (드롭박스의 사례 - 최대 4MB)

- 클라우드 저장소: 파일은 블록 단위로 나눠져 클라우드 저장소에 보관

- 아카이빙 저장소: 오랫동안 사용하지 않은 비활성 데이터를 저장하기 위한 시스템

- 로드밸런서: 요청을 모든 API 서버에 고르게 분산하는 구실을 함

- API 서버: 파일 업로드 외에 거의 모든 것을 담당하는 서버

- 메타데이터 데이터베이스: 사용자, 파일, 블록, 버전 등의 메타데이터 정보를 관리 (실제 파일은 클라우드에 보관, 여기에는 메타데이터만 둔다)

- 메타데이터 캐시: 성능을 높이기 위해 자주 쓰이는 메타데이터는 캐시

- 알림 서비스: 특정 이벤트가 발생했음을 클라이언트에게 알리는데 쓰이는 발생/구독 프로토콜 기반 시스템

- 오프라인 사용자 백업 큐: 클라이언트가 접속 중이 아니라서 파일의 최신 상태를 확인할 수 없을 때는 해당 정보를 이 큐에 두어 나중에 클라이언트가 접속했을 때 동기화될 수 있도록 한다.



3단계 - 상세 설계
블록 저장소 서버

- 델타 동기화: 파일이 수정되면 전체 파일 대신 수정이 일어난 블록만 동기화하는 것이다.

- 압축: 블록 단위로 압축해두면 데이터 크기를 많이 줄일 수 있다.



높은 일관성 요구사항

이 시스템은 강한 일관성 모델을 기본으로 지원해야 한다. (=같은 파일이 단말이나 사용자에 따라 다르게 보이는 것을 허용할 수 없다)

관계형 데이터베이스는 ACID(Atomicity, Consistency, Isolation, Durability)를 보장하므로 강한 일관성을 보장하기 쉽다. NoSQL 데이터베이스는 이를 기본으로 지원하지 않으므로 동기화 로직 안에 프로그램해 넣어야 한다.


업로드 절차

*파일 메타데이터 추가

1. 클라이언트 1이 새 파일의 메타데이터를 추가하기 위한 요청 전송

2. 새 파일의 메타데이터를 데이터베이스에 저장하고 업로드 상태를 대기 중으로 변경

3. 새 파일이 추가되었음을 알림 서비스에 통지

4. 알림 서비스는 관련된 클라이언트(클라이언트 2)에게 파일이 업로드되고 있음을 알림



*파일을 클라우드 저장소에 업로드

1. 클라이언트 1이 파일을 블록 저장소 서버에 업로드

2. 블록 저장소 서버는 파일을 블록 다누이로 쪼갠 다음 압축하고 암호화 한 다음에 클라우드 저장소에 전송

3. 업로드가 끝나면 클라우드 스토리지는 완료 콜백을 호출, 이 콜백 호출은 API 서버로 전송

4. 메타데이터 DB에 기록된 해당 파일의 상태를 완료로 변경

5. 알림 서비스에 파일 업로드가 끝났음을 통지

6. 알림 서비스는 관련된 클라이언트에게 파일 업로드가 끝났음을 알림



다운로드 절차

다른 클라이언트가 파일을 편집하거나 추가했다는 사실을 어떻게 감지하게 할지?

- 클라이언트 1이 접속중이고 다른 클라이언트가 파일을 변경하면 알림 서비스에서 클라이언트에게 변경이 발생했으니 새 버전을 끌어가야 한다고 알린다.

- 클라이언트 1이 네트워크에 연결된 상태가 아닐 경우에 데이터는 캐시에 보관될 것이다. 해당 클라이언트가 접속 중으로 바뀌었을 때 해당 클라이언트는 새 버전을 가져간다.



: 어떤 파일이 변경되었음을 감지한 클라이언트는 우선 API 서버를 통해 메타데이터를 새로 가져가야 하고, 그 다음에 블록들을 다운로드하여 파일을 재구성해야 한다.



1. 알림 서비스가 클라이언트에게 누군가 파일을 변경했음을 알림

2. 알림을 확인한 클라이언트는 새로운 메타데이터를 요청

3. API 서버는 메타데이터 데이터베이스에게 새 메타데이터 요청

4. API 서버에게 새 메타데이터가 반환

5. 클라이언트 2에게 새 메타데이터가 반환

6. 클라이언트는 새 메타데이터를 받는 즉시 블록 다운로드 요청 전송

7. 블록 저장소 서버는 클라우드 저장소에서 블록 다운로드

8. 클라우드 저장소는 블록 서버에 요청된 블록 반환

9. 블록 저장소 서버는 클라이언트에게 요청된 블록 반환, 클라이언트는 전송된 블록을 사용하여 파일 재구성



알림 서비스

파일의 일관성을 유지하기 위해 클라이언트는 로컬에서 파일이 수정되었음을 감지하는 순간 다른 클라이언트에게 그 사실을 알려서 충돌 가능성을 줄여야 한다.



- 롱 폴링(long polling): 드롭박스가 채택하고 있는 방식 - 선택!

- 웹 소캣(web socket): 클라이언트와 서버 사이에 지속적인 통신 채널을 제공, 양방향 통신 가능 - 채팅 서비스



롱 폴링 방안을 쓰게 되면 각 클라이언트는 알림 서버와 롱 폴링용 연결을 유지하다가 특정 파일에 대한 변경을 감지하면 해당 연결을 끊는다. 이때 클라이언트는 반드시 메타데이터 서버와 연결해 파일의 최신 내역을 다운로드해야 한다.



저장소 공간 절약

- 중복 제거

- 지능적 백업 전략 (한도 설정, 중요한 버전만 보관)

- 자주 쓰이지 않는 데이터는 아카이빙 저장소로 이동 (아마존 S3 글래시어와 같은 아카이빙 저장소 이용료는 S3보다 훨씬 저렴)



장애 처리

- 로드밸런서 장애: 일정 시간동안 박동 신호에 응답하지 않은 로드밸런서는 장애가 발생한 것으로 간주

- 블록 저장소 서버 장애: 다른 서버가 미완료 상태 또는 대기 상태인 작업을 이어받아야 한다.

- 클라우드 저장소 장애: 다중화 (다른 지역에서 파일을 가져오면 된다.)

- API 서버 장애: 트래픽을 해당 서버로 보내지 않음으로써 장애 서버를 격리

- 메타데이터 캐시 장애: 다중화

- 메타데이터 데이터베이스 장애: 주 데이터베이스가 장애일 경우 부 데이터베이스 서버 가운데 하나를 주로 바꾸고 부를 새로 하나 추가. 부 데이터베이스 서버가 장애일 경우, 다른 부 데이터베이스 서버가 읽기 연산을 처리하도록 하고 그동안 새것으로 교체한다.

- 알림 서비스 장애: 많은 사용자와 연결을 유지하고 관리해야 한다. (동시에 백만 개 접속을 유지는 가능하지만, 시작하는 것은 불가하다)

- 오프라인 사용자 백업 큐 장애: 다중화



4단계 - 마무리
1. 블록 저장소 서버를 거치지 않고 파일을 클라우드 저장소에 직접 업로드하면?

- 장점: 업로드 시간이 빨라진다.

- 단점: 분할, 압축, 암호화 로직을 클라이언트에 둬야 하므로 플랫폼별로 따로 구현해야 한다. 그리고 클라이언트가 해킹 당할 가능성이 있어 암호화 로직을 클라이언트에 두는 것은 적절치 않은 선택일 수 있다.



2. 접속 상태를 관리하는 로직을 별도 서비스로 옮긴다?

알림 서비스를 분리해내면 다른 서비스에서도 쉽게 활용할 수 있게 되므로 좋을 것.