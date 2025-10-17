# Virtual DOM과 Reconciliation

## 1. Virtual DOM이란?

### 정의

**Virtual DOM**은 실제 DOM의 **JavaScript 객체 표현**입니다.

```jsx
// JSX
<div className="container">
  <h1>Hello</h1>
  <p>World</p>
</div>

// Virtual DOM (단순화된 구조)
{
  type: 'div',
  props: {
    className: 'container',
    children: [
      {
        type: 'h1',
        props: { children: 'Hello' }
      },
      {
        type: 'p',
        props: { children: 'World' }
      }
    ]
  }
}
```

### 왜 Virtual DOM을 사용하나?

#### 문제: 실제 DOM은 느리다

```javascript
// 실제 DOM 조작 - 매우 비싼 작업
const div = document.createElement("div");
div.className = "container";
div.innerHTML = "<h1>Hello</h1>";
document.body.appendChild(div);

// Reflow, Repaint 발생 → 느림
```

**DOM 조작이 느린 이유**:

1. **Reflow**: 레이아웃 재계산
2. **Repaint**: 화면 다시 그리기
3. **브라우저 API 호출**: JavaScript ↔ C++ 통신 오버헤드

#### 해결: Virtual DOM

```javascript
// 1. 이전 Virtual DOM
const prevVDOM = {
  type: "div",
  props: { children: "Hello" },
};

// 2. 새로운 Virtual DOM
const nextVDOM = {
  type: "div",
  props: { children: "World" },
};

// 3. Diff 계산 (매우 빠름 - JavaScript 객체 비교)
const diff = findDifferences(prevVDOM, nextVDOM);

// 4. 최소한의 DOM 업데이트만 수행
applyDiff(diff);
```

**장점**:

- JavaScript 객체 비교는 **빠름**
- 변경된 부분만 **최소한**으로 DOM 업데이트
- 배치 업데이트로 **Reflow/Repaint 최소화**

### Android 비유

```kotlin
// Android: View 직접 수정
textView.text = "Hello"
textView.textColor = Color.RED
textView.textSize = 16f
// 각 변경마다 즉시 화면 업데이트

// React: Virtual DOM
setState({ text: "Hello", color: "red", size: 16 })
// → Virtual DOM 비교 → 한 번에 업데이트
```

---

## 2. Reconciliation (재조정)

### 정의

이전 Virtual DOM과 새 Virtual DOM을 **비교(Diff)**하여 **최소한의 변경**을 찾는 과정

### Diffing 알고리즘

일반적인 트리 Diff 알고리즘: **O(n³)** - 매우 느림!

React의 휴리스틱 알고리즘: **O(n)** - 매우 빠름!

#### 가정 1: 다른 타입의 엘리먼트는 다른 트리

```jsx
// 이전
<div>
  <Counter />
</div>

// 이후
<span>
  <Counter />
</span>
```

**React의 판단**:

- `div` → `span`: 타입이 다름!
- 전체 서브트리 폐기 후 새로 생성
- `Counter` 컴포넌트도 **언마운트 후 다시 마운트**

#### 가정 2: key prop으로 안정적인 자식 식별

```jsx
// key 없음
<ul>
  <li>A</li>
  <li>B</li>
</ul>

// C를 맨 앞에 추가
<ul>
  <li>C</li>  {/* A를 C로 변경 */}
  <li>A</li>  {/* B를 A로 변경 */}
  <li>B</li>  {/* 새로 추가 */}
</ul>
// → 3개 모두 업데이트!
```

```jsx
// key 있음
<ul>
  <li key="a">A</li>
  <li key="b">B</li>
</ul>

// C를 맨 앞에 추가
<ul>
  <li key="c">C</li>  {/* 새로 추가 */}
  <li key="a">A</li>  {/* 유지 (이동만) */}
  <li key="b">B</li>  {/* 유지 (이동만) */}
</ul>
// → 1개만 추가, 나머지는 이동!
```

