---
title: 스터디 3주차 / 4.3 ~ 4.4
date: '2019-03-28 17:00:00 -0400'
categories: study capstonedesign
published: true
---

4.3 JNI 함수 이용하기
---------------------

---

C코드인 JNI 네이티브 함수에서 자바 측 코드를 제어하는 방법을 살펴본다.

-	자바 객체 생성
-	클래스의 정적 멤버 필드에 접근하기
-	클래스의 정적 메서드 호출하기
-	자바 객체의 멤버 필드에 접근하기
-	자바 객체의 메서드 접근하기

### JNI 함수를 활용하는 예제 프로그램의 구조

크게 네이티브 메서드가 선언된 JniFuncMain 클래스, JniTest 객체, 네이티브 메서드의 실제 구현이 포함된 jnifunc.dll로 구성.

JNI 함수를 활용한 예제 프로그램의 전체 동작 순서 (그림 4-16 첨부)

자바측의 JniFuncMain 클래스에서 네이티브 메서드 `createJniObject()` 를 호출하는 것부터 시작. 이 메서드는 jnitest.dll의 Java_JniFuncMain_createJniObject()라는 C함수와 연결되어 있다.

`Java_JniFuncMain_createJniObject()` 함수는 자바 객체를 생성하거나 메서드를 호출하는 방식으로 자바코드와 상호작용. 이러한 작업을 지원하기 위해 JNI는 다양한 JNI 함수 제공.

### 자바측 코드 살펴보기 (JniFuncMain.java)

#### JniFuncMain.java 소스 코드의 `JniFuncMain 클래스`

```j
public class Java_JniFuncMain_createJniObject
{
  private static int staticIntField = 300;

  /* 1) 네이티브 라이브러리 로딩(jnifunc.dll을 로드함) */
  static { System.loadLibrary("jnifunc");}

  /* 2) 네이티브 메서드 선언. static으로 선언함으로써 별도 객체 생성할 필요없이 바로
        JNIFuncMain 클래스를 통해 호출 가능.           */
  public static native JniTest createJniObject();

  public static void main(String[] args)
  {
    /* 3) new 연산자를 통한 별도 객체 생성 없이 네이티브 코드로부터 JniTest 객체 생성 */
    System.out.println("[Java] createJniObject() 네이티브 메서드 호출");
    JniTest jniObj = createJniObject();

    /* 4) JniTest 객체의 메서드 호출 */
    jniObj.callTest();
  }
}
```

#### JniFuncMain.java 소스 코드의 `JniTest 클래스`

JNI 네이티브 함수인 `Java_JniFuncMain_createJniObject()`는 JNI 함수를 이용해 JniTest 객체를 생성하고, `callByNative()` 메서드를 호출할 것이다.

```j
class JniTest
{
  private int intField;

  public JniTest(int num) // 생성자.
  {
    intField = num;
    System.out.println("[Java] JniTest 객체의 생성자 호출 : intField = " + intField);
  }

  public int callByNative(int num) // JNI 네이티브 함수로부터 호출될 메서드.
  {
    System.out.println("[Java] JniTest 객체의 callByNative(" + num + ") 호출");
    return num;
  }

  public void callTest()
  {
    System.out.println("[Java] JniTest 객체의 callTest() 메서드 호출 : intField = " + intField);
  }  
}
```

### JNI 네이티브 함수의 코드 살펴보기

#### `JniFuncMain.h` 헤더 파일

JniFuncMain.java 에 선언한 createJniObject() 네이티브 메서드에 대한 실제 구현을 C함수로 작성해본다. `javah JniFuncMain` 명령으로 네이티브 메서드에 연결할 함수 원형을 생성한다. 명령을 실행하고 나면 `JniFuncMain.h` 헤더 파일이 생성된다.

헤더 파일을 살펴보면 createJniObject() 메서드에 대해 다음과 같은 JNI 네이티브 함수의 원형이 생성되었음을 확인할 수 있다. 이 함수는 JniFuncMain 클래스의 `createJniObject()` 네이티브 메서드와 매핑되는 함수이다.

```c
/*
 * Class:   JniFuncMain
 * Method:  createJniObject()
 * Signature: ()LJniTest;
 */
 JNIEXPORT jobobject JNICALL Java_JniFuncMain_createJniObject(JNIEnv *, jclass);
```

