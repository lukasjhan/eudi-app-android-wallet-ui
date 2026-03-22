# Android DCQL null 배열 와일드카드 + values 매칭 수정

## 목표

OpenID4VP DCQL에서 `null`을 이용한 배열 원소 매칭과 `values` 필터링이 작동하도록 수정.

```json
{"path": ["nationality", null], "values": ["NL"]}
```
→ nationality 배열의 모든 원소 중 "NL"이 있으면 매칭.

## 수정 대상: 3개 프로젝트, 13개 파일

---

## 1. eudi-lib-jvm-openid4vp-kt (1개 파일)

### `src/main/kotlin/.../dcql/DCQL.kt`
MsoMdoc 경로 검증 완화.

**변경 전:**
```kotlin
require(2 == claimsQuery.path.value.size)
require(claimsQuery.path.value.all { it is ClaimPathElement.Claim })
```

**변경 후:**
```kotlin
require(claimsQuery.path.value.size >= 2)
require(claimsQuery.path.value.take(2).all { it is ClaimPathElement.Claim })
```

**이유:** MsoMdoc도 `["ns", "nationality", null]` 같은 3개+ 경로를 허용해야 함. 처음 2개(namespace, elementIdentifier)만 Claim 타입 강제, 이후는 null/index 허용.

---

## 2. eudi-lib-android-wallet-core (4개 파일)

### `wallet-core/build.gradle.kts`
```diff
- implementation(libs.eudi.lib.jvm.siop.openid4vp.kt) {
+ api(libs.eudi.lib.jvm.siop.openid4vp.kt) {
```

**이유:** openid4vp의 `ClaimPath`/`ClaimPathElement` 타입을 앱에 노출하여, `List<String>` 대신 타입 안전한 경로를 사용.

### `SdJwtVcItem.kt`
```diff
- class SdJwtVcItem(val path: List<String>) : DocItem
+ class SdJwtVcItem(val path: ClaimPath) : DocItem
```

**이유:** `List<String>`은 `null`(AllArrayElements)과 정수 인덱스(ArrayElement)를 표현할 수 없음. `ClaimPath`는 세 가지 타입을 모두 보존.

### `dcql/DcqlRequestProcessor.kt`

#### SD-JWT VC: ClaimPath 직접 전달
```diff
- SdJwtVcItem(path = claim.path.value.map { it.toString() })
+ SdJwtVcItem(path = claim.path)
```

**이유:** `toString()` 변환이 타입 정보를 손실시킴. `AllArrayElements.toString()` = `"null"` → 문자열 `"null"`과 구분 불가.

#### MsoMdoc: 3개+ 경로 지원
```diff
- namespace = claim.path.value.first().toString(),
- elementIdentifier = claim.path.value.last().toString()
+ val namespace = (claim.path.value[0] as ClaimPathElement.Claim).name
+ val elementIdentifier = (claim.path.value[1] as ClaimPathElement.Claim).name
```

**이유:** `last()`는 경로가 3개 이상일 때 잘못된 값을 반환. 항상 인덱스 0, 1에서 추출.

#### values 매칭 로직 신규 구현

4개 함수 추가:

| 함수 | 역할 |
|---|---|
| `matchesClaimValues()` | claim 경로로 값을 탐색하고 `values` 배열과 비교 |
| `resolveValueAtPath()` | `Claim`(맵 키), `ArrayElement`(인덱스), `AllArrayElements`(전체 순회)로 값 네비게이션 |
| `resolveValueFromSdJwtClaims()` | SdJwtVcClaim 계층 구조를 따라 탐색 후 값 기반 네비게이션으로 전환 |
| `valueMatchesAny()` | JsonPrimitive와 실제 값(String, Int, Boolean 등) 비교 |

**매칭 흐름:**
```
DCQL: {path: ["ns", "nationality", null], values: ["LU", "FR"]}
  ↓ MsoMdoc claim 찾기: namespace="ns", elementIdentifier="nationality"
  ↓ claim.value = ["LU", "DE", "BE"]
  ↓ 남은 경로 [null] → AllArrayElements → 배열 전체 순회
  ↓ "LU" ∈ ["LU", "FR"] → 매칭 성공 ✅ → document 포함
```

매칭 실패 시 해당 document는 `RequestedDocuments`에서 제외됨.

### `internal/OpenId4VpUtils.kt`

VP 응답 생성 시 ClaimPathElement 타입 보존 변환.

```diff
  // 변경 전: 모든 원소를 Claim으로 잘못 변환
- val elements = item.path.map { ClaimPathElement.Claim(it) }

  // 변경 후: 타입별 정확한 변환
+ element.fold(
+     ifAllArrayElements = { SdJwtClaimPathElement.AllArrayElements },
+     ifArrayElement = { index -> SdJwtClaimPathElement.ArrayElement(index) },
+     ifClaim = { name -> SdJwtClaimPathElement.Claim(name) }
+ )
```

