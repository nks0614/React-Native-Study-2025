# Phase 2: React Hooks 심화와 최적화

## 📌 학습 목표

Phase 1에서 배운 기본 Hooks(useState, useEffect)를 넘어, **필수적인 고급 Hooks와 최적화 기법**을 마스터하기

## 🗂️ 학습 구조

### 1. [useEffect 심화](./01-useEffect-심화.md)

- useEffect 실행 타이밍 완벽 이해
- Cleanup 함수의 실행 순서
- 의존성 배열 깊이 있게 이해하기
- 무한 루프 디버깅과 해결
- useLayoutEffect vs useEffect
- Race Condition 해결

### 2. [성능 최적화 Hooks](./02-성능-최적화-Hooks.md)

- useRef: 렌더링 없이 값 유지하기
- useMemo: 비용이 큰 계산 메모이제이션
- useCallback: 함수 메모이제이션
- React.memo와 함께 사용하기
- 언제 최적화해야 하는가?
- 실제 성능 측정 방법

### 3. [Custom Hooks 마스터](./03-Custom-Hooks.md)

- Custom Hooks 설계 원칙
- 실무 필수 Custom Hooks 10가지
- useFetch, useDebounce, useLocalStorage
- useIntersectionObserver, useMediaQuery
- Hooks 조합 패턴
- 테스트 가능한 Hooks 작성

### 4. [Hooks 내부 구조 Deep Dive](./04-Hooks-내부구조-심화.md)

- React 소스코드 깊이 분석
- Fiber 아키텍처와 Hooks의 관계
- useEffect의 비동기 실행 메커니즘
- useMemo/useCallback의 내부 캐싱
- Hook 호출 순서가 중요한 이유
- 직접 구현해보는 고급 Hooks

### 5. [실습 프로젝트](./05-실습-프로젝트.md)

- 실시간 검색 기능 (Debounce + Custom Hook)
- 무한 스크롤 (Intersection Observer)
- 다크모드 토글 (Context + Custom Hook)
- 성능 최적화된 대용량 리스트
- API 캐싱 시스템

### 6. [학습 점검 질문](./06-학습-점검-질문.md)

- Phase 2 핵심 개념 복습
- 실무 시나리오 기반 문제
- 성능 최적화 판단 연습
- 코드 리뷰 시뮬레이션

## 🎯 Phase 1과의 차이점

| 구분         | Phase 1                  | Phase 2                     |
| ------------ | ------------------------ | --------------------------- |
| **난이도**   | 기초                     | 중급~고급                   |
| **Hooks**    | useState, useEffect 기본 | 최적화 Hooks + Custom Hooks |
| **목표**     | "사용할 수 있다"         | "제대로 사용할 수 있다"     |
| **초점**     | 개념 이해                | 성능 최적화 + 실무 패턴     |
| **소스코드** | 기본 구조                | 내부 메커니즘 깊이 분석     |
| **실습**     | 간단한 컴포넌트          | 실무급 기능 구현            |

## 💡 왜 Phase 2가 중요한가?

### 문제 상황

```jsx
// ❌ Phase 1 수준 - 동작은 하지만 비효율적
function UserList() {
  const [users, setUsers] = useState([]);
  const [filter, setFilter] = useState("");

  useEffect(() => {
    fetchUsers().then(setUsers);
  }, []); // 문제 없어 보임

  // 매 렌더링마다 필터링 재실행
  const filteredUsers = users.filter((u) => u.name.includes(filter));

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {filteredUsers.map((user) => (
        <UserItem
          key={user.id}
          user={user}
          onEdit={() => handleEdit(user)} // 매번 새 함수 생성
        />
      ))}
    </>
  );
}
```

### Phase 2 해결책

```jsx
// ✅ Phase 2 수준 - 최적화된 코드
function UserList() {
  const [users, setUsers] = useState([]);
  const [filter, setFilter] = useState("");

  // Custom Hook으로 로직 분리
  const { data: users, loading, error } = useFetch("/api/users");

  // 필터가 변경될 때만 재계산
  const filteredUsers = useMemo(() => users.filter((u) => u.name.includes(filter)), [users, filter]);

  // 함수 메모이제이션
  const handleEdit = useCallback((user) => {
    // 편집 로직
  }, []);

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {filteredUsers.map((user) => (
        <MemoizedUserItem key={user.id} user={user} onEdit={handleEdit} />
      ))}
    </>
  );
}

// React.memo로 불필요한 리렌더링 방지
const MemoizedUserItem = React.memo(UserItem);
```

## 📚 Native 개발자를 위한 비유표 확장

| React 개념      | Android                       | iOS               | 설명                         |
| --------------- | ----------------------------- | ----------------- | ---------------------------- |
| useRef          | remember (Compose)            | @State (SwiftUI)  | 렌더링 없이 값 유지          |
| useMemo         | derivedStateOf (Compose)      | computed property | 계산된 값 캐싱               |
| useCallback     | remember { lambda } (Compose) | -                 | 함수 메모이제이션            |
| useLayoutEffect | onLayout callback             | layoutSubviews    | 레이아웃 측정 후 동기 실행   |
| Custom Hook     | ViewModel Helper / Extension  | View Modifier     | 재사용 가능한 로직           |
| React.memo      | @Stable (Compose)             | Equatable         | Props 비교로 리렌더링 최적화 |

## ⚡ 선수 지식

Phase 1을 완료하고:

- [x] useState, useEffect를 자유롭게 사용할 수 있음
- [x] Hook 규칙을 이해함
- [x] 의존성 배열의 기본 개념을 알고 있음
- [x] 렌더링 흐름을 이해함

## 🎓 학습 방법

### 1단계: 개념 이해 (30%)

- 각 Hook이 **왜** 필요한지 이해
- 성능 최적화의 필요성 파악

### 2단계: 코드 분석 (30%)

- React 소스코드에서 실제 구현 확인
- 내부 메커니즘 추적

### 3단계: 실습 (40%)

- 실무 수준의 프로젝트 구현
- 성능 측정 도구로 최적화 효과 확인
- Custom Hooks 직접 작성

## 🔥 학습 후 기대 효과

Phase 2를 완료하면:

- ✅ 실무에서 성능 문제를 스스로 진단하고 해결할 수 있습니다
- ✅ 언제 최적화가 필요한지 판단할 수 있습니다
- ✅ 재사용 가능한 Custom Hooks를 설계할 수 있습니다
- ✅ 코드 리뷰에서 성능 문제를 발견할 수 있습니다
- ✅ React의 내부 동작을 깊이 있게 이해하게 됩니다

## 🚀 시작하기

Phase 2는 Phase 1보다 **실무 중심**입니다. 개념을 이해하는 것도 중요하지만, **직접 구현하고 최적화 효과를 측정**하는 것이 핵심입니다.

**다음**: [01-useEffect-심화.md](./01-useEffect-심화.md)로 시작하세요!