JniFuncMain.java 소스 코드의 JniFuncMain 클래스에서 `public static native JniTest createJniObject();` 선언이 static으로 되어있다. 자바의 정적 메서드는 객체를 생성하지 않고도 클래스를 통해 바로 호출이 가능하다. 즉, 네이티브 메서드가 객체가 아닌 클래스를 통해 호출되기 떄문에 두 번째 매개변수의 타입이 `jclass`이다. +) 매개변수 `JNIEnv *` 는 다양한 JNI 함수를 호출하는 JNI 인터페이스 포인터.

따라서 createJniObject() 네이티브 메서드는 JniFuncMain 클래스를 통해 호출되기 때문에 `Java_JniFuncMain_createJniObject()` JNI 네이티브 함수의 두 번째 매개변수로 `JniFuncMain 클래스 레퍼런스`가 넘어오게 된다.

#### `jnifunc.cpp` 파일

아래는 Java_JniFuncMain_createJniObject() JNI 네이티브 함수를 보여주는 jnifunc.cpp 코드이다.

```c++
JNIEXPORT jobject JNICALL Java_JniFuncMain_createJniObject(JNIEnv *env, jclass clazz)
{
  jclass targetClass;
  jmethodID mid;
  jobject newObject;
  jstring helloStr;
  jfieldID fid;
  jint staticIntField;
  jint result;

  // JniFuncMain 클래스의 staticIntField 필드 값 얻어오기
  fid = env->GetStaticFieldID(clazz, "staticIntField", "I");
  staticIntField = env->GetStaticIntField(clazz, fid);
  printf("[CPP] JniFuncMain 클래스의 staticIntField 필드 값 얻어오기\n");
  printf("      JniFuncMain.staticIntField = %d\n", staticIntField);

  // 객체 생성에 필요한 클래스 찾기
  targetClass = env->FindClass("JniTest");

  // 생성자 찾기
  mid = env->GetMethodID(targetClass, "<init>", "(I)V");

  // JniTest 객체 생성(객체 레퍼런스 반환)
  printf("[CPP] JniTest 객체 생성\n");
  newObject = env->NewObject(targetClass, mid, 100);

  // 객체의 메서드 호출하기
  mid = env->GetMethodID(targetClass, "callByNative", "(I)I");
  result = env->CallIntMethod(newObject, mid, 200);

  // JniObject 객체의 intField 값 설정하기
  fid = env->GetFieldID(targetClass, "intField", "I");
  printf("[CPP] JniTest 객체의 intField 필드 값을 200으로 세팅\n");
  env->SetIntField(newObject, fid, result);

  // 생성한 객체를 반환
  return newObject;
}
```

#### JNI를 통한 멤버 필드 값 얻어오기

```c++
// 1) 접근하려는 멤버 변수가 선언된 자바 클래스의 jclass 값을 구한다.

// 2) GetStaticFieldID() JNI 함수로 이 클래스의 멤버 변수에 대한 jfieldID 값을 구한다.
fid = env->GetStaticFieldID(clazz, "staticIntField", "I");
// +) "staticIntField"는 멤버 필드명. "I"는 멤버 필드의 시그너처명으로, int 타입을 나타내는 I.
// +) c++ 스타일이기에 GetStaticFieldID() 인자로 env 전달 제외. c 스타일이리면,
// fid = (*env)->GetStaticFieldID(env, clazz, "staticIntField", "I");    

// 3) jclass와 jfieldID 값을 읽어와 멤버 필드값을 설정한다.
staticIntField = env->GetStaticIntField(clazz, fid);
// +) JNI에서 멤버 필드를 얻어오는 함수는 Get<type>Field or GetStatic<type>Field

printf("[CPP] JniFuncMain 클래스의 staticIntField 필드 값 얻어오기\n");
printf("      JniFuncMain.staticIntField = %d\n", staticIntField);
```

staticIntField 멤버 필드는 JniFuncMain 클래스에 정적으로 선언되어 있다. 이 클래스에 대한 jclass 값은 Java_JniFuncMain_createJniObject() JNI 네이티브 함수의 두 번쨰 인자로 넘어오기 떄문에 그대로 사용한다. 만약, 특정 클래스의 jclass 값을 얻어오고 싶다면 JNI 함수인 `FindClass()`를 이용.

네이티브 코드에서 자바의 멤버 필드에 접근하기 위해서는 해당 멤버 필드에 대한 멤버 필드 ID 값을 구해야 하기 때문에 jfieldID를 구하는 것이다.

