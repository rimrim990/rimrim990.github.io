---
layout: post
title: 자바 NIO로 논블로킹 네트워크 I/O 구현하기
author: rimrim990
tags: [java]
toc: true
---
자바 NIO를 사용해 논블로킹 네트워크 I/O를 구현해보자!
<!--break-->

## NIO 패키지

## 자바 IO 패키지
`java.io` 패키지에서 제공하는 소켓 통신은 **블로킹 방식**으로 동작한다.
클라이언트 소켓을 생성해 서버 소켓에 요청을 전송하는 상황을 가정해보자.

```java
public void sendMessage(final Socket socket, final String message) throws IOException {
    // 서버에 요청을 전송한다
    OutputStream outputStream = socket.getOutputStream();
    PrintWriter writer = new PrintWriter(new OutputStreamWriter(outputStream));
    writer.print(message);
    writer.flush();
}
```
- 클라이언트는 `OutputStream`을 생성해 서버에 요청을 전송한다
- 소켓 전송 버퍼에 **데이터를 저장할 공간이 없으면** `print()` 메서드는 **블록**된다

클라이언트는 서버에 요청을 전송한 후 응답을 받아올 것이다.
```java
public String receiveMessage(final Socket socket) throws IOException {
    // 서버 응답을 받아온다
    InputStream serverInput = sock.getInputStream();
    BufferedReader reader = new BufferedReader(new InputStreamReader(serverInput));
    StringBuilder ServerResponse = new StringBuilder();

    // 데이터가 모두 도착할 때까지 받아온다
    String line;
    while (StringUtils.isNotBlank(line = reader.readLine())) {
        ServerResponse.append(line);
    }
}
```
- `InputStream`을 생성해 서버에서 보낸 응답을 가져온다
- 소켓에서 **읽어올 데이터가 없으면** `readLine()` 메서드는 **블록**된다

정리하면 `java.io`의 특징은 다음과 같다
- 데이터를 전송하거나 요청할 준비가 되지 않았다면 준비가 완료될 때까지 메서드는 블록된다
- 데이터를 전송할 때와 읽어올 때 각각 단방향 스트림 객체를 생성해야 한다

`java.io`와 다르게 `java.nio`는 논블로킹 방식의 네트워크 I/O를 지원한다.

### 자바 NIO 패키지
NIO는 채널과 버퍼, 셀렉터 (`Selector`)로 구성된다.

**SocketChannel**
```java
public void connectToServer() throws IOException {
    // 클라이언트 소켓 채널
    SocketChannel socketChannel = SocketChannel.open();
    sockChannel.configureBlocking(true);
    sockChannel.connect(address);
}
```
- 스트림 기반의 IO와 다르게 NIO는 채널 기반으로 동작한다
- IO는 데이터 입출력을 위해 입력 스트림과 출력 스트림을 생성했지만, 채널은 입출력이 모두 가능하다
- `open()` 메서드를 호출해 소켓을 위한 소켓 채널을 생성한다
- `configureBlocking()` 메서드를 호출해 **논블로킹 설정이 가능하다**

**Buffer**
```java
public void read(final SelectionKey key) throws IOException {
    // 소켓 데이터를 저장하기 위한 버퍼 메모리 할당
    ByteBuffer buffer = ByteBuffer.allocate(256);
    SocketChannel socketChannel = (SocketChannel)selectionKey.channel();
    // 소켓 데이터를 읽어 버퍼에 저장
    socketChannel.read(buffer);
    buffer.flip();
    return CHARSET.decode(buffer).toString();
}
```
- 채널은 소켓에 입출력된 데이터를 버퍼 공간에 저장한다

**Selector**
```java
public void createServerSocketChannel() throws IOException {
    // 논블로킹 서버 소켓 채널을 생성한다
    final serverSocketChannel = ServerSocketChannel.open();
    final serverSocketChannel.bind(new InetSocketAddress(PORT));
    final serverSocketChannel.configureBlocking(false);

    // 셀렉터에 작업을 등록한다
    final selector = Selector.open();
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

    // 작업이 완료되길 기다린다
    final Set<SelectionKey> selectedKeys = selector.select();
}
```
- 셀렉터는 등록된 작업 중에서 입출력 준비가 완료된 작업들을 알려준다
- 셀렉터에 여러 개의 채널과 작업을 등록할 수 있다
- `select()` 메서드는 등록된 작업 중에서 준비된 작업이 하나 이상 생성될 때까지 블로킹한다

### Buffer
소켓 채널은 입출력 데이터를 버퍼 메모리 공간에 저장한다.
- `Buffer`는 데이터를 읽고 쓸 수 있는 메모리 공간이다
- 추상 클래스인 `Buffer`에는 `ByteBuffer`, `CharBuffer`, `IntBuffer` 등 타입별 구현체가 존재한다

**버퍼 위치 정보**

버퍼는 데이터를 추적하기 위해 내부적으로 위치 정보를 기록한다.
```
position
limit
capacity
```
- position 은 현재 읽거나 쓰고 있는 인덱스 값으로, 0부터 시작한다
- limit 은 position 이 도달할 수 있는 최대 값이다. position과 limit 이 같아지면 더이상 버퍼에 데이터를 저장할 수 없다
- capacity 는 버퍼에 저장할 수 있는 데이터의 최대 개수이다

