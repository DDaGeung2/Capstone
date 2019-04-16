---
title: 스터디 4주차 / 5장
date: '2019-04-16 17:00:00 -0400'
categories: study capstonedesign
published: true
---

### Zygote란 무엇인가

안드로이드 시스템에서 새로운 애플리케이션을 실행하면 실행에 필요한 요소들을 미리 준비해준 **Zygote** 프로세스와 새로운 애플리케이션이 결합되어 실행.

Zygote 프로세스는 실행되면서 달빅 가상 머신을 초기화하고 구동. 이 프로세스는 애플리케이션이 실행되기 전에 실행된 가상 머신의 코드 및 메모리 정보를 공유함으로써 애플리케이션이 실행되는 시간을 단축.

또한 안드로이드 프레임워크에서 동작하는 애플리케이션이 사용할 클래스와 자원을 미리 메모리에 로딩해두고 이러한 자원에 대한 연결정보를 구성해놓기에, 새로 실행되는 안드로이드 애플리케이션은 필요한 자웓들에 대한 연결정보를 매번 새롭게 구성하지 않고 그대로 사용할 수 있어 빠르게 실행.

**목적:** *한정된 자원으로 효율적으로 동작한다* 는 임베디드 기기 플랫폼의 중요 성능 지표를 달성하기 위해 Zygote 프로세스를 사용.

#### Zygote를 통한 프로세스 생성

Zygote 프로세스는 init 프로세스가 시스템 구동에 필요한 각종 데몬을 실행하고 난 뒤에 실행. 실행 이후에 안드로이드 서비스 및 애플리케이션이 이를 통해 실행.

안드로이드 기기에서 실행 중인 프로세스는 부모 프로세스의 PID(PPID)를 기준으로 `데몬(PPID = 1)`과 `안드로이드 애플리케이션`으로 구분. 데몬 프로세스 안에 Zygote 프로세스 포함.

**안드로이드에서 새 애플리케이션 프로세스를 생성해서 실행하는 경우**

![Scanner_IMG_2019-04-16 07-37-33](https://user-images.githubusercontent.com/48465809/56204804-43716d00-6083-11e9-8a50-a5de9a6076cd.jpg)

1) fork() 시스템 콜 호출하여 자식 프로세스인 Zygote' 프로세스 생성.

2) 생성된 자식프로세스는 부모 프로세스의 코드 영역과 링크 정보 공유.

3) 새로운 애플리케이션 A는 fork() 통하여 생성된 프로세스의 코드 영역을 복제된 달빅 가상머신에 동적으로 로딩.

4) 자식프로세스가 애플리케이션 A 클래스의 메서드로 실행흐름 넘기고 안드로이드 애플리케이션 동작.

애플리케이션 A는 기존의 Zygote 프로세스가 구성해놓은 라이브러리 및 리소스에 대한 링크정보 그대로 사용하므로 빠름.

*"Zygote는 COW(Copy On Write)를 통해 기존에 이미 메모리 상에서 동작중인 프로세스의 재사용성을 극대화하고 공유 라이브러리를 통해 메모리 사용량을 최소화한다."*

`COW: 공유하는 메모리 정보를 자식 프로세스가 수정하는 시점에서 부모 프로세스의 메모리 정보를 자신의 메모리 공간을 복사하는 기법.`

#### app_process로 부터 ZygoteInit class 실행

Zygote는 자바로 작성되어있기에 init 프로세스에서 바로 실행 불가. **app_process** 로 달빅 가상 머신 위에 ZygoteInit 클래스 로딩하고 실행.

**app_process 동작 순서**

1) main() 함수에서 AppRuntime 객체 생성: AppRuntime은 AndroidRuntime 상속.

2) main() 함수에 전달된 인자 값 분석해서 AppRuntime 객체에 전달.

