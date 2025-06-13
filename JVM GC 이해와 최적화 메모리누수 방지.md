# JVM 가비지 컬렉션 (GC) 이해와 최적화, 메모리 누수 방지

JVM(Java Virtual Machine)의 핵심 기능인 가비지 컬렉션(GC)의 원리, 다양한 GC 알고리즘, 그리고 효율적인 GC를 위한 튜닝 방법 및 메모리 누수 방지 전략에 대해 깊이 있게 다뤄봅니다.

---

## 1. 가비지 컬렉션(GC)이란?

**가비지 컬렉션(Garbage Collection, GC)**은 자바 개발자가 직접 메모리를 관리(할당/해제)할 필요 없이, **JVM이 더 이상 사용되지 않는 객체(가비지)를 자동으로 찾아 메모리에서 회수하는 과정**을 의미합니다. 이 작업을 수행하는 주체를 **가비지 컬렉터(Garbage Collector)**라고 부릅니다.

**GC의 역할:**
* **메모리 자동 관리:** 개발자의 메모리 관리 부담을 줄여 생산성을 높입니다.
* **메모리 누수 방지 (부분적):** '도달 불가능한(unreachable)' 객체를 자동으로 제거하여 메모리 부족 문제를 예방합니다.

### 힙(Heap) 메모리 영역 구분

JVM 힙 메모리는 효율적인 GC를 위해 크게 두 가지 영역으로 나뉩니다.

* **Young Generation (Young 영역):**
    * 새로 생성된 객체가 할당되는 공간입니다. 대부분의 객체는 생성되자마자 수명이 짧게 끝납니다.
    * **Eden 영역:** 새로운 객체가 처음 생성되는 곳.
    * **Survivor 0 (S0) / Survivor 1 (S1) 영역:** Eden에서 Minor GC 후 살아남은 객체들이 잠시 머무는 공간.

* **Old Generation (Old 영역):**
    * Young 영역에서 여러 번의 GC를 거치면서도 살아남아 **수명이 길다고 판단된 객체(오래된 객체)**들이 이동(승격, Promotion)되는 공간입니다. Young 영역보다 훨씬 큽니다.

## 2. Minor GC, Major GC, Full GC

힙 영역에 따라 GC의 종류와 특징이 달라집니다.

* **Minor GC (Young GC):**
    * **대상:** **Young Generation (Eden + Survivor 영역)**
    * **시점:** Eden 영역이 가득 찼을 때 발생합니다.
    * **특징:**
        * **빈번하게 발생:** 짧은 수명 객체가 많기 때문에 자주 일어납니다.
        * **빠른 속도:** 대상 영역이 작고 쓰레기 객체가 많아 GC 시간이 매우 짧습니다 (수 ms ~ 수십 ms).
        * **Stop-The-World (STW):** 애플리케이션 스레드를 일시적으로 멈추지만, 그 시간이 매우 짧아 체감하기 어렵습니다.

* **Major GC (Old GC):**
    * **대상:** **Old Generation (Old 영역)**
    * **시점:** Old 영역이 가득 찼을 때 주로 발생합니다.
    * **특징:**
        * **드물게 발생:** Minor GC보다 훨씬 드물게 발생합니다.
        * **상대적으로 느림:** Old 영역이 커서 GC 시간이 Minor GC보다 오래 걸립니다 (수백 ms ~ 수 초).
        * **Stop-The-World (STW):** Minor GC보다 훨씬 긴 STW를 유발하며, 애플리케이션 응답성에 영향을 줄 수 있습니다.

* **Full GC (전체 힙 GC):**
    * **대상:** **힙(Heap) 메모리 전체** (Young Generation + Old Generation + 필요하다면 Metaspace/PermGen)
    * **시점:** Major GC가 Old 영역을 정리해야 할 때 대부분 Full GC로 이어지며, 메타스페이스 부족 시에도 발생합니다.
    * **특징:**
        * **가장 느림:** 힙 전체를 스캔하므로 가장 긴 시간이 소요됩니다.
        * **가장 치명적:** Full GC가 너무 자주 발생하거나 한 번에 오래 걸리면 애플리케이션이 장시간 멈추기 때문에, 서비스 안정성에 심각한 영향을 미칩니다. **애플리케이션 튜닝의 주 목표는 Full GC의 발생 빈도와 시간을 최소화하는 것입니다.**

---

## 3. 가비지 컬렉션(GC) 최적화 방법

