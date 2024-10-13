---
layout: post
title:  "Spring Application에서 파일 처리하기"
author: 김연우
date: '2024-10-13'
thumbnail: /assets/img/posts/javaio.jpeg
keywords: Spring, File, WebFlux
---

이전 회사에 다닐때는 어플리케이션에서 파일 데이터를 다뤄야 하는 일이 많았다. 헬스허브가 의료영상을 주로 다루는 회사이고, 또 당시 우리 팀이 개발을 맡았던 서비스가 의료영상을 병원에서 가져오거나 병원으로 전송해주는 서비스였기 때문이다. 이직하고 나서는 파일을 다룰 일이 많이 없어졌는데, 최근 어드민 페이지 작업을 하면서 파일을 여러개 가져와서 압축하여 내보내는 API 작업을 할 일이 있어서 이전에 겪었던 파일 처리 관련 난관이 다시금 많이 떠올랐다. 이전의 미흡했던 접근들까지 포함해서 스프링 어플리케이션에서 파일을 처리하는 여러가지 방법들을 정리해보고자 한다.

### 접근 1: byte[] 사용하기
스프링 서버에서 가장 간단하게 파일을 응답하는 방법은 `ResponseEntity<byte[]>` 로 응답하는 것이다. 파일을 바이트 배열로 읽어들이고 응답 헤더에 `MediaType`을 적절히 설정하는 것으로 파일을 응답하는 엔드포인트를 작성할 수 있다. Spring이 제공하는 `Resource` 인터페이스를 활용한다면 `ResponseEntity<ByteArrayResource>` 로 바이트배열을 한겹 랩핑해서 응답할 수도 있다. 그렇다면 이 방식의 문제점을 무엇일까?

이 방식의 문제점은 바로 서버가 파일을 응답하기 위해 **파일 전체를 메모리에 올린다**는 것이다. 응답하는 파일의 크기가 2GB라면 자바 힙에 2GB가 실제로 할당되는 것이라고 이해하면 된다. 거기에 더해 스프링 어플리케이션은 동시에 여러 요청을 받을 수 있으므로 응답하고자 하는 파일의 크기가 엄청 크지 않더라도 요청이 몰리면 메모리에 심각한 부하를 줄 수 있고, 쉽사리 OOM으로 이어질 것이다. 따라서 이 방법은 파일의 크기가 매우 작고 고정적일때만 사용할 수 있는 방법이며, 운용 서비스에서는 권장하지 않는 구현법이다.

실제로 아무것도 모르던 때에 이렇게 구현했다가 OOM을 만난적도 있었다. 그 다음에 시도한 방법이 바로 뒤에 소개할 디스크에 쓰고 InputStream으로 읽어오는 방법이다.

### 접근 2: 파일로 저장하고 FileInputStream으로 읽어오기
메모리 부하를 줄이면서 파일을 서빙할 수 있는 또다른 방법으로는 파일을 디스크에 쓰는 방법이 있다. 예를 들어, 여러개의 파일을 다른 서버에서 받아와서 ZIP으로 압축해서 응답하는 요구사항이 있다고 가정해보자. 이럴 때 쓸 수 있는 것은 `java.util.zip.ZipOutputStream` 이다.  `ZipOutputStream`을 사용하면 아래와 같이 여러개의 파일들을 하나의 ZIP파일로 묶을 수 있다.
```
    ZipOutputStream(FileOutputStream("test.zip")).use { out ->
        for (file in files) {
            FileInputStream(file).use { fi ->
                fi.use { origin ->
                    val entry = ZipEntry(file.substring(file.lastIndexOf("/")))
                    out.putNextEntry(entry)
                    origin.copyTo(out)
                }
            }
        }
    }
```

여기서 주목해야할 부분은 `ZipOutputStream` 은 압축한 결과를 낼 OutputStream을 인자로 받고, 위 예제에서는 이를 위해 “test.zip”이라는 새로운 파일을 생성한다는 것이다. 이렇게 파일을 생성하면 파일 전체를 메모리에 올리지 않아서 안정적이고, 영구 저장소에 저장되어있기 때문에 반복해서 읽을 수 있다는 확실한 장점이 있다. 만약 만들고자 하는 서비스가 본격적인 파일 관리 서버라면, 이렇게 파일을 쓰고 해당 파일을 스트리밍하여 응답하는 방식을 많이 택할 것이다. 하지만 파일을 쓴다는것은 disk IO를 수반하기 때문에 순수하게 메모리 위에서 처리하는 것에 비해 시간이 오래 걸린다는 단점이 있다. 또한, 파일을 저장하게 되는 만큼 서버를 올리는 하드웨어의 디스크 용량이나, 파일의 삭제 시점 등 관리해야하는 포인트가 늘어나는 단점도 있을 것이다.

파일을 저장하는 방식으로 구현하는 경우에는 주로 `try finally` 나 reactor 스택의 경우에는 `doAfterTerminate` 절을 사용하여 서버 응답을 끝마친 이후에 파일을 삭제하는 식으로 구현했었다.