**이유:** openid4vp의 `ClaimPath`와 sd-jwt의 `ClaimPath`는 같은 구조의 다른 클래스. `fold()`로 타입별 매핑.

---

## 3. eudi-app-android-wallet-ui (8개 파일)

### `core-logic/.../ClaimPathDomain.kt` (핵심)

```diff
- val value: List<String>,
+ val value: List<ClaimPathElement>,
```

`isPrefixOf()` 수정:
```kotlin
// 트레일링 와일드카드 허용
val effectiveSize =
    this.value.indexOfLast { it !is ClaimPathElement.AllArrayElements } + 1
if (effectiveSize > other.value.size) return false

// openid4vp의 contains 연산자 사용 (AllArrayElements가 ArrayElement를 포함)
val compareSize = minOf(this.value.size, other.value.size)
return this.value.take(compareSize).zip(other.value.take(compareSize))
    .all { (a, b) -> a in b || b in a }
```

`toSdJwtVcPath()` → `toSdJwtVcClaimPath()`:
```diff
- fun toSdJwtVcPath(itemId: String): List<String>
+ fun toSdJwtVcClaimPath(itemId: String): ClaimPath
```

### `core-logic/.../DocItemExtensions.kt`

```diff
- is SdJwtVcItem -> this.path  // List<String>
+ is SdJwtVcItem -> ClaimPathDomain(
+     value = this.path.value,  // List<ClaimPathElement> 직접 사용
+     type = ClaimType.SdJwtVc
+ )
```

### `core-logic/.../DocumentClaimExtensions.kt`

```diff
- parentPath: List<String> = emptyList()
+ parentPath: List<ClaimPathElement> = emptyList()

- val currentPath: List<String> = parentPath + this.identifier
+ val currentPath = parentPath + ClaimPathElement.Claim(this.identifier)
```

### `common-feature/.../RequestTransformer.kt`

```diff
- path = ClaimPathDomain.toSdJwtVcPath(selectedItemId)
+ path = ClaimPathDomain.toSdJwtVcClaimPath(selectedItemId)
```

### `common-feature/.../DocumentHelper.kt`

```diff
- val key = path.value.first()
+ val key = path.value.first().toString()
```

### 테스트 파일 (3개)

- `TestClaimPath.kt`: `SdJwtVcItem(ClaimPath(...))` 패턴으로 전환, 와일드카드 테스트 4개 추가
- `TestData.kt`: `listOf("x")` → `listOf(ClaimPathElement.Claim("x"))` (15개소)
- `TestDocumentDetailsInteractor.kt`: 동일 패턴 (1개소)

---

## 타입 흐름 (수정 후)

```
Verifier DCQL JSON: {path: ["nationality", null], values: ["NL"]}
    ↓ openid4vp 파싱
ClaimPath([Claim("nationality"), AllArrayElements])
    ↓ wallet-core DcqlRequestProcessor
    ↓ values 매칭: document.nationality 배열에서 "NL" 확인
    ↓ SdJwtVcItem(path: ClaimPath)  ← 타입 보존
    ↓ Android 앱
ClaimPathDomain(value: List<ClaimPathElement>)
    ↓ isPrefixOf(): AllArrayElements가 와일드카드로 동작
    ↓ UI에 claim 표시
    ↓ 사용자 선택 → SdJwtVcItem(ClaimPath) 생성
    ↓ wallet-core OpenId4VpUtils
    ↓ fold()로 sd-jwt ClaimPath 변환 (타입 보존)
    ↓ VP 응답 생성
```

## 로컬 빌드 방법

### Gradle mavenLocal 구조

수정된 외부 라이브러리를 앱에서 사용하려면 **Maven Local** (`~/.m2/repository/`)에 발행합니다.
Gradle의 `mavenLocal()` 리포지토리가 `mavenCentral()`보다 위에 있으면 로컬 artifact가 우선 적용됩니다.

### 의존 체인 (발행 순서 중요)

```
openid4vp (0.12.4-SNAPSHOT) → publishToMavenLocal → ~/.m2/repository/
        ↓ (wallet-core가 의존)
wallet-core (0.99.0-SNAPSHOT) → publishToMavenLocal → ~/.m2/repository/
        ↓ (앱이 의존)
Android 앱 → gradle/libs.versions.toml에서 SNAPSHOT 버전 참조
```

### 실행 순서