***TIP)*** JNI에서 요구하는 멤버 필드나 메서드의 시그너처는 자바 클래스 파일 디스어셈블러인 `javap`를 이용하여 손쉽게 생성할 수 있다.

#### 객체 생성하기

```c++
// 1) 객체 생성에 필요한 클래스 찾기
targetClass = env->FindClass("JniTest");
// Tip) targetClass에 저장된 FindClass()의 반환값은 JniTest의 지역 레퍼런스 값.
//      따라서 이 값은 Java_JniFuncMain_createJniObject() 함수가 호출 된 후 반환되기 전까지만
//      JniTest 클래스를 가리킴. 이 함수가 종료되고 반환되면 해당 값 의미 없어짐.  

// 2) 생성자 찾고 jemethodID 구하기
mid = env->GetMethodID(targetClass, "<init>", "(I)V");
// <init> 이 생성자를 가리킴. 일반적으로 구하려는 메서드 이름을 넘겨야 하지만 생성자는 예외.

// 3) JniTest 객체 생성(객체 레퍼런스 반환)
printf("[CPP] JniTest 객체 생성\n");
newObject = env->NewObject(targetClass, mid, 100);
// JniTest 클래스 생성자는 JniTest(int num)처럼 int형의 인자를 받기 때문에
// NewObject() 함수 호출 시, 100이라는 int형 값을 인자로 넘긴 것.
// 생성된 객체 jobject 변수에 저장.
```

JNI 네이티브 함수에서 자바 메서드를 이용하려면 이용하고자 하는 메서드에 대한 jemethodID 타입의 메서드 ID를 구해야 한다. `GetMethodID()` 함수를 이용하여 구하는데, 이 함수는 생성자 외에도 객체의 여러 멤버 함수의 메서드 ID를 구하는 데 사용할 수 있다.

***TIP)*** JNI 네이티브 함수를 구현할 때 GetObjectClass()나 FindClass() 같은 JNI 함수가 반환값으로 생성하는 jclass, jobject 등의 레퍼런스 값은 `지역 레퍼런스(Local Reference)`에 해당한다. 지역 레퍼런스는 JNI의 디폴트 레퍼런스 값의 형태로, JNI 네이티브 함수 내에서만 레퍼런스가 가리키는 값이 유효하다. 즉 JNI 함수가 반환되면 레퍼런스 값은 *무효* 한 값이 된다.

***TIP)*** JNI는 잘못된 지역 레퍼런스 사용 시 오류가 발생하는 것에 대해 JNI 네이티브 함수에서 레퍼런스 값을 전역적으로 사용할 수 있게 `전역 레퍼런스(Global Reference)` 를 제공한다. 이는 NewGlobalRef() 함수를 통해 생성할 수 있으며, 오류 발생시 Null을 반환한다. 전역 레퍼런스는 사용이 끝난 경우 DeleteGlobalRef() JNI 함수를 통해 명시적으로 소멸시켜주어야 한다.

#### 자바 메서드 호출하기

```c++
// 1) 호출할 메서드가 포함된 자바 클래스의 jclass 값을 구함.
targetClass = env->GetObjectClass(newObject);
// 전체 코드에서 FindClass() JNI 함수를 이용해 targetClass에 저장했기 때문에 이 값을 그대로
// 이용해도 됨. 여기서는 GetObjectClass() JNI 함수 사용법을 설명하기 위해 이 함수 이용.

// 2) 호출할 메서드(여기서는 callByNative)의 메서드 ID 값을 구함.  
mid = env->GetMethodID(targetClass, "callByNative", "(I)I");

// 3) 호출할 메서드의 반환 값 타입에 따라 적당한 JNI 함수를 통해 메서드 호출.
result = env->CallIntMethod(newObject, mid, 200);
// callByNative() 메서드의 반환값과 인자 모두 int 타입이므로 CallIntMethod()와 정수 200 사용.

```

`GetObjectClass()` JNI 함수는 jobject 타입의 객체 레퍼런스 값을 인자로 받아 해당 객체에 해당하는 클래스 레퍼런스 값(jclass) 타입을 반환한다. JniTest 객체에 대한 jobject 값은 newObject 변수에 저장되어 있으므로 GetObjectClass()를 이용하여 JniTest 클래스 레퍼런스 값을 구할 수 있다.

