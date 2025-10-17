# Hooks 내부 구조 Deep Dive

> 이 문서는 React 소스코드를 분석하여 Hooks가 실제로 어떻게 동작하는지 깊이 파고듭니다.

## 1. useState는 어디에 있나?

### 파일 위치

```
react/
├── packages/
│   ├── react/
│   │   └── src/
│   │       └── ReactHooks.js          ← useState export
│   └── react-reconciler/
│       └── src/
│           └── ReactFiberHooks.js     ← 실제 구현
```

### ReactHooks.js - Public API

```javascript
// packages/react/src/ReactHooks.js

export function useState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

**분석**:

- `useState`는 단순한 **wrapper 함수**!
- 실제 구현은 `dispatcher`가 가지고 있음
- `dispatcher`는 렌더링 상태에 따라 다름

### resolveDispatcher()

```javascript
function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;

  if (dispatcher === null) {
    throw new Error("Invalid hook call. Hooks can only be called inside of the body of a function component.");
  }

  return dispatcher;
}
```

**중요**:

- React 렌더링 외부에서는 `dispatcher`가 `null`
- 이것이 "Hooks can only be called..." 에러의 원인!

### Dispatcher의 종류

```javascript
// react-reconciler/src/ReactFiberHooks.js

// 1. 마운트 시 (첫 렌더링)
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  // ...
};

// 2. 업데이트 시 (리렌더링)
const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  // ...
};

// 3. 리렌더링 중 (useEffect에서 setState 호출)
const HooksDispatcherOnRerender = {
  useState: rerenderState,
  useEffect: updateEffect,
  // ...
};
```

**핵심**:

- 첫 렌더링과 리렌더링이 **다른 함수**를 사용!
- `mountState`는 Hook을 **생성**
- `updateState`는 Hook을 **읽음**

---

## 2. Hook의 자료구조

### Hook 객체

```javascript
type Hook = {
  memoizedState: any, // 현재 상태 값
  baseState: any, // 기준 상태
  baseQueue: Update | null, // 대기 중인 업데이트 큐
  queue: UpdateQueue | null, // 업데이트 큐
  next: Hook | null, // 다음 Hook (연결 리스트!)
};
```

### 연결 리스트 (Linked List)

```javascript
// 컴포넌트에서
function Component() {
  const [name, setName] = useState('Alice');  // Hook 1
  const [age, setAge] = useState(25);         // Hook 2
  const [city, setCity] = useState('Seoul');  // Hook 3
}

// 내부 구조
{
  memoizedState: 'Alice',
  next: {
    memoizedState: 25,
    next: {
      memoizedState: 'Seoul',
      next: null
    }
  }
}
```

**중요**:

- Hook은 **배열이 아니라 연결 리스트**!
- 순서대로 저장됨
- 따라서 **호출 순서가 바뀌면 안됨**!

### Fiber 노드

컴포넌트는 **Fiber 노드**로 표현됩니다.

```javascript
type Fiber = {
  // 컴포넌트 타입
  type: any,

  // 현재 Props
  memoizedProps: any,

  // 현재 State (Hook 연결 리스트의 시작점!)
  memoizedState: Hook | null,

  // 업데이트 큐
  updateQueue: UpdateQueue | null,

  // 자식, 형제, 부모 노드
  child: Fiber | null,
  sibling: Fiber | null,
  return: Fiber | null,

  // ...
};
```

**구조**:

```
Fiber (Component)
  └─ memoizedState → Hook 1 → Hook 2 → Hook 3 → null
```

---

## 3. mountState - 첫 렌더링

### 소스코드 분석

```javascript
// react-reconciler/src/ReactFiberHooks.js

