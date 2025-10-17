# Hooks 내부 구조 Deep Dive

## 📌 학습 목표

- React 소스코드에서 고급 Hooks 구현 분석
- Fiber 아키텍처와 Hooks의 관계 이해
- useEffect의 비동기 실행 메커니즘 파악
- useMemo/useCallback의 내부 캐싱 원리
- 직접 간단한 Hooks 구현해보기

---

## 1. useEffect 내부 구현

### 소스코드 위치

**위치**: `react/packages/react-reconciler/src/ReactFiberHooks.js` [링크](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)

### mountEffect 구현

```javascript
function mountEffect(create: () => (() => void) | void, deps: Array<mixed> | void | null): void {
  return mountEffectImpl(PassiveEffect | PassiveStaticEffect, HookPassive, create, deps);
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;

  // Fiber에 플래그 설정
  currentlyRenderingFiber.flags |= fiberFlags;

  // Effect를 큐에 추가
  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, undefined, nextDeps);
}
```

### 핵심 발견

1. **Effect는 큐에 저장됨**: `pushEffect`로 Effect 리스트에 추가
2. **Fiber 플래그**: Effect 실행이 필요함을 React에 알림
3. **의존성 저장**: `nextDeps`로 의존성 배열 저장

### pushEffect 구현

```javascript
function pushEffect(tag, create, destroy, deps) {
  const effect = {
    tag,
    create, // Effect 함수
    destroy, // Cleanup 함수
    deps, // 의존성 배열
    next: null,
  };

  // Effect를 Fiber의 Effect 리스트에 추가
  let componentUpdateQueue = currentlyRenderingFiber.updateQueue;
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    const firstEffect = lastEffect.next;
    lastEffect.next = effect;
    effect.next = firstEffect;
    componentUpdateQueue.lastEffect = effect;
  }

  return effect;
}
```

**구조**: 원형 연결 리스트로 Effect들을 관리!

### 의존성 비교 로직

```javascript
function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) {
    return false; // 첫 렌더링
  }

  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    // Object.is 사용 (===와 거의 동일, +0과 -0, NaN 처리 차이)
    if (is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

**중요**: 얕은 비교(Shallow Comparison)이므로 객체/배열은 참조만 비교!

### 예제: 의존성 비교의 함정

```jsx
// ❌ 매번 실행됨 (새로운 객체 생성)
function Component() {
  const filters = { name: "John" }; // 매 렌더링마다 새 객체

  useEffect(() => {
    fetchUsers(filters);
  }, [filters]); // filters는 매번 다른 참조!
}

// ✅ 개별 값으로 분리
function Component() {
  const filterName = "John";

  useEffect(() => {
    fetchUsers({ name: filterName });
  }, [filterName]); // 원시값은 값으로 비교
}
```

### updateEffect 구현

```javascript
function updateEffect(create: () => (() => void) | void, deps: Array<mixed> | void | null): void {
  return updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;

    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // 의존성이 같으면 Effect 스킵
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  // 의존성이 변경되었으면 실행 플래그 설정
  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, destroy, nextDeps);
}
```

**핵심**: 의존성이 같으면 `HookHasEffect` 플래그를 설정하지 않아 Effect 실행을 스킵!

---

## 2. useRef 내부 구현

### 소스코드

```javascript
function mountRef<T>(initialValue: T): { current: T } {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}

function updateRef<T>(initialValue: T): { current: T } {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState; // 같은 객체 반환!
}
```

### 핵심 발견

- `ref`는 단순히 `{ current: value }` 객체
- 매 렌더링마다 **같은 객체**를 반환
- `current` 변경해도 리렌더링 안 되는 이유: React가 추적 안 함

### 직접 구현해보기

```javascript
function MyUseRef(initialValue) {
  // Hook 데이터 가져오기 (첫 렌더링 시 생성)
  const hook = getCurrentHook();

  if (hook.value === undefined) {
    // 첫 렌더링: 새 ref 생성
    hook.value = { current: initialValue };
  }

  // 항상 같은 객체 반환
  return hook.value;
}

// 사용
const countRef = MyUseRef(0);
countRef.current += 1; // 변경해도 리렌더링 없음
```

---

## 3. useMemo 내부 구현

### 소스코드

```javascript
function mountMemo<T>(nextCreate: () => T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;

  const nextValue = nextCreate(); // 계산 실행
  hook.memoizedState = [nextValue, nextDeps]; // [값, 의존성] 저장
  return nextValue;
}

function updateMemo<T>(nextCreate: () => T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null && nextDeps !== null) {
    const prevDeps = prevState[1];
    // 의존성이 같으면 캐시된 값 반환
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0]; // 재계산 X
    }
  }

  const nextValue = nextCreate(); // 재계산
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

### 핵심 발견

