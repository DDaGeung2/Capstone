---
title: 스터디 3주차 / 4.5 ~ 4.6
date: '2019-03-28 17:00:00 -0400'
categories: study capstonedesign
published: true
---

---

4.5 JNI 네이티브 함수 직접 등록하기
-----------------------------------

---

자바 가상 머신은 네이티브 메서드를 포함하는 자바 애플리케이션을 실행할 떄 아래의 두 단계를 거친다.

1.	System.loadLibrary() 메서드를 이용해서 네이티브 메서드의 실제 구현이 포함된 C/C++ 라이브러리를 메모리 상에 로드한다.
2.	자바 가상 머신은 위에서 로드된 라이브러리의 함수 심볼을 검색해서 자바에서 선언된 네이티브 메서드의 시그너처와 일치하는 JNI 네이티브 함수 심볼을 찾은 다음 네이티브 메서드와 실제 구현인 JNI 네이티브 함수를 매핑한다.

안드로이드 프레임워크처럼 각종 네이티브 메서드가 포함된 자바 클래스가 많은 경우에는 자바 가상 머신이 라이브러리를 로딩하고 이들의 심볼을 일일이 검색해서 각 네이티브 메서드를 적당한 함수와 매핑하는 작업은 성능 저하의 원인이 된다.

JNI는 이러한 문제점을 해결하기 위해 C/C++ 개발자가 JNI 네이티브 함수를 직접 자바 클래스의 네이티브 메서드에 매핑할 수 있게 해주는 `RegisterNatives()` 라는 JNI 함수를 제공한다.

자바 가상 머신이 자동으로 심볼을 검색해서 적절한 JNI 네이티브 함수를 연결하는 작업을 `RegisterNatives()` 함수가 대신해준다. 프로그래머가 이를 이용해 직접 작업을 처리하기 때문에 별도의 매핑 과정을 생략할 수 있어 로딩 속도를 향상시킬 수 있다.

프로그래머가 JNI 네이티브 함수와 네이티브 메서드를 직접 연결하므로 JNI 네이티브 함수의 이름을 JNI 지원 가능한 네이밍 룰에 맞출 필요 없이 아무 함수 이름으로 바로 네이티브 메서드로 연결할 수 있게 된다.

### 네이티브 라이브러리 로드 시에 JNI 네이티브 함수 등록하기

자바 가상 머신 대신 프로그래머가 네이티브 메서드와 JNI 네이티브 함수를 매핑하려면, 라이브러리를 로딩할 때 *자동으로* 호출되는 JNI_OnLoad() 함수 안에서 RegisterNatives() 함수를 이용해 매핑을 처리해야 한다.

`JNI_OnLoad()` 함수의 기본 역할은 로드한 라이브러리를 이용하는 데 필요한 JNI 버전을 자바 가상 머신으로부터 확인하는 것이다. 때문에 이 함수는 JNI 버전 정보를 반드시 반환해야 한다. 자바 가상 머신이 네이티브 라이브러리를 로드할 때 JNI에 대한 초기화 작업도 할 수 있게 해주는 함수이다.

JNI 네이티브 함수가 포함되어 있는 라이브러리에 JNI_OnLoad() 함수를 추가하고 이 함수에서 RegisterNatives() 함수를 호출해서 매핑을 진행하는 것을 확인해본다.

#### RegisterNatives() JNI 함수

RegisterNatives() JNI 함수의 원형은 아래와 같다.

`jarray RegisterNatives(JNIEnv *env, jclass clazz, const JNINativeMethod *methods, jint nMethods)`

clazz 인자에 포함된 네이티브 메서드를 JNI 네이티브 함수와 연결한다. 인자 `env`는 JNI 인터페이스 포인터, `clazz`는 자바 클래스, `methods`는 네이티브 메서드를 실제 JNI 네이티브 함수와 연결시키기 위한 정보, `nMethods`는 methods의 배열의 원소의 개수를 나타낸다. 성공 시 생성한 배열의 레퍼런스, 실패 시 NULL을 리턴한다.

#### JNINativeMethod 구조체

```
typedef struct {
  char *name; // 네이티브 메서드 이름
  char *signature; // 네이티브 메서드 시그너처
  void *fnPtr; // 네이티브 메서드와 매핑할 JNI 네이티브 함수 포인터
} JNINativeMethod
```