function mountState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>] {
  // 1. 새 Hook 생성 및 연결 리스트에 추가
  const hook = mountWorkInProgressHook();

  // 2. 초기값이 함수면 실행
  if (typeof initialState === "function") {
    initialState = initialState();
  }

  // 3. 상태 저장
  hook.memoizedState = initialState;
  hook.baseState = initialState;

  // 4. 업데이트 큐 생성
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null, // 대기 중인 업데이트
    lanes: NoLanes, // 우선순위
    dispatch: null, // setState 함수
    lastRenderedReducer: basicStateReducer, // 상태 계산 함수
    lastRenderedState: initialState, // 마지막 렌더링된 상태
  };
  hook.queue = queue;

  // 5. dispatch (setState) 함수 생성
  const dispatch: Dispatch<BasicStateAction<S>> = (queue.dispatch = dispatchSetState.bind(
    null,
    currentlyRenderingFiber, // 현재 컴포넌트
    queue // 업데이트 큐
  ));

  // 6. [state, setState] 반환
  return [hook.memoizedState, dispatch];
}
```

### mountWorkInProgressHook

```javascript
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 첫 번째 Hook
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 이후 Hook: 연결 리스트에 추가
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}
```

**동작 과정**:

```javascript
// 컴포넌트
function Counter() {
  const [count, setCount] = useState(0);     // ← 호출
  const [name, setName] = useState('Alice'); // ← 호출
}

// 내부 동작
// 1. mountState(0) 호출
//    → Hook 1 생성: { memoizedState: 0, next: null }
//    → Fiber.memoizedState = Hook 1

// 2. mountState('Alice') 호출
//    → Hook 2 생성: { memoizedState: 'Alice', next: null }
//    → Hook 1.next = Hook 2

// 최종 구조
Fiber.memoizedState → Hook 1 (count: 0) → Hook 2 (name: 'Alice')
```

---

## 4. updateState - 리렌더링

### 소스코드 분석

```javascript
function updateState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, initialState);
}

function updateReducer<S, A>(reducer: (S, A) => S, initialArg: any): [S, Dispatch<A>] {
  // 1. 기존 Hook 가져오기 (연결 리스트에서 순서대로)
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  // 2. 대기 중인 업데이트 처리
  const pending = queue.pending;

  if (pending !== null) {
    // 업데이트 큐를 순회하며 새 상태 계산
    const first = pending.next;
    let newState = hook.baseState;
    let update = first;

    do {
      const action = update.action;
      // 상태 업데이트
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== first);

    // 새 상태 저장
    hook.memoizedState = newState;
    hook.baseState = newState;
    queue.pending = null;
  }

  const dispatch = queue.dispatch;
  return [hook.memoizedState, dispatch];
}
```

### updateWorkInProgressHook

```javascript
function updateWorkInProgressHook(): Hook {
  // 다음 Hook 가져오기 (순서대로!)
  let nextCurrentHook: Hook | null;

  if (currentHook === null) {
    // 첫 번째 Hook
    nextCurrentHook = currentlyRenderingFiber.alternate.memoizedState;
  } else {
    // 다음 Hook
    nextCurrentHook = currentHook.next;
  }

  if (nextCurrentHook === null) {
    // Hook이 더 없음 - 순서 변경됨!
    throw new Error("Rendered more hooks than during the previous render.");
  }

  currentHook = nextCurrentHook;

  // 기존 Hook 복사
  const newHook: Hook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };

  // 연결 리스트에 추가
  if (workInProgressHook === null) {
    workInProgressHook = newHook;
  } else {
    workInProgressHook = workInProgressHook.next = newHook;
  }

  return workInProgressHook;
}
```

**왜 순서가 중요한가?**

```javascript
// 첫 렌더링
function Component() {
  const [a, setA] = useState(1); // Hook 1
  const [b, setB] = useState(2); // Hook 2
  const [c, setC] = useState(3); // Hook 3
}
// Hook 리스트: 1 → 2 → 3

// 조건문으로 순서 변경하면?
function Component() {
  const [a, setA] = useState(1); // Hook 1
  if (condition) {
    const [b, setB] = useState(2); // Hook 2 (조건부!)
  }
  const [c, setC] = useState(3); // Hook 2 또는 3?
}