```bash
# 1단계: openid4vp 발행 (DCQL.kt 수정 포함)
cd eudi-lib-jvm-openid4vp-kt
# gradle.properties의 version을 0.12.4-SNAPSHOT으로 변경
./gradlew publishToMavenLocal

# 2단계: wallet-core 발행 (SdJwtVcItem, DcqlRequestProcessor 수정 포함)
cd eudi-lib-android-wallet-core
# gradle.properties의 version을 0.99.0-SNAPSHOT으로 변경
# gradle/libs.versions.toml에서 openid4vp 버전도 0.12.4-SNAPSHOT으로 변경
./gradlew publishToMavenLocal

# 3단계: Android 앱 빌드
cd eudi-app-android-wallet-ui
./gradlew assembleDebug
```

### 앱 측 설정 변경 내역

#### `settings.gradle.kts` (이미 설정되어 있었음)
```kotlin
dependencyResolutionManagement {
    repositories {
        mavenLocal()      // ← 로컬 우선
        mavenCentral()
        // ...
    }
}
```

#### `gradle/libs.versions.toml`
```diff
- eudiWalletCore = "0.25.0"
+ eudiWalletCore = "0.99.0-SNAPSHOT"
```

#### `core-logic/build.gradle.kts`
```kotlin
// openid4vp 타입(ClaimPath, ClaimPathElement)을 앱 코드에서 직접 사용하기 위해 추가
api("eu.europa.ec.eudi:eudi-lib-jvm-openid4vp-kt:0.12.4-SNAPSHOT") {
    exclude(group = "org.bouncycastle")
}
```

### 원복 방법

`gradle/libs.versions.toml`의 버전을 원래대로 돌리고, `core-logic/build.gradle.kts`의 openid4vp api 줄을 제거하면 원격 Maven artifact로 복원됨.

---

## 트러블슈팅: 로컬 SNAPSHOT 빌드가 반영되지 않는 문제

### 증상

`publishToMavenLocal`로 수정된 라이브러리를 발행했는데, 앱 APK에 여전히 **옛날 코드**가 들어감.
(예: 에러 메시지가 수정 전 "exactly two elements"로 나옴)

### 원인 1: `mavenLocal()` 우선순위

`settings.gradle.kts`의 `repositories` 블록에서 `mavenLocal()`이 다른 저장소보다 **아래에** 있으면, 원격 Maven 저장소(특히 Sonatype Snapshots)에 같은 SNAPSHOT 버전이 있을 경우 **원격 버전이 우선** 적용됨.

```kotlin
// ❌ 문제: mavenLocal이 맨 아래
repositories {
    google()
    mavenCentral()
    maven { url = uri("https://central.sonatype.com/repository/maven-snapshots/") }
    mavenLocal()  // 원격에서 먼저 받아버림
}

// ✅ 해결: mavenLocal을 맨 위로
repositories {
    mavenLocal()  // 로컬 우선
    google()
    mavenCentral()
    maven { url = uri("https://central.sonatype.com/repository/maven-snapshots/") }
}
```

### 원인 2: Gradle 글로벌 캐시

`~/.gradle/caches/modules-2/files-2.1/` 에 이전에 다운로드된 같은 SNAPSHOT 버전의 JAR가 캐싱됨.
`mavenLocal()` 우선순위를 올려도 이미 캐시된 JAR을 사용하는 경우가 있음.

```bash
# 특정 라이브러리의 글로벌 캐시 삭제
rm -rf ~/.gradle/caches/modules-2/files-2.1/eu.europa.ec.eudi/eudi-lib-jvm-openid4vp-kt/

# 또는 find로 일괄 삭제
find ~/.gradle/caches -path "*openid4vp*" -exec rm -rf {} + 2>/dev/null
```

### 원인 3: Gradle transform 캐시

글로벌 캐시만 삭제하면 DEX transform 캐시(`~/.gradle/caches/X.X.X/transforms/`)가 깨져서 빌드 실패할 수 있음.

```bash
# transform 캐시도 함께 삭제
rm -rf ~/.gradle/caches/9.3.1/transforms/
```

### 원인 4: Configuration cache

프로젝트 `.gradle/` 디렉토리의 configuration cache가 이전 의존성 해결 결과를 기억함.

```bash
# 프로젝트 캐시 + build-logic 캐시 삭제
rm -rf .gradle build-logic/.gradle
```

### 확실한 해결 방법 (권장)

위 원인들이 복합적으로 발생할 수 있으므로, 한번에 해결하는 방법:

```bash
# 1. 해당 라이브러리의 글로벌 캐시 삭제
find ~/.gradle/caches -path "*openid4vp*" -exec rm -rf {} + 2>/dev/null

# 2. 프로젝트 캐시 삭제
rm -rf .gradle build-logic/.gradle

# 3. clean 빌드 + 의존성 새로고침
./gradlew clean assembleDevDebug --no-configuration-cache --refresh-dependencies
```

### APK에 올바른 코드가 포함되었는지 검증

```bash
# APK 내 DEX에서 특정 문자열 검색
unzip -p app/build/outputs/apk/dev/debug/app-dev-debug.apk "classes*.dex" \
  | strings | grep "two elements"
```
