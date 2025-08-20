[창경] 창조경제혁신센터 공지사항 업로드 관련 이슈 정리
Heap Dump 적용 가이드
1. Heap Dump란?

JVM의 힙 메모리 상태를 그대로 덤프한 파일

객체 정보, 참조 구조, 메모리 사용량 등을 확인할 수 있는 스냅샷

2. Heap Dump를 만드는 이유

메모리 문제 분석을 위해 사용됨

주요 사례

OOM(OutOfMemoryError) 분석

메모리 부족으로 OOM 발생 시 어떤 객체가 메모리를 차지하는지 확인

메모리 누수(Memory Leak) 확인

GC로 해제되지 못하고 계속 남아 있는 객체 파악

메모리 최적화

객체 할당 구조 및 패턴 분석, 개선 포인트 확인

3. 참고사항

운영 서버 제어

sudo systemctl start/stop node10,20,30.service


개발 서버 제어

start.sh / stop.sh

4. 운영 및 개발서버 PID 정리

PID 확인

jps -l


PID별 힙 크기 확인

jcmd <PID> VM.flags

서버	PID	MaxHeapSize	힙 크기
운영 node10	3764120	2147483648	2G
운영 node20	3766664	4294967296	4G
운영 node30	3771163	2147483648	2G
개발 node10	2394700	4150263808	3.8G
개발 node20	2394702	4150263808	3.8G
개발 node30	2669255	4150263808	3.8G
5. Heap Dump 적용

Dump 파일 경로

/DATA/tomcat9/domain/node10,20,30/Logs


setenv.sh 추가 명령어

CATALINA_OPTS="$CATALINA_OPTS -Xms2g -Xmx3.8g \
-Xloggc:/DATA/tomcat9/domain/node30/logs/gc.log \
-XX:+PrintGCDetails \
-XX:+PrintGCDateStamps \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof"
export CATALINA_OPTS

6. 개발서버 TEST 진행 방법 (실제 OOM 발생시키기)

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

java.lang.OutOfMemoryError: Java heap space 발생 시

동일 경로에 heapdump.hprof 파일과 gc.log 파일 생성됨

7. 운영서버 TEST 진행 방법

JVM 플래그 확인

jcmd <PID> VM.flags


-XX:+HeapDumpOnOutOfMemoryError

-XX:HeapDumpPath=/DATA/tomcat9/domain/node30/logs/heapdump.hprof

Tomcat PID 확인

ps -ef | grep java


GC 로그 설정 확인

-Xloggc:/DATA/tomcat9/domain/node30/logs/gc.log \
-XX:+PrintGCDetails \
-XX:+PrintGCDateStamps


✅ 정리 요약

Heap Dump는 메모리 문제 분석 필수 도구

OOM 발생 시 자동으로 .hprof 파일 생성되도록 JVM 옵션 설정 필요

개발/운영 서버 모두 setenv.sh에 HeapDumpOnOutOfMemoryError와 HeapDumpPath 지정 완료

테스트 코드로 OOM 유발 → Heap Dump 생성 정상 확인 완료