```c
int main(int argc, const char* const argv[])
{
  ...
  if(i < argc) {
    arg = argv[i++];
    if (0 == strcmp("--zygote",arg)) { // 실행 클래스 이름 확인.
      bool startSystemServer = (i < argc) ?
        strcmp(argv[i], "--start-system-server") == 0 : false;
      setArgv0(argv0, "zygote");
      set_process_name("zygote");

      // start() 함수로 가상머신 생성 및 초기화.
      // 첫번째 인자는 클래스 로더로 전달하는 경로명.
      runtime.start("com.android.internal.os.ZygoteInit",
                    startServer);
    } else {
      ...
    }
    ...
  }
}
```

3) 달빅 가상 머신 초기화 : 가상 머신 실행 옵션 설정 위해 `property_get()` 함수 호출. AndroidRuntime의 `JNI_CreateJAvaVM()` 함수를 통해 달빅 가상 머신 생성 및 실행.

4) ZygoteInit 클래스의 main() 메서드 호출.

#### ZygoteInit 클래스의 기능

![Scanner_IMG_2019-04-16 07-36-22](https://user-images.githubusercontent.com/48465809/56204686-fab9b400-6082-11e9-8f8f-38f8ab8f846b.jpg)

**ZygoteInit.java의 main() 메서드**

```c
public static void main(String argv[]) {
  try {
    // 1) 새로운 안드로이드 애플리케이션의 실행 요청을 받기 위한 소켓 바인딩
    registerZygoteSocket();

    ...
    // 2) 안드로이드 애플리케이션 프레임워크에서 사용할 클래스와 리소스 로딩
    preloadClasses();
    preloadResources();

    ...
    // 3) SystemServer 실행 -> 주요한 네이티브 서비스들 실행
    if (argv[1].equals("true")) {
      startSystemServer();
    }

    ...
    // 모니터링
    if (ZYGOTE_FORK_MODE) {
      runForkMode();
    } else {
      // 4) 새로운 안드로이드 애플리케이션 실행 요청을 처리
      runSelectLoopMode();
    }
    closeServerSocket();
  } catch (MethodAndArgsCaller caller) {
      caller.run();
  } catch (RuntimeException ex) {
    Log.e(TAG, "Zygote died with exception", ex);
    closeServerSocket();
    throw ex;
  }
}
```

##### 1) /dev/socket/zygote 소켓 바인딩

-	ZygoteInit 클래스는 /dev/socket/zygote에 생성된 유닉스 도메인 소켓을 사용해서 ActivityManager로부터 전달되는 새 안드로이드 애플리케이션의 생성 요청 메시지를 수신.

-	소켓은 부팅과정에서 init 프로세스에 의해 생성.

##### 2) 애플리케이션 프레임워크에 속한 클래스와 플랫폼 자원의 로딩

-	`preloadClasses()` , `preloadResources()` 함수로 자원을 메모리에 로딩 및 이에 대한 연결정보를 생성하는데, 이후 새로 생성되는 안드로이드 애플리케이션에서는 미리 로딩한 클래스나 자원을 이용할 때 새로 연결정보 생성 없이 *그대로 이용* .

-	ZygoteInit 클래스는 공통적으로 사용되는 클래스들을 한꺼번에 미리 로딩하여 두고, 매번 애플리케이션이 실행될 때마다 로딩하는 시간을 줄이게 되어, 애플리케이션의 시작을 빠르게 함.

-	마찬가지로 안드로이드 애플리케이션에서는 사용할 리소스도 미리 로딩해두기 떄문에 해당 리소스를 사용할 때마다 저장 장치에서 메모리로 로딩하는 시간을 줄일 수 있음.

-	**ZygoteInit.java의 `preloadClasses()` 메서드 주요부분**

```c
...
private static final String PRELOADED_CLASSES = "preloaded-classes";
...

private static void preloadClasses() {
  ...
  // 1) 입력 스트림 생성해서 preloaded-classes 파일에 기술된 클래스 목록 읽어옴.
  InputStream is =
    ZygoteInit.class.getClassLoader().getResourceAssStream(
        PRELOADED_CLASSES);
  ...
  // 2) 한 줄씩 읽어들임
  BufferedReader br
    = new BufferedReader(new InputStreamReader(is), 256);

  int count = 0;
  String line;
  while ((line = br.readLine()) != null) {
    // 3) 읽어들인 코드가 주석이나 빈 줄이면 처리없이 다음 줄 읽음
    line = line.trim();
    if (line.startsWith("#") || line.equals("")) {
      continue; }
    ...
    // 4) Class.forName() 메서드로 필요한 클래스를 메모리에 동적 로딩.
    //    실제로 메모리 상에 해당 클래스의 인스턴스 생성 X. 메모리에 로딩하고 정적 변수 초기화.
    Class.forName(line);
  }
}
```

-	**ZygoteInit.java의 `preloadResource()` 메서드 주요부분**

```c
private static void preloadResource() {
  ...
  mResources = Resources.getSystem(); // 1) 시스템 리소스에 접근
  mResources.startPreloading(); // 2) 중복호출로 인한 재로딩 방지

  // 3) Drawble, Color State 리소스 로딩
  if(PRELOAD_RESOURCES) {
    TypedArray ar = mResources.obtainTypedArray(
      com.android.internal.R.array.preloaded_drawables);
    int N = preloadDrawables(runtime, ar);
    ...

    ar = mResources.obtainTypedArray(
      com.android.internal.R.array.preloaded_color_state_lists);
    N = preloadColorStateLists(runtime, ar);
  }
  mResources.finishPreloading();
}
```

##### 3) SystemServer 실행

-	Zygote에서 달빅 가상 머신을 구동한 후, 시스템 서버(SystemServer)라는 자바 서비스를 실행하기 위해 새로운 달빅 가상 머신 인스턴스 생성.

-	시스템 서버는 필요한 네이티브 서비스를 실행. 이들이 실행되고 나면 시스템 서버는 안드로이드 프레임워크의 서비스들을 시작. ex) 액티비티 매니저, 패키지 매니저 등.