본 코드에서 사용될 JNINativeMethod 구조체 배열은 위와 같이 정의되어 있다. 네이티브 메서드와 JNI 네이티브 함수를 매핑하기 위해 이 구조체 배열을 사용하여 정의한 다음 RegisterNatives() 함수를 호출하게 된다.

#### hellojnimap.cpp 살펴보기

```c++
// JNI 네이티브 함수를 구현하려면 반드시 선언해야 하는 헤더파일로,
// 각종 JNI 자료구조 및 JNI 함수, 매크로 상수등이 정의되어 있음.
#include "jni.h"
#include <stdio.h>

// 1) 네이티브 메서드에 매핑할 JNI 네이티브 함수의 원형 선언.
//    네이밍 룰에 맞출 필요가 없으므로 간단한 함수명 이용.
//    단, JNI 네이티브 함수에서 사용하는 공통 매개변수는 기존 방식과 동일하게 지정해주어야 함.
void printHelloNative(JNIEnv *env, jobject obj);
void printStringNative(JNIEnv *env, jobject obj, jstring string);

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved)
{
  JNIEnv* env = NULL;
  JNINativeMethod nm[2];
  jclass cls;
  jint result = -1;

  // 2) JNI_OnLoad() 함수의 기본 기능인 JNI 버전을 확인하는 부분.
  if(vm->GetEnv((void **) &env, JNI_VERSION_1_4) != JNI_OK) {
    printf("Error");
    return JNI_ERR;
  }
  // GetEnv() 호출 API를 통해 버전 지원 여부 확인 후 env에 JNI 인터페이스 포인터 설정.
  // 지원하면 매크로 상수(JNI_VERSION_1_4) 반환.
  // 지원하지 않으면 ERR 반환 및 라이브러리 로딩 종료.
  // JNI 인터페이스 포인터가 설정되었으므로 이를 이용해 JNI 함수 사용 가능.

  // 3) 1)에서 선언한 JNI 네이티브 함수 연결 위해 네이티브 메서드를 포함하고 있는 클래스를 로딩하고 레퍼런스 값 저장.
  cls = env->FindClass("HelloJNI");

  // 4-1) 네이티브 메서드와 JNI 네이티브 함수 매핑.
  //      HelloJNI 클래스의 printHello()와 printHelloNative() 매핑.
  nm[0].name = "printHello";
  nm[0].signature = "()V";
  nm[0].fnPtr = (void *)printStringNative;

  // 4-2) 네이티브 메서드와 JNI 네이티브 함수 매핑.
  //      HelloJNI 클래스의 printString()과 printStringNative() 매핑.
  nm[1].name = (char*)"printString";
  nm[1].signature = (char*)"(Ljava/lang/String;)V";
  nm[1].fnPtr = (void *)printStringNative;

  // 5) 매핑 정보 RegisterNatives() 함수의 인자로 넘김.
  env->RegisterNatives(cls, nm, 2);

  return JNI_VERSION_1_4;
}

// JNI 네이티브 함수 구현
void printHelloNative(JNIEnv *env, jobject obj)
{
  printf("Hello World\n");
  return;
}

void printStringNative(JNIEnv *env, jobject obj, jstring string)
{
  const char *str = env->GetStringUTFChars(string, 0);
  printf("%s\n", str);

  return;
}
```

(그림 4-28 첨부)

### 안드로이드에서의 활용 예

#### System Server: JNI_OnLoad에서 JNI 네이티브 함수를 등록하기

안드로이드에서도 JNI_OnLoad()를 이용해서 네이티브 메서드를 직접 매핑하는 방법을 활용한다. `시스템 서버(System Server)`는 안드로이드가 로딩될 때 각종 서비스를 실행하는 역할을 하며, 안드로이드 부팅 시에 JNI로 매핑될 다양한 네이티브 라이브러리를 로딩한다.

```j
native public static void init1(String args[]); // 네이티브 메서드

public static void main(String args[]) {
  ...
  // 1) 라이브러리를 로드하면서 JNI_OnLoad() 함수 실행.
  System.loadLibrary("android_servers");
  init1(args);
}
```