`CallStatic<type>Method()` JNI 함수는 메서드 ID가 가리키는 자바 클래스 메서드 호출, `Call<type>Method()` JNI 함수는 메서드 ID가 가리키는 자바 객체 메서드 호출.

#### JNI를 통한 멤버 필드 값 설정하기

멤버 필드 값을 읽어 오는 과정과 유사하다. Set<type>Field() 함수를 통해 멤버 필드의 값을 설정.

```c++
// 1) intField 멤버 변수를 포함한 JniTest 클래스의 jclass 값을 구함.
// 이미 targetClass에 저장되어 있음.

// 2) JniTest 객체의 "intField"에 대한 필드 ID 값을 구함.
fid = env->GetFieldID(targetClass, "intField", "I");
printf("[CPP] JniTest 객체의 intField 필드 값을 200으로 세팅\n");

// 3) intField의 값을 result 변수의 값으로 설정.
env->SetIntField(newObject, fid, result);
// newObject는 값을 설정할 멤버 변수가 포함된 자바 객체 레퍼런스.
// fid는 값을 설정할 멤버 필드의 ID.
// result는 멤버 필드에 설정할 값.
```

`SetStatic<type>Method()` JNI 함수는 필드 ID에 해당하는 자바 클래스의 정적 멤버 필드의 값을 설정, `Set<type>Method()` JNI 함수는 필드 ID에 해당하는 자바 객체의 멤버 필드의 값을 설정.

---

4.4 C 프로그램에서 자바 클래스 실행하기
---------------------------------------

---

지금까지는 자바 코드가 메인 프로그램이고 자바 쪽 코드에서 네이티브 메서드를 통해 C 함수를 호출해서 JNI를 이용하는 방식을 살펴 보았다면, 이제는 C/C++ 로 구현된 메인 애플리케이션에서 자바 클래스를 실행하는 JNI 이용 방식을 알아본다.

C/C++ 기반의 네이티브 애플리케이션에서 자바 클래스를 실행하려면 자바 가상 머신이 필요하다. 이를 위해 JNI는 네이티브 애플리케이션이 자신의 메모리 영역 내에 자바 가상 머신을 로드할 수 있게끔 `호출 API(inovation API)`라는 것을 제공한다. 이를 활용하여 C코드에서 자바 클래스를 로딩하고 메서드를 실행할 수 있다.

C/C++ 프로그램에서 기존에 작성한 자바 기반의 라이브러리를 이용하고 싶을 때, C/C++ 프로그램에서 자바 표준 라이브러리르 사용하고 싶을 때, 기존의 C/C++ 프로그램이 자바 프로그램과의 상호작용이 자주 필요로 할 때 호출 API를 이용하여 기존 프로그램과 자바 프로그램을 하나의 프로그램으로 만들 수 있다.

### 호출 API 사용 예제: InvokeJava.cpp, InvocationApiTest.java

(그림 4-24 링크 첨부)

#### 자바 코드(InvocationApiTest.java) 살펴보기

```j
public class InvocationApiTest
{
  // 정적 메서드로 문자열 객체 배열을 인자로 받아 그 중 첫번째 문자열을 가리키는 'args[0]'를
  // 화면애 출력하는 단순한 기능 수행.
  public static void main(String[] args)
  {
    System.out.println(args[0]);
  }
}
```

#### C 코드(invocationApi.c) 살펴보기

1.	JavaVMInitArgs 구조체는 내부적으로 JavaVMOption 구조체 배열을 포함. 즉 `JavaVMOption 구조체`는 자바 가상 머신에 전달할 각 옵션의 값을 나타내고 `JavaVMInitArgs`는 이러한 옵션 값을 묶어 자바 가상 머신으로 전달하는 데 사용된다.

2.	ignoreUnrecognized 필드 JNI_TRUE 값을 설정하면 잘못 정의된 옵션이라도 무시하고 자바 가상 머신은 실행을 계속한다. 그러나 이 필드의 값을 JNI_FALSE 값으로 설정하면 자바 가상 머신은 오류를 반환하고 바로 종료한다.

3.	`JNI_CreateJavaVM()` 함수에서 주목해야 할 점은 이 함수의 호출이 처리되면 두 번째 인자로 넘긴 env에는 JNI 인터페이스 포인터의 주소가 저장된다는 것이다. 따라서 이후부터는 env가 가리키는 JNI 인터페이스 포인터를 통해 우리가 지금껏 살펴본 각종 JNI 함수를 이용할 수 있다. 이를 통해 C/C++ 코드에서 자바의 객체를 생성하거나 메서드를 호출하는 등의 일을 할 수 있다.

