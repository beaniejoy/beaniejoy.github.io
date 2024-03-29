---
title: "[Kotlin] JAVA 코드를 가져올 때 nullable 주의점"
author:
  name: beaniejoy
  link: https://github.com/beaniejoy
date: 2022-02-18 03:03:00 +0900
categories: [Kotlin]
tags: [kotlin]     # TAG names should always be lowercase
img_path: /assets/img/posts/
image:
  src: kotlin.jpeg
  width: 800
  height: 500
  alt: not found
---

Java 코드로 작성된 클래스를 Kotlin으로 Converting하는 작업에서 중요한 부분이 전환하려는 클래스와 여러 형태로 연결된 기존 Java 클래스들을 고려해야 한다는 점입니다. 특히 Kotlin에서는 Java와 다르게 컴파일 단계에서 null 처리를 할 수 있다는 점입니다.

그래서 Kotlin 코드로 작성된 클래스에서 Java로 된 클래스를 사용할 때 **Java 클래스를 신뢰해서는 안된다는 점**이 중요합니다.

<br>

## 📌 Java 클래스를 인자로 받을 때

```kotlin
// Module: Java class
fun processExample(module: Module) {
    // ...
    module.sendAnything(anything)
    // ...
}
```

예를 들어 코틀린으로 이루어진 클래스에서 Java 코드로 된 클래스(예시에선 `Module`)를 인자로 받아 sending 처리를 하는 메소드가 있다고 해봅시다.

Java 클래스를 가져와 `sendAnything` 작업을 수행하게 됩니다. 그런데 `Module`이라는 Java 클래스가 null이 아니라는 것을 확정할 수 있을까요? Java 컴파일러는 null에 대해서 기본적으로 용인하기 때문에 **null처리를 따로 해야한다는 번거로움**이 있습니다.

```kotlin
// Module: Java class
fun processExample(module: Module) {
    if (module == null) throw RuntimeException("module을 가져올 수 없습니다.")
    // ...
    module.sendAnything(anything)
    // ...
}
```

이렇게 명시적으로 module에 대한 null 처리를 해줘야 합니다. 안그러면 module 인자가 null인 상태로 들어왔을 때 `sendAnything`에서 `NullPointerException`이 발생할 것입니다. 

send 처리하는데 Null 에러가 발생하면 비즈니스 로직 상 상당한 문제가 발생하게 되는데요.  
예를 들어 send 처리하기 전에 사용자가 요청한 작업을 처리하고 완료된 내용에 대해서 사용자에게 완료처리에 대한 내용을 전송하는 로직이 있다고 해봅시다. 요청사항에 대해서 성공적으로 처리가 이루졌음에도 sending 과정에서 위의 에러가 발생하면 전체 요청에 대해서 작업실패처리가 될 수 있습니다.

혹여나 해당 메소드가 하나의 트랜잭션으로(`@Transactional` 등으로) 묶여 있고 요청작업에서 테이블에 insert하는 내용이 있는 경우엔 더 큰 문제가 발생할 수 있습니다.  
module `NullPointerException`으로 인해 성공적으로 insert 했던 모든 데이터가 전부 롤백될 수 있습니다.  
(그래서 이러한 로직에서는 MessageQueue를 사용하는 것 같네요,,,)

예시를 sending 처리하는 비즈니스 로직으로 들었는데 모든 부분에 있어서 `NullPointerException`은 코틀린으로 마이그레이션하는 과정에서 Java와 혼용해서 사용할 때 가장 주의해야합니다.

<br>

## 📌 Java 클래스의 메소드와 필드

```kotlin
override fun afterJob(jobExecution: JobExecution) {
    val exitCode = jobExecution.exitStatus.exitCode
}
```

배치 관련해서 기능 추가하는 작업을 진행한 적이 있는데요. `jobExecution`을 받아 후처리해야 하는 부분이 있었습니다. 만약 위처럼 처리한다면 무슨 문제가 발생할 수 있을까요.

인자로 받은 `JobExecution`은 Java 코드로 이루어진 클래스입니다. `JobExecution`은 `JobExecutionListener`에서 제공하는 `afterJob` 메소드의 파라미터인데 Java 코드로 이루어졌다고 listener가 `JobExecution`을 null로 보내줄리는 없을 것입니다.

그럼에도 `JobExecution`에서 `exitStatus`를 받아오는 지점과 `exitCode`를 받아오는 지점에서 충분히 null이 될 수 있음을 인지해야 합니다. 

그래서 해당 코드는 다음과 같이 코틀린 null 처리 문법과 함께 사용하면 좋을 것 같습니다.

```kotlin
override fun afterJob(jobExecution: JobExecution) {
    val exitCode: String? = jobExecution.exitStatus?.exitCode
}
```

`exitStatus`에서도 nullable이 될 수 있다는 것과 `exitCode`에서 nullable이 될 수 있다는 것을 고려해서 각각 null 처리를 하면 됩니다. 변수는 type을 명시적으로(`String?`) 지정해줌으로써 nullable하다는 것을 지정해야 합니다.  
(nullable 처리하지 않으면 String! 로 받아드려서 후에 `NullPointerException` 발생 여지를 남길 수 있습니다.)

<br>

## 📌 정리

코틀린 초보자인 저에게 있어서 이런 사소한 요소 하나하나가 생소하고 중요하게 받아들이게 되는 것 같네요.   
이번 글의 핵심은 Java 클래스를 가져오는 지점에서 Java 클래스를 절대로 신뢰하지 말고 항상 null이 될 수 있음을 염두해서 코드를 작성해야 한다는 것입니다.

핵심 로직의 성공유무와 무관하게 이러한 지점에서 `NullPointerException`이 발생한다면 생각지못한 결과를 초래할 수 있기 때문에 제 마음 속에 새겨놓아야 될 것 같네요!