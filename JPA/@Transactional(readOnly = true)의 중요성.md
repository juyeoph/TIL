# JPA `@Transactional(readOnly = true)`의 중요성

오늘은 Spring Data JPA에서 트랜잭션 관리 시 사용되는 `@Transactional` 어노테션의 `readOnly = true` 속성이 어떤 의미를 가지며, 이를 명시하는 것이 왜 중요한지에 대해 정리합니다.

## 1. `@Transactional` 어노테이션의 기본 동작

Spring의 `@Transactional` 어노테이션은 메서드 또는 클래스에 적용되어 트랜잭션의 시작과 끝을 자동으로 관리해줍니다. 이는 JDBC의 `Connection.setAutoCommit(false)`, `commit()`, `rollback()`과 같은 번거로운 코드를 개발자가 직접 작성할 필요 없이, 선언적으로 트랜잭션을 관리할 수 있게 해줍니다.

**기본 동작:**
* `@Transactional`만 붙인 경우, 해당 트랜잭션은 **읽기(Read) 및 쓰기(Write) 작업 모두를 허용**하는 트랜잭션으로 간주됩니다.
* 이 트랜잭션 내에서는 JPA의 **변경 감지(Dirty Checking)** 기능이 활성화되어, 영속 상태의 엔티티 객체에 변경이 발생하면 트랜잭션 커밋 시점에 자동으로 `UPDATE` 쿼리를 데이터베이스에 전송합니다.

## 2. `@Transactional(readOnly = true)`란?

`readOnly = true` 속성은 해당 트랜잭션이 **데이터를 조회하는 목적만을 가질 뿐, 데이터를 변경하지 않을 것임**을 JPA와 데이터베이스에 명시적으로 알려주는 설정입니다.

### `@Transactional(readOnly = true)`와 일반 `@Transactional`의 차이점

| 특징             | `@Transactional` (기본값: readOnly=false)                                   | `@Transactional(readOnly = true)`                                        |
| :--------------- | :-------------------------------------------------------------------------- | :------------------------------------------------------------------------- |
| **목적** | 데이터 **조회 및 변경 (Read/Write)** | 데이터 **조회 (Read-only)** |
| **JPA 변경 감지** | **활성화됨.** 트랜잭션 커밋 시 엔티티 변경 감지 후 `UPDATE` 쿼리 자동 실행. | **비활성화됨.** 엔티티 변경을 감지하지 않으므로 불필요한 `UPDATE` 쿼리 발생 안함. |
| **스냅샷 생성** | 엔티티의 초기 상태(스냅샷) 생성 및 관리                                     | 스냅샷을 생성할 필요가 없으므로 **메모리 절약**.                          |
| **데이터베이스 잠금** | 쓰기 작업 시 강한 잠금(Lock)을 유발할 수 있어 동시성 저하 가능             | 읽기 전용으로 인식되어 **약한 잠금**을 사용하거나 아예 잠금을 걸지 않아 동시성 향상 |
| **DB 최적화 힌트** | -                                                                           | 일부 DB(MySQL, PostgreSQL 등)는 읽기 전용임을 인지하고 내부적 최적화 수행. |
| **데이터 무결성** | 실수로 읽기 전용 메서드에서 엔티티 변경 시 DB에 반영될 수 있음              | **실수로 엔티티 변경 시 DB에 반영되지 않아** 데이터 손상 방지 (안전망).   |
| **코드 가독성** | -                                                                           | 해당 메서드가 조회 전용임을 명확히 나타내어 **가독성 향상**.             |

### 3. `@Transactional(readOnly = true)`를 사용해야 하는 이유 (핵심 이점)

1.  **성능 최적화:**
    * **변경 감지 오버헤드 제거:** JPA는 읽기 전용 트랜잭션에서 변경 감지를 위한 스냅샷 비교 작업을 건너뜁니다. 이는 대량의 데이터를 조회할 때 불필요한 CPU 및 메모리 사용을 줄여줍니다.
    * **불필요한 `UPDATE` 쿼리 방지:** 트랜잭션 커밋 시점에 변경 감지로 인한 `UPDATE` 쿼리가 발생하지 않으므로, 네트워크 트래픽 및 DB 부하를 줄일 수 있습니다.
    * **데이터베이스 최적화 활용:** 데이터베이스는 `readOnly` 트랜잭션을 인식하여 더 효율적인 방식으로 쿼리를 처리하거나, 잠금 경합을 줄여 동시성을 높일 수 있습니다.

2.  **데이터 무결성 보호:**
    * `readOnly = true`로 설정된 메서드 내에서 개발자가 실수로 엔티티의 값을 변경하더라도, 해당 변경은 데이터베이스에 반영되지 않습니다. 이는 개발자의 실수로부터 데이터베이스의 데이터를 보호하는 **안전 장치** 역할을 합니다.

3.  **코드의 의도 명확화:**
    * 메서드의 목적이 데이터를 조회하는 것임을 명확히 드러내어 코드의 가독성을 높입니다. 이는 팀원 간의 협업 시 해당 메서드의 역할을 빠르게 파악하는 데 도움을 줍니다.

### 4. `@Transactional(readOnly = true)` 적용 예시

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.example.model.Member;
import com.example.repository.MemberRepository;

@Service
public class MemberService {

    private final MemberRepository memberRepository;

    public MemberService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }

    // 데이터 조회 전용 메서드: 읽기 전용 트랜잭션 설정
    @Transactional(readOnly = true)
    public Member getMemberById(Long id) {
        return memberRepository.findById(id)
                               .orElseThrow(() -> new RuntimeException("Member not found"));
    }

    // 데이터 변경이 필요한 메서드: 기본 트랜잭션 (readOnly=false)
    @Transactional // 또는 @Transactional(readOnly = false) 명시 가능
    public Member updateMemberName(Long id, String newName) {
        Member member = memberRepository.findById(id)
                                        .orElseThrow(() -> new RuntimeException("Member not found"));
        member.setName(newName); // 변경 감지 대상
        // 트랜잭션 커밋 시 DB에 UPDATE 쿼리 자동 반영
        return member;
    }

    // 클래스 레벨에 @Transactional을 적용하고, 필요한 경우 메서드 레벨에서 오버라이드
    @Transactional // 클래스 레벨에 적용된 트랜잭션
    public class ProductService {

        // ... 필드 및 생성자

        // 이 메서드는 클래스 레벨 @Transactional의 기본 속성을 따릅니다 (쓰기 트랜잭션).
        public void createProduct(String name, int price) {
            // save 로직
        }

        // 이 메서드는 클래스 레벨 @Transactional을 오버라이드하여 읽기 전용으로 설정합니다.
        @Transactional(readOnly = true)
        public Product getProductDetails(Long id) {
            // find 로직
            return null;
        }
    }
}
```

### 결론

읽기 전용 트랜잭션에 `@Transactional(readOnly = true)`를 명시하는 것은 **단순한 선택이 아니라, JPA 기반 애플리케이션의 성능, 효율성, 그리고 데이터 안정성을 높이는 데 필수적인 모범 사례**입니다. 조회 작업이 많은 애플리케이션이라면 이 설정을 적극적으로 활용하여 최적화를 이루어야 합니다.

---