### 접근 3: Buffering과 Streaming 이용하기
접근 2에서의 예제를 파일에 저장하지 않고, 또 메모리에 전체 파일을 올리지 않은채로 구현하기 위해서는 어떻게 해야할까? 전통적인 blocking IO 모델에서는 `PipedInputStream` `PipedOutputStream`을 사용하여 구현할 수 있다. `PipedOutputStream`은 선언시에 버퍼의 크기를 지정하고 기본적으로 버퍼의 양만큼만 쓸 수 있다. 따라서 ZIP 파일을 `PipedOutputStream`에 써내려간다고 가정했을때, ZIP 파일의 총 용량과는 상관없이 사용자가 지정한 버퍼의 크기만큼만 쓰고 쓰레드는 기다리게 된다.

이 기다림은 다른 쓰레드가 연결된 `PipedInputStream`에서 데이터를 읽어서 파이프를 비워줘야지만 해소된다. 따라서 아래 그림과 같이 PipedOutputStream, InputStream을 활용하기 위해서는 서로 다른 두개의 쓰레드가 필요하며, 하나는 쓰고 하나는 읽는 작업을 같이 진행한다.

따라서 `Piped IO`를 사용해 ZIP파일을 응답해주기 위해서는 요청 쓰레드와는 별도의 쓰레드가 ZIP 파일을 쓰는 작업을 맡아야하며, 요청 쓰레드는 쓰여지고 있는 스트림을 읽어들여서 다시 Servlet의 응답 스트림에 써내려가면 된다. 이렇게 하면 파일을 전부 메모리에 올리지 않고, 또 디스크에 저장하는 오버헤드 없이도 파일을 처리하고 응답할 수 있다.

Piped IO는 다른 Stream 구현체들과 마찬가지로 blocking한 것이 큰 특징이다. 아래 그림의 Thread 1은 있지만 2는 없다면 Thread 1은 버퍼 크기만큼 써내려간 이후에 파이프가 빌때까지 계속 기다릴 것이고, 마찬가지로 Thread 2도 1이 없다면 InputStream에 데이터가 들어오기를 계속 기다릴 것이다.

![](/assets/img/posts/piped.png)

그렇다면 비슷하게 이런 버퍼링과 스트리밍을 구현할 수 있는 non-blocking한 방법은 뭐가 있을까? WebFlux 스택에서는 비슷한 처리를 위해 `DataBuffer`를 사용한다. 스프링의 non-blocking HTTP client인 `WebClient`는 데이터를 `Flux<DataBuffer>` 형태로 받아오는 것을 지원하며, 컨트롤러 또한 `ResponseEntity<Flux<DataBuffer>>` 형태로 데이터를 응답하는 것을 지원한다. 이렇게 응답할 시에 `Flux<DataBuffer>` 는 요청 시에 subscribe 되어 버퍼 하나하나 HTTP 응답에 차례로 써내려가질 것이다. 이러한 메커니즘은 버퍼링과 스트리밍의 측면에서 전통적인 `Piped IO`와 꽤나 비슷하다고 할 수 있다.

하지만 두가지 방식엔 큰 차이가 있는데, 가장 큰 차이는 `Flux<DataBuffer>`로 응답하는 방식으로 reactive, non-blocking하다는 것이다. 먼저 **reactive** 하다는 것의 뜻은 간단하게 얘기하면 이렇게 말할 수 있을것 같다. “Piped IO에서 InputStream은 OutputStream이 무언가를 쓰는 것을 기다리는 반면, WebFlux에서 `Flux<DataBuffer>`는 다음 데이터를 줄 것을 요청한다”. 또한 **non-blocking** 하다는 것은 “PipedInputStream은 그 자리에서 데이터가 오기를 기다리고 있지만 `Flux<DataBuffer>`는 데이터가 왔다는 것을 알려준다”. 따라서 Piped IO와 달리 DataBuffer 방식은 dedicated thread가 필요없고, 비동기로 진행되어 대용량 처리에 훨씬 효율적이라고 이해할 수 있다. 

또한 `Piped IO`의 유즈케이스는 쓰레드간 데이터 스트리밍에 많이 한정되는 반면에, DataBuffer의 경우 이 파이프라이닝을 서버간 Network IO에까지 쭉 이을 수 있다. WebClient에서 데이터를 받아와서 해당 데이터를 적절히 프로세싱한 이후에 응답하는 전체 과정을 하나의 스트림으로 이을 수 있는 것이다. 아래는 Piped IO와 WebFlux의 데이터 버퍼간 비교를 해본 표이다.


### Piped IO와 DataBuffer 비교표

| **Feature** | **Piped I/O** | DataBuffer **(WebFlux)** |
|:-:|:-:|:-:|
| **Main Use Case** | Thread-to-thread communication | Reactive I/O in web applications (WebFlux) |
| **I/O Type** | Blocking I/O | Non-blocking, reactive I/O |
| **Data Handling** | Direct stream communication between threads | Handles HTTP request/response bodies, file I/O |
| **Data Representation** | byte or char stream | DataBuffer chunks, typically byte arrays |
| **Concurrency** | Synchronous between two threads | Asynchronous, handles multiple clients/reactive streams |
| **Internal Buffering** | Small fixed-size buffer | Memory-efficient, potentially pooled buffers |