---

## 3. Reconciliation 과정 상세

### 단계별 분석

#### 1. Element 타입 비교

```jsx
// Case 1: 타입이 다름
// 이전: <div>Hello</div>
// 이후: <span>Hello</span>
// → 전체 재생성

// Case 2: 타입이 같음
// 이전: <div className="old">Hello</div>
// 이후: <div className="new">Hello</div>
// → Props만 업데이트
```

#### 2. 같은 타입의 DOM 엘리먼트

```jsx
// 이전
<div className="before" title="old" />

// 이후
<div className="after" title="old" />

// React의 동작
// 1. className만 변경됨을 감지
// 2. DOM 노드 유지하고 속성만 업데이트
element.setAttribute('className', 'after');
```

#### 3. 같은 타입의 컴포넌트

```jsx
// 이전
<Counter count={1} />

// 이후
<Counter count={2} />

// React의 동작
// 1. 컴포넌트 인스턴스 유지
// 2. Props 업데이트
// 3. componentDidUpdate 또는 useEffect 실행
// 4. render() 재호출
```

#### 4. 자식 재귀 처리

```jsx
// 이전
<ul>
  <li>A</li>
  <li>B</li>
</ul>

// 이후
<ul>
  <li>A</li>
  <li>B</li>
  <li>C</li>
</ul>

// React의 동작
// 1. <li>A</li> 비교 → 동일
// 2. <li>B</li> 비교 → 동일
// 3. <li>C</li> → 새로 추가
```

---

## 4. Fiber 아키텍처

### Fiber란?

**React 16부터 도입된 새로운 Reconciliation 엔진**

### 이전 문제점 (Stack Reconciler)

```javascript
// Stack Reconciler - 재귀적 처리
function reconcile(element) {
  // DOM 업데이트
  updateDOM(element);

  // 자식들 재귀 처리
  element.children.forEach((child) => {
    reconcile(child); // 중단 불가!
  });
}
```

**문제**:

- 한 번 시작하면 **중단 불가**
- 큰 컴포넌트 트리 → UI 버벅임 (Jank)
- 우선순위 없음 (애니메이션 < 데이터 fetching)

### Fiber의 해결책

**작업을 쪼개서 중단 가능하게!**

```javascript
// Fiber - 작업 단위로 쪼갬
function workLoop(deadline) {
  let shouldYield = false;

  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);

    // 시간이 없으면 브라우저에게 제어권 양도
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (nextUnitOfWork) {
    // 다음 프레임에 계속
    requestIdleCallback(workLoop);
  }
}

requestIdleCallback(workLoop);
```

### Fiber 노드 구조 (복습)

```javascript
type Fiber = {
  // 노드 타입
  type: any, // 'div', Component 등

  // 관계
  child: Fiber | null, // 첫 자식
  sibling: Fiber | null, // 형제
  return: Fiber | null, // 부모

  // 상태
  memoizedState: any, // 현재 State
  memoizedProps: any, // 현재 Props

  // Effect
  effectTag: number, // 수행할 작업 (추가, 업데이트, 삭제)

  // 대체 (Double Buffering)
  alternate: Fiber | null, // 이전 또는 다음 버전
};
```

### Double Buffering

```
Current Tree (화면에 표시 중)
│
├─ div
│  ├─ h1
│  └─ p
│
Work-in-Progress Tree (작업 중)
│
├─ div
│  ├─ h1
│  └─ span  ← 변경됨
```

**동작**:

1. Current Tree: 현재 화면
2. Work-in-Progress Tree: 다음 화면 준비
3. 작업 완료 후 **포인터만 교체** (Commit)

---

## 5. 렌더링 단계 (Render Phase & Commit Phase)

### Render Phase (중단 가능)

**작업**: Virtual DOM 비교, Fiber 트리 생성