GC 최적화는 애플리케이션의 성능과 안정성을 향상시키는 핵심 요소입니다.

### 3.1. JVM 옵션 튜닝

GC 관련 옵션은 `java` 명령어 실행 시 JVM 인수로 추가합니다.

* **GC 알고리즘 선택:**
    * **`-XX:+UseSerialGC`**: 단일 코어, 소규모 힙 (클라이언트 앱)
    * **`-XX:+UseParallelGC`**: 멀티 코어, 높은 처리량 목표 (Java 8 기본)
    * **`-XX:+UseConcMarkSweepGC`**: 낮은 응답 시간 목표 (Java 9+ Deprecated)
    * **`-XX:+UseG1GC`**: 멀티 코어, 대용량 힙, 처리량/응답 시간 균형 (Java 9+ 기본)
    * **`-XX:+UseZGC` / `-XX:+UseShenandoahGC`**: 극도로 낮은 GC 일시 중지 시간 목표 (최신 JVM)
    * **추천:** 대부분의 서버 애플리케이션은 **`G1 GC`**가 좋은 성능을 보입니다.

* **힙 메모리 크기 설정:**
    * **`-Xms<size>`**: JVM 초기 힙 크기 (예: `-Xms1g`)
    * **`-Xmx<size>`**: JVM 최대 힙 크기 (예: `-Xmx4g`)
    * **최적화 팁:** `–Xms`와 `–Xmx`를 **동일하게 설정**하여 힙 크기 변경에 따른 성능 저하를 방지하는 것이 일반적입니다. (예: `-Xms2g -Xmx2g`)

* **Young Generation 크기 조절:**
    * **`-Xmn<size>`**: Young Generation의 전체 크기를 명시적으로 지정. (예: `-Xmn512m`)
    * **`-XX:NewRatio=<ratio>`**: `Old Generation : Young Generation = N : 1` 비율 설정. (예: `-XX:NewRatio=2`는 Old:Young = 2:1)
    * **최적화 팁:** 애플리케이션의 객체 생명주기 패턴에 맞춰 조절합니다. 짧게 살아남는 객체가 많으면 Young Gen을 크게 하여 Minor GC 효율을 높일 수 있습니다.

### 3.2. GC 로그 분석 (핵심!)

GC 튜닝의 시작점이자 가장 중요한 부분입니다.

* **로그 활성화 옵션:**
    * **Java 8 이하:** `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:<file_path>`
    * **Java 9 이상:** `-Xlog:gc*:file=<file_path>` (세분화된 태그 및 레벨 설정 가능)
* **분석 도구:**
    * **GCViewer:** 데스크톱 기반의 오픈소스 분석 도구.
    * **GCeasy:** 웹 기반의 온라인 분석 도구.
    * **VisualVM:** JVM 모니터링 및 프로파일링 도구.

**GC 로그를 통한 튜닝 예시:**
1.  **문제 진단:** GC 로그 분석 결과, `Max GC Pause`가 3초 이상으로 너무 길고, `Full GC`가 잦게 발생하며, `Heap Usage` 그래프에서 Full GC 직전에 힙이 거의 가득 차는 것을 발견.
2.  **원인 추정:** Old Generation이 너무 작거나, Young Generation에서 Old Generation으로 객체가 너무 많이 승격되고 있거나, 심각한 메모리 누수가 의심됨.
3.  **튜닝 시도:**
    * 힙 크기(`-Xmx`)를 점진적으로 늘려 Full GC 빈도 감소 시도. (예: 4GB -> 6GB)
    * `NewRatio` 또는 `Xmn`으로 Young Generation 크기를 조절하여 Minor GC 효율을 높이고 Old Generation 승격률 조절 시도.
    * (만약 힙이 계속 증가한다면) 다음 섹션의 메모리 누수 방지 코드를 통해 코드 레벨에서 문제 해결 시도.
4.  **결과 확인:** 튜닝 후 GC 로그를 다시 분석하여 `Max GC Pause`와 `Full GC` 발생 빈도가 목표치 이하로 줄었는지 확인.

### 3.3. 코드 레벨 GC 최적화

불필요한 객체 생성을 줄이는 것은 GC 부담을 줄이는 가장 기본적인 방법입니다.

* **객체 생성 최소화 및 재사용:**
    * **문자열 연산:** `String` `+` 연산 대신 `StringBuilder` (멀티스레드 환경에서는 `StringBuffer`) 사용.
    * **정규 표현식:** `Pattern.compile()` 결과를 `static final` 필드로 저장하여 재사용.
    * **불필요한 객체 줄이기:** Stream API 사용 시 중간 객체 생성에 유의, 필요시 전통적인 `for` 루프 고려.