```c
// C코드에서 JNI를 사용하는 데 필요한 각종 변수 타입이나 JNI 함수가 정의되어 있으며,
// 네이티브 코드에서 JNI를 이용하려면 무조건 이 헤더파일을 포함해야 함.
#include <jni.h>

int main()
{
  JNIEnv *env;
  JavaVM *vm;
  JavaVMInitArgs vm_args;   // jni.h에 정의되어 있음.
  JavaVMOption options[1];  // jni.h에 정의되어 있음. 옵션 하나만 지정할 것이기에 원소 1.
  jint res;
  jclass cls;
  jmethodID mid;
  jstring jstr;
  jclass stringClass;
  jobjectArray args;

  // 1) JavaVMInitArgs 구조체인 vm_args와 JavaVMOption 구조체인 options 사용하여 자바 가상 머신에 전달할 옵션 값을 생성. *** 1 참고 ***
  // 여기서 설정한 옵션은 자바 가상 머신의 환경을 설정하거나 동작을 제어하는 데 쓰임.
  options[0].optionString = "-Djava.class.path=."; // 자바 가상머신이 클래스를 로딩할 디폴트 디렉터리를 현재 디렉터리로(.) 설정.
  vm_args.version = 0x00010002; // 자바 가상 머신에게 넘긴 옵션의 매개변수의 형식을 지정.
  vm_args.options = options; // JavaVMInitArgs에서 가리킬 JavaVMOption 구조체 배열의 주소.
  vm_args.nOptions = 1; //  JavaVMInitArgs에서 가리킬 JavaVMOption 구조체 배열의 원소 개수.
  vm_args.ignoreUnrecognized = JNI_TRUE; // 자바 가상 머신이 잘못 정의된 옵션 값을 만났을 경우 무시하고 계속 진행. *** 2 참고 ***

  // 2) C 애플리케이션에서 JNI_CreateJavaVM() 호출 API 이용하여 자바 가상머신 로드.
  res = JNI_CreateJavaVM(&vm, (void**)&env, &vm_args);
  // vm의 타입인 JavaVM은 자바 가상 머신을 생성하거나 종료하는 자바 가상 머신 인터페이스. JavaVM의 포인터 주소 지정.
  // 함수의 호출이 처리되면 두 번째 인자로 넘긴 env에는 JNI 인터페이스 포인터의 주소가 저장.  *** 3 참고 ***
  // 자바 가상 머신에게 JavaVMInitArgs 구조체 vm_args 넘김.

  // 3) FindClass() 함수로 실행할 클래스 검색 후 로드.
  cls = (*env)->FindClass(env, "InvocationApiTest");

  // 4) 로드한 클래스 안의 main() 메서드 호출 위해 메서드 ID 획득.
  mid = (*env)->GetStaticMethodID(env, cls, "main", "([Ljava/lang/String;)V");

  // 5) 자바의 main 클래스 메서드의 인자로 넘겨줄 문자열 객체 생성.
  jstr = (*env)->NewStringUTF(env, "Hello Invocation API!!");
  stringClass = (*env)->FindClass(env, "java/lang/String");
  args = (*env)->NewObjectArray(env, 1, stringClass, jstr);
  // NewStringUTF() 함수로 UTF-8 형식의 C 문자열을 자바의 문자열 타입인 String 객체로 변환.
  // NewObjectArray JNI 함수를 이용해 String 객체의 배열 args 생성 후, 앞에서 만든 String 객체 jstr로 초기화. 이로써 args의 첫 원소에 문자열 저장.
  // NewObjectArray() 함수의 1은 생성한 배열의 원소 개수, stringClass는 배열로 만들 클래스 타입,  jstr은 배열의 초기값. 함수 실행 성공시 생성한 배열의 레퍼런스 리턴.

  // 6) InvocationApiTest의 main() 메서드를 호출
  (*env)->CallStaticVoidMethod(env, cls, mid, args);
  // 위에서 생성한 args를 CallStaticVoidMethod의 인자로 넘겨서 출력하게 됨.

  // 7) 자바 가상 머신 소멸(종료).
  (*vm)->DestroyJavaVM(vm);
}
```