// 리렌더링 시 Hook 리스트 불일치!
// updateWorkInProgressHook가 엉뚱한 Hook을 가져옴
```

---

## 5. dispatchSetState - setState 호출

### 소스코드 분석

```javascript
function dispatchSetState<S, A>(
  fiber: Fiber, // 컴포넌트
  queue: UpdateQueue<S, A>, // 업데이트 큐
  action: A // 새 상태 또는 함수
) {
  // 1. 업데이트 객체 생성
  const update: Update<S, A> = {
    lane: lane, // 우선순위
    action: action, // 새 상태
    next: null,
  };

  // 2. 업데이트 큐에 추가
  const pending = queue.pending;
  if (pending === null) {
    // 첫 업데이트: 순환 리스트 생성
    update.next = update;
  } else {
    // 기존 업데이트 뒤에 추가
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;

  // 3. 리렌더링 스케줄링
  const root = scheduleUpdateOnFiber(fiber, lane);

  // 4. (최적화) 즉시 상태 계산 시도
  if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes)) {
    // Eager evaluation
    const lastRenderedReducer = queue.lastRenderedReducer;
    const currentState = queue.lastRenderedState;
    const eagerState = lastRenderedReducer(currentState, action);

    // 상태가 같으면 리렌더링 스킵!
    if (Object.is(eagerState, currentState)) {
      return;
    }
  }
}
```

### 업데이트 큐 (Circular Linked List)

```javascript
// setState 3번 호출
setCount(1);
setCount(2);
setCount(3);

// 업데이트 큐 구조 (순환 리스트)
queue.pending → Update3
                  ↓
Update1 → Update2 → Update3 → Update1 (순환)
```

### Batching (배치 처리)

```javascript
function handleClick() {
  setCount(1); // ← 업데이트 1
  setCount(2); // ← 업데이트 2
  setCount(3); // ← 업데이트 3
  // 여기까지 리렌더링 안 함!
}
// 이벤트 핸들러 종료 후 한 번만 리렌더링!
```

**이유**:

- 성능 최적화
- 여러 setState를 **모아서 한 번에** 처리
- React 18부터는 `setTimeout`, Promise에서도 자동 배칭!

---

## 6. 직접 구현해보는 useState

### 간단한 버전

```javascript
// 전역 변수
let currentComponent = null;
let hookIndex = 0;

const MyReact = {
  hooks: [],

  useState(initialValue) {
    const index = hookIndex;

    // 첫 렌더링: Hook 생성
    if (this.hooks[index] === undefined) {
      this.hooks[index] = {
        value: typeof initialValue === "function" ? initialValue() : initialValue,

        setValue: (newValue) => {
          // 함수형 업데이트 지원
          const value = typeof newValue === "function" ? newValue(this.hooks[index].value) : newValue;

          // 상태가 실제로 변경된 경우만 리렌더링
          if (!Object.is(value, this.hooks[index].value)) {
            this.hooks[index].value = value;
            this.render(currentComponent);
          }
        },
      };
    }

    hookIndex++;
    return [this.hooks[index].value, this.hooks[index].setValue];
  },

  render(Component) {
    currentComponent = Component;
    hookIndex = 0; // 리셋!
    const output = Component();
    currentComponent = null;
    return output;
  },
};
```

### 사용 예제

```javascript
function Counter() {
  const [count, setCount] = MyReact.useState(0);
  const [name, setName] = MyReact.useState("Alice");

  return {
    count,
    name,
    increment: () => setCount((prev) => prev + 1),
    changeName: (newName) => setName(newName),
  };
}

// 테스트
let app = MyReact.render(Counter);
console.log(app.count); // 0
console.log(app.name); // 'Alice'

app.increment();
app = MyReact.render(Counter);
console.log(app.count); // 1

app.changeName("Bob");
app = MyReact.render(Counter);
console.log(app.name); // 'Bob'
```

### 연결 리스트 버전 (더 React스럽게)

```javascript
class Hook {
  constructor(initialValue) {
    this.value = initialValue;
    this.next = null;
  }
}

