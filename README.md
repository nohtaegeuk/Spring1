📌 Heap Dump 적용 가이드
1. Heap Dump란?

JVM의 힙 메모리 상태를 그대로 덤프한 파일

객체 정보, 참조 구조, 메모리 사용량 등을 확인할 수 있는 스냅샷

2. Heap Dump 목적

메모리 문제 분석을 위해 사용

🛑 OOM(OutOfMemoryError) 분석 → 어떤 객체가 메모리를 점유하는지 확인

🧩 Memory Leak 확인 → GC로 회수되지 못하고 남는 객체 파악

⚡ 메모리 최적화 → 객체 할당 패턴 분석 및 개선

3. 서버 제어 방식

운영 서버

sudo systemctl start|stop node10,20,30.service


개발 서버

./start.sh | ./stop.sh

4. 서버별 PID & 힙 크기
서버	PID	MaxHeapSize	힙 크기
운영 node10	3764120	2147483648	2G
운영 node20	3766664	4294967296	4G
운영 node30	3771163	2147483648	2G
개발 node10	2394700	4150263808	3.8G
개발 node20	2394702	4150263808	3.8G
개발 node30	2669255	4150263808	3.8G

🔍 확인 방법

jps -l               # PID 확인
jcmd <PID> VM.flags  # 힙 크기 확인

5. Heap Dump 적용 방법

Dump 파일 경로

/DATA/tomcat9/domain/node10,20,30/logs


setenv.sh 설정 예시

CATALINA_OPTS="$CATALINA_OPTS -Xms2g -Xmx3.8g \
-Xloggc:/DATA/tomcat9/domain/node30/logs/gc.log \
-XX:+PrintGCDetails \
-XX:+PrintGCDateStamps \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof"
export CATALINA_OPTS

6. 개발 서버 테스트 (OOM 유발)

테스트 코드 (OOMTest.java)

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


실행 명령어

java -cp . -Xms2g -Xmx3g \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof \
com.pkg.OOMT.OOMTest


결과
OutOfMemoryError: Java heap space 발생 시
동일 경로에 heapdump.hprof & gc.log 파일 생성

7. 운영 서버 확인 절차

JVM 플래그 확인

jcmd <PID> VM.flags


-XX:+HeapDumpOnOutOfMemoryError

-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof

Tomcat PID 확인

ps -ef | grep java


GC 로그 설정

-Xloggc:/DATA/tomcat9/domain/node30/logs/gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps

✅ 최종 정리

Heap Dump는 메모리 문제 분석 필수 도구

OOM 발생 시 자동 .hprof 파일 생성되도록 JVM 옵션 필수

개발/운영 서버 모두 setenv.sh에 HeapDump 옵션 반영 완료

개발 서버 OOMTest 실행으로 정상 작동 검증 완료