```
0 <= position <= limit <= capacity
```
- 위치 정보 간에는 위와 같은 대소 관계가 성립한다

버퍼에 데이터를 저장하고, 이를 읽는 상황을 가정해보자.
```java
// 소켓 데이터를 저장하기 위한 버퍼 메모리 할당
ByteBuffer buffer = ByteBuffer.allocate(256);
SocketChannel socketChannel = (SocketChannel)selectionKey.channel();
```
- 소켓에서 읽어온 데이터를 저장하기 위해 버퍼를 할당했다

```java
// 소켓 데이터를 읽어 버퍼에 저장
socketChannel.read(buffer);
```
- 소켓에서 읽은 데이터를 버퍼에 저장했다

버퍼에 데이터를 저장했기 때문에 `position` 값이 이동했다.
만약 저장된 데이터를 읽고 싶다면 어떻게 해야할까?
저장된 데이터는 0번부터 `position-1`의 위치까지 저장되어 있을 것이다.

```java
// 버퍼 데이터를 읽기 위해 포지션을 이동한다
buffer.flip();
return CHARSET.decode(buffer).toString();
```
- `flip()` 메서드를 호출해 위치 정보를 조정해줘야 한다
- `flip()` 호출로 `position`은 0의 위치로, `limit`은 기존 `position`의 위치로 이동된다
- 따라서 0번부터 `position-1` 까지 저장된 데이터를 읽어올 수 있다

버퍼에 저장된 데이터를 전부 읽어왔다.
이번에는 버퍼에 새로운 데이터를 작성하려고 할 때, 어떻게 해야할까?

```java
// 새로운 데이터 저장을 위해 포지션을 이동한다
buffer.compact();

// 소켓으로 전송할 데이터를 버퍼에 저장한다
socketChannel.write(buffer);
```
- `compact()`를 호출해 이미 읽어온 데이터는 버퍼에서 제거한다
- 아직 읽지 않은 데이터들은 0번 인덱스를 시작점으로 차례대로 복사된다
- `limit` 값은 `capacity`로 이동한다

### Selector
셀렉터는 소켓 채널에 발생한 이벤트를 감지해주는 일종의 이벤트 리스너라고 볼 수 있다.
셀렉터는 **등록된 작업 유형**에 대해, 해당 작업을 실행할 수 있는 **준비가 완료되면** 이를 알려주는 역할을 한다.

```java
// 셀렉터에 작업을 등록한다
final selector = Selector.open();
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
```
- 예를 들어, 셀렉터에 서버 소켓 채널과 `ACCEPT` 작업을 등록한다
- 서버 소켓이 `ACCEPT` 작업을 실행할 준비가 완료되면 셀렉터를 통해 이를 전달받을 수 있다

**셀렉터 작업 유형**

셀렉터에 등록 가능한 작업 유형은 다음과 같다.
- OP_ACCEPT - 서버 소켓 채널의 연결 수락 작업
- OP_CONNECT - 소켓 채널의 연결 작업
- OP_READ - 소켓 채널의 데이터 읽기 작업
- OP_WRITE - 소켓 채널의 데이터 쓰기 작업

앞서 IO 패키지를 사용해 소켓에 읽기 요청을 전송했을 때, 데이터가 존재하지 않으면 메서드가 블록됐었다.
그러나 논블로킹 메서드와 셀렉터를 사용하면 이를 다음과 같이 변경할 수 있다.
- 읽기 메서드는 데이터가 존재하는지 여부와 관계없이 **일단 반환된다**
- 셀렉터는 소켓이 읽을 수 있는 데이터가 존재하면 이를 전달해준다

**셀렉터 키셋**

셀렉터는 준비가 완료된 작업들을 `SelectionKey`로 반환해준다.

```java
selector.select();
Set<SelectionKey> selected = selector.selectedKeys();
```
- `select()` 메서드는 반환 가능한 키셋이 생성될 때까지 블록한다
- 준비 완료된 작업이 있으면 키셋을 반환한다

```java
for (SelectionKey key : selectedKeys) {

    // 서버 소켓 - 클라이언트 연결 수락 작업
  if (key.isAcceptable()) {
      register(selector, serverSocket);
  }

  // 서버 소켓 - 읽기 작업
  if (key.isReadable()) {
      response(buffer, key);
  }
}
```
- 키셋을 순회하며 `SelectionKey`가 가리키는 작업 유형에 따라 적절히 처리해준다

**셀렉터로 동시성 향상하기**

셀렉터를 사용해 싱글 스레드만으로도 여러 소켓 채널을 처리할 수 있다.
- 작업 처리가 가능해질 때까지 하나의 메서드에서 블로킹되지 않고 다른 작업을 처리할 수 있다

만약 오래 걸리는 작업이 존재할 경우, 이를 다른 스레드에 위임해 처리 성능을 향상시킬 수 있다.