```javascript
function performUnitOfWork(fiber) {
  // 1. 컴포넌트 렌더링
  const children = fiber.type(fiber.props);

  // 2. Reconciliation (Diff)
  reconcileChildren(fiber, children);

  // 3. 다음 작업 단위 반환
  if (fiber.child) {
    return fiber.child;
  }

  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.return;
  }
}
```

**특징**:

- **비동기**: 작업을 쪼개서 실행
- **중단 가능**: 우선순위 높은 작업이 있으면 양보
- **부수 효과 없음**: DOM 변경 안 함

### Commit Phase (중단 불가)

**작업**: 실제 DOM 업데이트

```javascript
function commitRoot(root) {
  // 삭제
  root.deletions.forEach(commitWork);

  // 업데이트 및 추가
  commitWork(root.child);

  // Current Tree 교체
  root.current = root.finishedWork;
}

function commitWork(fiber) {
  if (!fiber) return;

  const parentDOM = fiber.return.dom;

  if (fiber.effectTag === "PLACEMENT") {
    // 추가
    parentDOM.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE") {
    // 업데이트
    updateDOM(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    // 삭제
    parentDOM.removeChild(fiber.dom);
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```

**특징**:

- **동기**: 한 번에 실행
- **중단 불가**: 일관성 유지 위해
- **부수 효과**: 실제 DOM 변경

---

## 6. 성능 최적화 기법

### 1. React.memo - 컴포넌트 메모이제이션

```jsx
// 최적화 전
function Child({ name }) {
  console.log("Child render");
  return <Text>{name}</Text>;
}

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <View>
      <Text>{count}</Text>
      <Button onPress={() => setCount(count + 1)} />
      <Child name="Alice" /> {/* count 변경 시에도 리렌더링! */}
    </View>
  );
}
```

```jsx
// 최적화 후
const Child = React.memo(function Child({ name }) {
  console.log("Child render");
  return <Text>{name}</Text>;
});

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <View>
      <Text>{count}</Text>
      <Button onPress={() => setCount(count + 1)} />
      <Child name="Alice" /> {/* Props가 같으면 리렌더링 스킵! */}
    </View>
  );
}
```

**동작 원리**:

```javascript
function memo(Component) {
  return function MemoizedComponent(props) {
    const prevProps = usePrevious(props);

    // Props 얕은 비교
    if (shallowEqual(props, prevProps)) {
      return previousResult; // 이전 결과 재사용
    }

    return Component(props);
  };
}
```

### 2. useMemo - 값 메모이제이션

```jsx
// 최적화 전
function TodoList({ todos, filter }) {
  // todos나 filter가 안 바뀌어도 매번 계산!
  const filteredTodos = todos.filter((todo) => todo.status === filter);

  return filteredTodos.map((todo) => <TodoItem key={todo.id} todo={todo} />);
}
```

```jsx
// 최적화 후
function TodoList({ todos, filter }) {
  // todos나 filter가 바뀔 때만 계산
  const filteredTodos = useMemo(() => todos.filter((todo) => todo.status === filter), [todos, filter]);

  return filteredTodos.map((todo) => <TodoItem key={todo.id} todo={todo} />);
}
```

### 3. useCallback - 함수 메모이제이션

```jsx
// 최적화 전
function Parent() {
  const [count, setCount] = useState(0);

  // 매 렌더링마다 새 함수 생성!
  const handleClick = () => {
    console.log("clicked");
  };

  return (
    <View>
      <Text>{count}</Text>
      <Button onPress={() => setCount(count + 1)} />
      <Child onClick={handleClick} /> {/* 항상 리렌더링! */}
    </View>
  );
}

const Child = React.memo(({ onClick }) => {
  return <Button onPress={onClick} />;
});
```

```jsx
// 최적화 후
function Parent() {
  const [count, setCount] = useState(0);

  // 함수를 메모이제이션
  const handleClick = useCallback(() => {
    console.log("clicked");
  }, []); // 의존성 없음 → 한 번만 생성

  return (
    <View>
      <Text>{count}</Text>
      <Button onPress={() => setCount(count + 1)} />
      <Child onClick={handleClick} /> {/* 리렌더링 스킵! */}
    </View>
  );
}
```