-	ZygoteInit.java의 startSystemServer() 함수에서 `forkSystemServer()` 메서드를 통해 새로운 프로세스를 생성하고 시스템 서버 실행. 이 메서드로 생성한 시스템 서버 프로세스의 동작 여부 확인.

##### 4) 새로운 안드로이드 애플리케이션 실행

![Scanner_IMG_2019-04-16 07-35-01](https://user-images.githubusercontent.com/48465809/56203631-95fd5a00-6080-11e9-91df-66169a39158e.jpg)

-	시스템 서버가 실행되고 나면 앞서 바인딩한 소켓으로 들어오는 요청을 처리하기 위한 루프 실행. `runSelectLoopMode()`

```c
private static void runSelectLoopMode() throws MethodAndArgsCaller {
  ...

  // 1) 앞서 바인딩한 소켓 디스크립터를 디스크립터 배열에 추가.
  //    이를 이용해서 외부에서 발생하는 연결 요청을 처리.
  fds.add(sServerSocket.getFileDescriptor());
  while (true) {
    ...

    // 2) selectReadable(): JNI 네이티브 함수. 인자로 전달되는 파일 디스크립터 배열 감시하다가
    //    디스크립터 목록에 이벤트 있으면 배열의 해당 인덱스 반환.
    index = selectReadable(fdArray);
    ...
    if (index < 0) {
      throw new RuntimeException("Error in select()");
    } else if (index == 0) {

      // 3) 0번째 인덱스에는 소켓으로 전달된 새로운 연결요청을 처리하기 위해
      //    ZygoteConnection 객체를 생성.
      ZygoteConnection newPeer = acceptCommandPeer();
      peers.add(newPeer);

      // 3-1) ZygoteConnection 인스턴스의 입출력 이벤트 처리 위해 배열에 추가.
      fds.add(newPeer.getFileDescriptor());
    } else {
      boolean done;

      // 4) 새롭게 연결된 입출력 소켓 처리.
      //    새로운 안드로이드 애플리케이션 생성.
      done = peers.get(index).runOnce();

      if (done) {
        peers.remove(index);
        fds.remove(index);
      }
    }
  }
}
```