- `memoizedState`에 `[계산된 값, 의존성 배열]` 저장
- 의존성이 변경되지 않으면 이전 값 반환
- `nextCreate()` 호출 안 함 = 계산 스킵

### 직접 구현해보기

```javascript
function MyUseMemo(createFn, deps) {
  const hook = getCurrentHook();

  if (hook.value === undefined) {
    // 첫 렌더링: 계산 실행
    const value = createFn();
    hook.value = [value, deps];
    return value;
  }

  const [prevValue, prevDeps] = hook.value;

  // 의존성 비교
  if (depsAreEqual(deps, prevDeps)) {
    return prevValue; // 캐시된 값 반환
  }

  // 의존성 변경: 재계산
  const value = createFn();
  hook.value = [value, deps];
  return value;
}

function depsAreEqual(nextDeps, prevDeps) {
  if (!prevDeps) return false;

  for (let i = 0; i < nextDeps.length; i++) {
    if (!Object.is(nextDeps[i], prevDeps[i])) {
      return false;
    }
  }
  return true;
}
```

### 성능 비교

```jsx
function ExpensiveComponent({ items }) {
  // ❌ 매 렌더링마다 정렬 (예: 100ms)
  const sortedItems = items.sort((a, b) => b.price - a.price);

  // ✅ items 변경 시에만 정렬
  const sortedItems = useMemo(() => items.sort((a, b) => b.price - a.price), [items]);
}
```

---

## 4. useCallback 내부 구현

### 소스코드

```javascript
function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null && nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0]; // 이전 함수 반환
    }
  }

  hook.memoizedState = [callback, nextDeps];
  return callback; // 새 함수 반환
}
```

### 핵심 발견

- `useMemo`와 거의 동일한 구조!
- 차이점: 계산 함수를 실행하지 않고 **함수 자체**를 저장

### useMemo와의 관계

```javascript
// useCallback은 useMemo의 syntactic sugar
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);

// 위와 동일
const memoizedCallback = useMemo(() => {
  return () => doSomething(a, b);
}, [a, b]);
```

---

## 5. Fiber 아키텍처와 Hooks

### Fiber란?

React 16부터 도입된 새로운 Reconciliation 엔진의 핵심 데이터 구조

```javascript
type Fiber = {
  tag: WorkTag, // 컴포넌트 타입 (함수/클래스/...)
  type: any, // 컴포넌트 함수/클래스
  stateNode: any, // DOM 노드 또는 인스턴스

  return: Fiber | null, // 부모 Fiber
  child: Fiber | null, // 첫 자식 Fiber
  sibling: Fiber | null, // 형제 Fiber

  memoizedState: any, // Hooks 연결 리스트!
  memoizedProps: any, // 이전 props
  pendingProps: any, // 새로운 props

  updateQueue: any, // Effect 큐
  flags: Flags, // 업데이트 플래그

  // ...
};
```

### Hooks와 Fiber의 관계

```
Fiber
  └── memoizedState (첫 번째 Hook)
        ├── memoizedState: Hook의 실제 값
        ├── queue: setState 큐
        └── next → 두 번째 Hook
                ├── memoizedState
                ├── queue
                └── next → 세 번째 Hook
                        └── ...
```

**Hooks는 Fiber의 `memoizedState`에 연결 리스트로 저장!**

### Hook 데이터 구조

```javascript
type Hook = {
  memoizedState: any, // 현재 값
  baseState: any, // 기본 상태
  baseQueue: Update<any, any> | null,
  queue: UpdateQueue<any, any> | null, // setState 큐
  next: Hook | null, // 다음 Hook
};
```

### 예제: Hook 호출 순서

```jsx
function Component() {
  const [count, setCount] = useState(0);      // Hook 1
  const [name, setName] = useState('John');   // Hook 2
  useEffect(() => { ... }, [count]);          // Hook 3

  return ...;
}
```

**내부 구조**:

```
Fiber.memoizedState
  ↓
Hook 1 (useState) → Hook 2 (useState) → Hook 3 (useEffect) → null
  ├─ memoizedState: 0
  └─ next ──────────→ ├─ memoizedState: 'John'
                      └─ next ──────────→ ├─ memoizedState: Effect 객체
                                          └─ next: null
```

### Hook 호출 순서가 중요한 이유

```jsx
// ❌ 조건문에서 Hook 호출
function BadComponent({ condition }) {
  const [count, setCount] = useState(0);  // Hook 1

  if (condition) {
    const [name, setName] = useState(''); // Hook 2 (조건부!)
  }

  useEffect(() => { ... }, []);           // Hook 2 또는 3?
}
```

**문제**:

- 첫 렌더링: Hook 1 → Hook 2 → Hook 3
- 두 번째 렌더링 (condition=false): Hook 1 → Hook 3 (Hook 2 스킵)
- React는 Hook 3가 이전의 Hook 2 위치라고 착각! 💥