const MyReact = {
  currentFiber: null,
  currentHook: null,

  useState(initialValue) {
    const fiber = this.currentFiber;

    // 마운트: 새 Hook 생성
    if (!this.currentHook) {
      const hook = new Hook(initialValue);

      if (!fiber.hooks) {
        fiber.hooks = hook;
      } else {
        let last = fiber.hooks;
        while (last.next) last = last.next;
        last.next = hook;
      }

      this.currentHook = hook;
    }
    // 업데이트: 다음 Hook 가져오기
    else {
      this.currentHook = this.currentHook.next;
    }

    const hook = this.currentHook;

    const setValue = (newValue) => {
      hook.value = typeof newValue === "function" ? newValue(hook.value) : newValue;

      this.render(fiber.component);
    };

    return [hook.value, setValue];
  },

  render(Component) {
    const fiber = { component: Component, hooks: null };
    this.currentFiber = fiber;
    this.currentHook = fiber.hooks; // 시작점

    const output = Component();

    this.currentFiber = null;
    this.currentHook = null;

    return output;
  },
};
```

---

## 7. useEffect 내부 구조

### Effect 자료구조

```javascript
type Effect = {
  tag: EffectTag, // 효과 유형
  create: () => (() => void) | void, // Effect 함수
  destroy: (() => void) | null, // Cleanup 함수
  deps: Array<any> | null, // 의존성 배열
  next: Effect | null, // 다음 Effect (연결 리스트)
};
```

### mountEffect

```javascript
function mountEffect(create: () => (() => void) | void, deps: Array<any> | void): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;

  hook.memoizedState = pushEffect(
    HookHasEffect | HookPassive, // tag
    create, // effect 함수
    undefined, // cleanup (아직 없음)
    nextDeps // dependencies
  );
}
```

### updateEffect

```javascript
function updateEffect(create: () => (() => void) | void, deps: Array<any> | void): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy; // 이전 cleanup

    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;

      // 의존성 비교
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 변경 없음: Effect 스킵
        pushEffect(HookPassive, create, destroy, nextDeps);
        return;
      }
    }
  }

  // 의존성 변경: Effect 실행 예약
  hook.memoizedState = pushEffect(HookHasEffect | HookPassive, create, destroy, nextDeps);
}
```

### areHookInputsEqual - 의존성 비교

```javascript
function areHookInputsEqual(nextDeps: Array<any>, prevDeps: Array<any> | null): boolean {
  if (prevDeps === null) {
    return false;
  }

  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    // Object.is 사용 (===과 거의 동일)
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

**중요**:

- `Object.is`로 **얕은 비교**
- 객체/배열은 **참조**를 비교!

```javascript
// ❌ 매번 새 배열 → 항상 다름
useEffect(() => {
  // ...
}, [{ userId: 1 }]); // 새 객체!

// ✅ 변수 사용
const user = { userId: 1 };
useEffect(() => {
  // ...
}, [user]);
```

---

## 8. 정리

### Hook의 핵심 원리

1. **연결 리스트**: Hook은 순서대로 연결 리스트에 저장
2. **Dispatcher 패턴**: 렌더링 상태에 따라 다른 함수 실행
3. **Fiber 노드**: 각 컴포넌트는 Fiber 노드로 표현
4. **업데이트 큐**: setState는 큐에 추가 후 배치 처리
5. **얕은 비교**: 의존성과 상태는 Object.is로 비교

### 왜 이렇게 설계했을까?

- **단순성**: 배열 인덱스 대신 연결 리스트
- **성능**: 배치 처리로 불필요한 렌더링 방지
- **예측 가능성**: 순수 함수 원칙 유지
- **확장성**: Custom Hooks로 로직 재사용

### 실무에 적용하기

- Hook 호출 순서 절대 변경 금지
- 의존성 배열 정확히 명시
- 객체/배열 참조 변경에 주의
- React DevTools로 Hook 상태 디버깅

**다음**: [05-Virtual-DOM-Reconciliation.md](./05-Virtual-DOM-Reconciliation.md)
