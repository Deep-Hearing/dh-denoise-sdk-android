# SDK 적용 방법

> Android를 위한 딥히어링 노이즈 제거 C/C++ 라이브러리 적용 가이드입니다.
> 
> 
> 현재 지원되는 ABI는 [armeabi-v7a](https://developer.android.com/ndk/guides/abis#v7a), [arm64-v8a](https://developer.android.com/ndk/guides/abis#arm64-v8a) 입니다. 
> 

## 안드로이드 프로젝트 설정

프로젝트 설정에 앞서 아래와 같은 파일이 필요합니다

- `libdh_denoise.a` : 딥히어링 노이즈 제거 라이브러리
- `dh_denoise.h` : 딥히어링 노이즈 제거 라이브러리 헤더
- `jni_bridge.c` : JNI 연결 코드가 작성된 C 소스

 안드로이드 프로젝트에서 딥히어링 노이즈 제거 라이브러리(C/C++ 라이브러리)를 사용하기 위해서는 NDK 설정이 필요합니다.

### NDK

[](https://developer.android.com/ndk/guides)

기존 프로젝트에서 NDK를 사용하지 않는 경우 [가이드](https://developer.android.com/ndk/guides)를 통해 NDK를 시작할 수 있습니다.

### Gradle

`Gradle` 설정 방법입니다. `src/main` 디렉토리 하위에 `jniLibs/${ABI}` 폴더를 생성하고 딥히어링 노이즈 제거 라이브러리를 위치시켜 줍니다.

빌드 시, 해당 파일들을 사용할 수 있도록 `build.gradle` 파일에 아래와 같이 작성해줍니다.

```
android {
    ...

    defaultConfig {
	      ...
        ndk.abiFilters 'arm64-v8a'
    }

    ...

    sourceSets {
        main {
            jni.srcDirs = ['src/main/jniLibs/']
        }
    }

    packagingOptions {
        exclude "src/main/jniLibs/arm64-v8a/libdh_denoise.a"
    }
}
```

### CMake

`src/main/cpp` 디렉토리 하위에 위치하는 `CMakeLists.txt` 파일에 다음과 같이 작성합니다.

1. `add_library()` 로 미리 빌드된 `dh_denoise` 라이브러리를 프로젝트로 가져옵니다.
2. `set_target_properties()` 로 라이브러리의 경로를 지정합니다

💡 자세한 내용은 다음 <a href='https://developer.android.com/studio/projects/configure-cmake?hl=ko#add-other-library'>[안드로이드 공식 문서]</a>에서 참고할 수 있습니다.

```
...
set(DH_LIB_DIR ${CMAKE_SOURCE_DIR}/../jniLibs)
...

# 1번에 해당하는 내용입니다.
add_library(dh_denoise STATIC IMPORTED) 

# 2번에 해당하는 내용입니다.
set_target_properties( # Specifies the target library.
                       dh_denoise

                       # Specifies the parameter you want to define.
                       PROPERTIES IMPORTED_LOCATION

                       # Provides the path to the library you want to import.
                       ${DH_LIB_DIR}/${ANDROID_ABI}/libdh_denoise.a)

target_link_libraries( # Specifies the target library.
                       jni_bridge

                       # Links the target library to the log library
                       # included in the NDK.
                       dh_denoise ... )
```

## Java 에서 C/C++ 라이브러리 함수 호출

아래와 같이 Java Enum Singletone을 구현하여 JNI에 작성된 C Native 함수를 호출 할 수 있습니다.

```java
public enum DenoiseEngine {

    INSTANCE;

    static {
        System.loadLibrary("jni_bridge");
    }

    // Native methods
    static native long initModel();
    static native void deleteModel(long modelPtr);
    static native float[] denoise(long modelPtr, float[] audioData, int sampleSize, int enableBypass);
}
```

**Example : C Native 함수 호출에 대한 예시**

```java
long modelPtr = DenoiseEngine.initModel();

...

DenoiseEngine.denoise(modelPtr, audioBuffer, FRAME_SIZE, TRUE);

...

DenoiseEngine.deleteModel(modelPtr);
```

## 노이즈 제거 라이브러리 API

다음은 딥히어링 노이즈 제거 라이브러리 API에 관한 내용입니다.

### `initModel()`

```java
public native long initModel();
```

필요한 리소스 할당 및 초기화 작업을 진행합니다. 

초기화된 딥히어링 노이즈 제거 모델의 포인터를 long type으로 반환합니다.

⚠️  **라이브러리의 만료 기간이 다 된 경우에는 NULL pointer를 반환합니다.**

```java
long modelPtr = DenoiseEngine.initModel();

if (modelPtr == 0) {
    toastMessage("Library expired");
    return;
}
```

### `deleteModel()`

```java
public native void deleteModel(long modelPtr);
```

딥히어링 노이즈 제거 모델의 리소스를 할당 해제 합니다.

### `denoise()`

```java
public native float[] denoise(long modelPtr, float[] audioData, int sampleSize, int enableBypass);
```

입력된 오디오 데이터에 대해 노이즈 제거 처리를 합니다.

enableBypass 를 통해서 노이즈 제거를 on/off 할 수 있습니다.

## 오디오 포맷

현재 지원되는 오디오 format은 다음과 같습니다.

**샘플레이트(SampleRate)**

현재 지원되는 샘플레이트는 `16kHz/48kHz` 입니다. 다른 샘플레이트를 사용하는 경우 리샘플링이 필요합니다.

**프레임 크기(Frame Size)**

`8ms` 길이의 오디오 프레임을 처리합니다. 16kHz 기준 `128`개, 48kHz 기준 `384`개 입니다.

**채널(Channel)**

현재 노이즈 제거는 `1개` 채널(`mono`)로 처리됩니다.

**데이터 타입(Data Type)**

데이터 타입은 `Float32`이며 `-1.0~1.0` 사이의 값을 가집니다.