---

## 6. Effect 실행 메커니즘

### Effect 실행 타임라인

```
1. 렌더링 페이즈 (Render Phase)
   └─ 컴포넌트 함수 실행
      └─ useEffect 호출 → Effect를 큐에 추가 (실행 X)

2. 커밋 페이즈 (Commit Phase)
   ├─ Before Mutation
   ├─ Mutation (DOM 업데이트)
   ├─ Layout (useLayoutEffect 실행) ← 동기
   └─ After Commit

3. 브라우저 Paint
   └─ 화면 업데이트 (사용자가 볼 수 있음)

4. Passive Effect (useEffect 실행) ← 비동기
   ├─ 이전 Effect의 Cleanup 실행
   └─ 새로운 Effect 실행
```

### Scheduler를 통한 비동기 실행

```javascript
// React 내부 (단순화)
function commitRoot(root) {
  // 1. DOM 업데이트
  commitMutationEffects(root);

  // 2. useLayoutEffect 실행 (동기)
  commitLayoutEffects(root);

  // 3. 브라우저 paint 허용

  // 4. useEffect를 스케줄링 (비동기)
  scheduleCallback(NormalPriority, () => {
    flushPassiveEffects(root);
  });
}
```

**핵심**: useEffect는 `scheduleCallback`으로 다음 이벤트 루프에서 실행!

---

## 7. 직접 구현해보는 간단한 Hooks

### 미니 React 구현

```javascript
let currentComponent = null;
let hookIndex = 0;

class Component {
  constructor() {
    this.hooks = [];
  }

  render(fn) {
    currentComponent = this;
    hookIndex = 0;
    return fn();
  }
}

// useState 구현
function useState(initialValue) {
  const component = currentComponent;
  const index = hookIndex;
  hookIndex++;

  // 첫 렌더링이면 초기화
  if (component.hooks[index] === undefined) {
    component.hooks[index] = initialValue;
  }

  const setState = (newValue) => {
    component.hooks[index] = newValue;
    rerender(); // 리렌더링 트리거
  };

  return [component.hooks[index], setState];
}

// useEffect 구현 (단순화)
function useEffect(callback, deps) {
  const component = currentComponent;
  const index = hookIndex;
  hookIndex++;

  const prevDeps = component.hooks[index]?.deps;

  let hasChanged = true;
  if (prevDeps) {
    hasChanged = deps.some((dep, i) => !Object.is(dep, prevDeps[i]));
  }

  if (hasChanged) {
    // Cleanup 실행
    if (component.hooks[index]?.cleanup) {
      component.hooks[index].cleanup();
    }

    // Effect 실행 (비동기)
    setTimeout(() => {
      const cleanup = callback();
      component.hooks[index] = { deps, cleanup };
    }, 0);
  }
}

// 사용 예제
const app = new Component();

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("Count:", count);
    return () => console.log("Cleanup");
  }, [count]);

  return {
    count,
    increment: () => setCount(count + 1),
  };
}

// 렌더링
const result = app.render(Counter);
console.log(result.count); // 0
result.increment();
```

---

## 8. Android Lifecycle과 Hook 매핑

| Android Lifecycle          | React Hook                                    | 설명                    |
| -------------------------- | --------------------------------------------- | ----------------------- |
| `onCreate()`               | `useEffect(() => {}, [])`                     | 마운트 시 한 번 실행    |
| `onResume()`               | `useEffect(() => {}, [])` + Focus 감지        | 화면 재개 시 실행       |
| `onPause()`                | useEffect cleanup                             | 화면 일시정지 시 실행   |
| `onDestroy()`              | `useEffect(() => { return () => {...} }, [])` | 언마운트 시 실행        |
| `LiveData.observe()`       | `useEffect(() => {}, [dependency])`           | 값 변경 감지            |
| `remember` (Compose)       | `useRef()`                                    | 값 유지 (리렌더링 없음) |
| `derivedStateOf` (Compose) | `useMemo()`                                   | 계산된 값 캐싱          |

---

## 9. 정리

### 핵심 개념 요약

1. **Hooks는 Fiber의 연결 리스트**

   - 호출 순서로 식별
   - 조건부 호출 금지

2. **useEffect는 비동기 실행**

   - DOM 업데이트 후
   - 브라우저 paint 후
   - Scheduler로 스케줄링

3. **의존성 배열은 얕은 비교**

   - `Object.is` 사용
   - 객체/배열은 참조 비교

4. **useMemo/useCallback은 캐싱**
   - `[값, 의존성]` 형태로 저장
   - 의존성 변경 시에만 재계산/재생성

### 다음 단계

이제 실전 프로젝트로 배운 내용을 적용해봅시다!

**다음**: [05-실습-프로젝트.md](./05-실습-프로젝트.md)
