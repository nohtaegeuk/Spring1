### 📌 Heap Dump 적용 가이드
## 1. Heap Dump란?

JVM의 힙 메모리 상태를 그대로 덤프한 파일

객체 정보, 참조 구조, 메모리 사용량 등을 확인할 수 있는 스냅샷

* Dump 의 컴퓨터 용어적 의미 : **'자료를 복사하거나 출력하다'**

---

## 2. Heap Dump 목적

메모리 문제 분석을 위해 사용

🛑 OOM(OutOfMemoryError) 분석 → 어떤 객체가 메모리를 점유하는지 확인

🧩 Memory Leak 확인 → GC로 회수되지 못하고 남는 객체 파악

⚡ 메모리 최적화 → 객체 할당 패턴 분석 및 개선

---

## 3. Heap 용량 확인 방법 및 JVM Heap 영역 구조

```bash
jmap -heap <PID>
```

# A. Young Generation (신세대) 👉 새로 생성된 객체가 처음 저장되는 공간

# 🌱 Eden Space
- 모든 객체는 처음 **Eden**에 생성됨  
- Eden이 가득 차면 **Minor GC(Young GC)** 발생  
- GC 후 살아남은 객체는 **Survivor 영역**으로 이동  

---

# 🔄 From Space / To Space (Survivor 영역)
- Eden에서 살아남은 객체가 이동하는 공간  
- **두 개의 Survivor 영역(From / To)** 이 번갈아 사용됨  
| 항목        | 역할                                      |
|------------|-----------------------------------------|
| From Space | 현재 살아남은 객체가 임시로 있는 영역         |
| To Space   | 다음 GC에서 객체를 복사할 대상 영역          |
| 특징        | 두 영역은 번갈아 사용되며, 실제 구조는 동일   |

- 여러 번 GC를 거쳐도 살아남은 객체는 결국 **Old Generation**으로 승격(Promotion)

# B. Old Generation (구세대) 👉 오래 살아남은 객체가 저장되는 공간

Survivor에서 여러 번 GC를 거쳐도 살아남으면 Old Gen으로 이동

큰 객체(Large Object)도 직접 Old Gen에 들어갈 수 있음

Old Gen이 가득 차면 Major GC(Full GC)가 발생


   +-------------------+
   |       Eden        |  ← 새 객체 생성
   |   (Young Gen)     |
   +-------------------+
           |
    Minor GC 발생
           |
           v
+-----------------------+
|     Survivor Area      |  ← Eden에서 살아남은 객체 이동
|  +-------------+      |
|  |  From Space | ← 현재 GC에서 사용
|  +-------------+      |
|  |  To Space   | ← 다음 GC 대비
|  +-------------+      |
+-----------------------+
           |
    여러 번 살아남은 객체
    v
    +-------------------+
    | Old Generation | ← 장수 객체 저장
    | (Full GC 대상) |
    +-------------------+

---

### 5. Heap Dump 적용 방법

Dump 파일 경로

/DATA/tomcat9/domain/node10,20,30/logs

setenv.sh 설정 예시
```bash
CATALINA_OPTS="$CATALINA_OPTS -Xms2g -Xmx3.8g \
-Xloggc:/DATA/tomcat9/domain/node30/logs/gc.log \
-XX:+PrintGCDetails \
-XX:+PrintGCDateStamps \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof"
export CATALINA_OPTS
```

| 옵션         | 설명                            |
| ---------- | ----------------------------- |
| `-Xms2g`   | JVM 시작 시 힙(Heap) 초기 크기 2GB 지정 |
| `-Xmx3.8g` | JVM 최대 힙 크기 3.8GB 지정          |
| `-Xloggc:/DATA/.../gc.log` | GC 로그를 지정한 파일에 기록                |
| `-XX:+PrintGCDetails`      | GC 발생 시 상세 정보(Heap 영역별 사용량 등) 출력 |
| `-XX:+PrintGCDateStamps`   | GC 발생 시 시간 정보 포함                 |
| `-XX:+HeapDumpOnOutOfMemoryError`           | OOM 발생 시 자동으로 Heap Dump(.hprof) 생성 |
| `-XX:HeapDumpPath=/DATA/.../heapdump.hprof` | Heap Dump 파일 저장 경로 지정              |


---

### 6. 개발 서버 테스트 (OOM 유발)

테스트 코드 (OOMTest.java)

```java
package com.pkg.OOMT;
import java.util.ArrayList;
import java.util.List;

public class OOMTest {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            list.add(new byte[1024 * 1024]); // 1MB 할당
            System.out.println("Allocated = " + list.size() + " MB");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

실행 명령어

```bash
java -cp . -Xms2g -Xmx3g \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof \
com.pkg.OOMT.OOMTest
```

결과
OutOfMemoryError: Java heap space 발생 시
동일 경로에 heapdump.hprof & gc.log 파일 생성

---

### 7. 운영 서버 확인 절차

JVM 플래그 확인
```bash
jcmd <PID> VM.flags
```

-XX:+HeapDumpOnOutOfMemoryError

-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof

Tomcat PID 확인
```bash
ps -ef | grep java
```

GC 로그 설정

-Xloggc:/DATA/tomcat9/domain/node30/logs/gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps

---

### ✅ 최종 정리

Heap Dump는 메모리 문제 분석 필수 도구

OOM 발생 시 자동 .hprof 파일 생성되도록 JVM 옵션 필수

개발/운영 서버 모두 setenv.sh에 HeapDump 옵션 반영 완료

개발 서버 OOMTest 실행으로 정상 작동 검증 완료