위의 코드는 안드로이드 프레임워크의 System Server 프로그램에서 android_servers 라이브러리를 로딩하는 예제이다. 안드로이드는 리눅스 기반이므로 로드하는 라이브러리 이름은 `libanroid_server.so` 가 된다.

libanroid_server.so 라이브러리의 구성 파일 중 `onload.cpp` 소스 코드에 JNI_OnLoad() 함수가 구현되어 있다. 즉 이 라이브러리에 `JNI_OnLoad()` 함수가 포함되어 있다.

##### onload.cpp 코드 안의 `JNI_OnLoad()` 함수

```c++
extern "C" jint JNI_OnLoad(JavaVM *vm, void* reserved)
{
  JNIEnv* env = NULL;
  jint result = -1;

  // 2) JNI 버전 확인 후 JNI 인터페이스 포인터 얻어옴.
  if(vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
    LOGE("GetEnv failed!");
    return result;
  }

  // 3) libanroid_server.so 안드로이드에 포함된 각 JNI 네이티브 함수와
  //    안드로이드 프레임워크를 구성하는 클래스의 네이티브 메서드 매핑.
  register_android_server_KeyInputQueue(env);
  register_android_server_HardwareService(env);
  register_android_server_AlarmManagerService(env);
  register_android_server_BatteryService(env);
  register_android_server_SensorService(env);
  register_android_server_SystemServer(env);

  ...

  return JNI_VERSION_1_4;
}
```

```
static JNINativeMethod gMethods[] = {
  { "init1",
  "([LJava/lang/string];)V",
  (void*) android_server_SystemServer_init1 },
};

int register_android_server_SystemServer(JNIEnv* env)
{
  return jniRegisterNativeMethods(env, "com/android/server/SystemServer",
         gMethods, NELEM(gMethods));
}

static void android_server_SystemServer_init1(JNIEnv* env, jobject clazz)
{
  system_init();
}
```

추후내용추가예정

---

4.6 안드로이드 NDK로 개발하기
-----------------------------

---

`안드로이드 NDK(Native Development Kit)`는 애플리케이션 개발자가 JNI를 활용한 작업을 쉽게 할 수 있도록 구글에서 제공하는 개발 도구로, 안드로이드에서 C/C++같은 네이티브 코드를 빌드해서 라이브러리를 만든 다음 이를 안드로이드 애플리케이션 패키지에 삽입해주는 도구이다.

안드로이드 NDK는 안정적인 동작이 보장된 몇 개의 시스템 헤더 파일 및 라이브러리를 제공한다. 만약 NDK에서 지원하지 않는 시스템 라이브러리를 사용한다면 추후 안드로이드 플랫폼 업데이트로 인한 라이브러리 삭제나 업데이트 등이 있을 경우 정상 작동을 보장하지 못 하게 된다.

***TIP)* 안드로이드 NDK 설치 및 환경설정**

-	NDK가 설치된 루트 디렉터리의 경로를 `<NDK_HOME>`으로 가정하고, 이후로 이를 자신이 설치한 NDK 디렉터리로 생각한다. - 리눅스 기반의 프레임워크인 안드로이드에서 동작하는 네이티브 라이브러리는 리눅스용 공유 라이브러리(*.so)로 만들어야 한다. 윈도우 환경에서 리눅스 기반의 라이브버리를 빌드하기 위해 `CygWin`을 설치한다. - NDK를 위한 환경설정은 단순히 <NDK_HOME>을 윈도 PATH 환경변수에 지정하기만 하면 된다.

### 안드로이드 NDK 개발 따라하기

직접 만들어 볼 예제 프로그램은 크게 `자바로 구현된 안드로이드 애플리케이션`과 두 수의 합을 구하는 `C코드로 구성된 네이티브 라이브러리`로 나뉜다. (그림 4-36 링크)

SDK를 통해 구현되는 자바 측 코드에서는 `add() 네이티브 메서드`를 호출해서 두 수의 합을 구한 다음 이를 TextView를 통해 출력한다.

add() 네이티브 메서드의 실제 구현이 들어있는 `libndk-exam.so` 공유 라이브러리는 first.c 와 second.c 라는 두 개의 소스파일로부터 만들어진다. add() 메서드는 second.c 파일의 `Java_org_example_ndk_NDKExam_add() 함수`와 JNI를 통해 매핑된다.