---

## 4. 가비지 컬렉션과 메모리 누수

GC는 '도달 불가능한(unreachable) 객체'를 회수합니다. 하지만 **유효한 참조 변수가 객체를 계속 가리키고 있으면 GC는 해당 객체를 '도달 가능하다'고 판단하여 회수하지 못하며, 이로 인해 메모리 누수(Memory Leak)가 발생**할 수 있습니다.

### 4.1. 메모리 누수 발생 원인

1.  **불필요한 객체 참조 유지 (Stale References):**
    * **오래된 캐시:** 만료되거나 더 이상 필요 없는 객체를 `Map`, `List` 같은 컬렉션에 계속 담아두는 경우. `clear()` 또는 `remove()`를 호출하지 않으면 참조가 계속 유지됩니다.
    * **스태틱 필드/컬렉션:** 애플리케이션 전역에서 유지되는 `static` 컬렉션에 객체를 추가하고 제거하지 않는 경우. `static` 필드가 참조하는 객체는 JVM 종료 시까지 GC되지 않습니다.
    * **이벤트 리스너/콜백:** 객체가 리스너로 등록된 후 해제되지 않아, 리스너가 객체를 계속 참조하는 경우.
    * **내부 클래스/익명 클래스:** 비정적(non-static) 내부 클래스가 외부 클래스 인스턴스를 묵시적으로 참조하여, 외부 클래스 인스턴스가 GC되지 않는 경우. `static` 내부 클래스를 사용하거나 필요한 데이터를 명시적으로 주입하여 해결.
    * **`ThreadLocal` 사용 후 정리 누락:** `ThreadLocal`에 값을 저장한 후 `remove()`를 호출하지 않으면, 스레드가 재활용될 때 메모리 누수 발생 가능.

2.  **외부 자원(Native Resources) 미해제:**
    * 데이터베이스 커넥션, 파일 핸들, 네트워크 소켓 등 자바 힙 메모리 외부에 존재하는 OS 자원은 GC가 직접 관리하지 않습니다.
    * 개발자가 반드시 `close()` 메서드를 호출하여 명시적으로 해제해야 합니다. (`try-with-resources` 구문 권장)

### 4.2. 메모리 누수 방지 방법 (코드 레벨)

* **참조 해제 및 `null` 할당:**
    * 더 이상 사용하지 않는 객체 참조를 `null`로 설정하거나, 컬렉션에서 `remove()` 또는 `clear()`를 호출하여 GC 대상이 되도록 합니다. (특히 장기 유지되는 컬렉션)
* **`try-with-resources` 구문 사용:**
    * `AutoCloseable` 인터페이스를 구현하는 모든 자원(예: `BufferedReader`, `Connection`, `PreparedStatement`, `ResultSet` 등)은 `try-with-resources` 구문을 사용하여 자동으로 닫히도록 합니다.
    * ```java
        try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
            // 파일 읽기 로직
        } catch (IOException e) {
            // 예외 처리
        } // reader.close()가 자동으로 호출됨
        ```
* **전문 캐시 라이브러리 사용:**
    * 직접 캐시를 구현하기보다 Caffeine, Ehcache 등과 같이 만료 정책, 크기 제한, 약한 참조(WeakReference) 등을 자동으로 관리해주는 검증된 캐시 라이브러리 사용.
* **`ThreadLocal`은 사용 후 반드시 `remove()`:**
    * 특히 WAS 환경의 스레드 풀에서 `ThreadLocal` 사용 후 `threadLocalVar.remove()`를 호출하여 누수를 방지합니다.
* **메모리 프로파일링 도구 활용:**
    * JProfiler, YourKit, VisualVM 같은 도구로 주기적으로 메모리 사용량을 모니터링하고, 힙 덤프(Heap Dump)를 분석하여 누수되는 객체와 그 객체를 참조하고 있는 경로를 찾아냅니다.

---

GC 최적화와 메모리 누수 방지는 JVM 설정 튜닝과 더불어, **개발자가 코드를 작성하는 습관과 객체 생명주기에 대한 이해**에서 시작됩니다. 불필요한 객체 생성을 줄이고, 사용이 끝난 자원과 참조를 명확히 해제하는 것이 견고하고 효율적인 자바 애플리케이션을 만드는 데 필수적입니다.