### 4. key 최적화

```jsx
// ❌ 안 좋음: index를 key로
{
  todos.map((todo, index) => <TodoItem key={index} todo={todo} />);
}
// 순서가 바뀌면 모든 항목 리렌더링!

// ✅ 좋음: 고유 ID를 key로
{
  todos.map((todo) => <TodoItem key={todo.id} todo={todo} />);
}
// 정확히 변경된 항목만 업데이트!
```

### 5. 리스트 가상화 (Virtualization)

```jsx
// 1000개 항목 → 1000개 DOM 노드 (느림)
{
  items.map((item) => <Item key={item.id} item={item} />);
}
```

```jsx
// react-window 사용
import { FixedSizeList } from "react-window";

// 화면에 보이는 10개만 렌더링 (빠름)
<FixedSizeList height={500} itemCount={1000} itemSize={50}>
  {({ index, style }) => (
    <div style={style}>
      <Item item={items[index]} />
    </div>
  )}
</FixedSizeList>;
```

---

## 7. 실제 Reconciliation 예제

### 시나리오: Todo 항목 완료 처리

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: "Buy milk", done: false },
    { id: 2, text: "Clean room", done: false },
    { id: 3, text: "Study React", done: false },
  ]);

  const toggleTodo = (id) => {
    setTodos(todos.map((todo) => (todo.id === id ? { ...todo, done: !todo.done } : todo)));
  };

  return (
    <View>
      {todos.map((todo) => (
        <TodoItem key={todo.id} todo={todo} onToggle={toggleTodo} />
      ))}
    </View>
  );
}
```

### Reconciliation 단계별 분석

```javascript
// 1. toggleTodo(2) 호출

// 2. setState → 리렌더링 스케줄링

// 3. Render Phase
// 이전 Virtual DOM
[
  { key: 1, props: { todo: { done: false } } },
  { key: 2, props: { todo: { done: false } } },  // ← 변경됨
  { key: 3, props: { todo: { done: false } } },
]

// 새 Virtual DOM
[
  { key: 1, props: { todo: { done: false } } },
  { key: 2, props: { todo: { done: true } } },   // ← 변경됨
  { key: 3, props: { todo: { done: false } } },
]

// 4. Diff 계산
// key=1: Props 동일 → 업데이트 없음
// key=2: Props 변경 → 업데이트 필요
// key=3: Props 동일 → 업데이트 없음

// 5. Commit Phase
// key=2의 DOM만 업데이트
document.getElementById('todo-2').className = 'todo done';
```

**결과**: 1000개 항목 중 **1개만 DOM 업데이트**!

---

## 8. 정리

### Virtual DOM의 핵심

1. **빠른 비교**: JavaScript 객체 비교는 빠르다
2. **최소 업데이트**: 변경된 부분만 DOM 조작
3. **배치 처리**: 여러 변경을 한 번에
4. **추상화**: 개발자는 상태만 관리, DOM은 React가

### Reconciliation 원칙

1. **타입이 다르면 재생성**
2. **key로 자식 식별**
3. **Props 얕은 비교**
4. **작업을 쪼개서 중단 가능하게** (Fiber)

### 최적화 가이드

- `React.memo`: 불필요한 리렌더링 방지
- `useMemo`: 무거운 계산 캐싱
- `useCallback`: 함수 참조 유지
- `key`: 리스트 항목 안정적으로 식별
- Virtualization: 긴 리스트 최적화

### 실무 팁

- 성능 문제가 있을 때만 최적화
- React DevTools Profiler로 측정
- 불변성 유지 (얕은 비교가 정확히 동작하도록)
- key는 항상 고유하고 안정적인 값 사용

**다음**: [06-실습-프로젝트.md](./06-실습-프로젝트.md)
