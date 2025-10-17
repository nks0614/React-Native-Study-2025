# 성능 최적화 Hooks

## 📌 학습 목표

- useRef로 렌더링 없이 값을 유지하는 방법
- useMemo로 비용이 큰 계산을 최적화하는 방법
- useCallback으로 함수를 메모이제이션하는 방법
- React.memo와 조합하여 리렌더링 최적화
- 언제 최적화해야 하는지 판단 기준
- 실제 성능 측정 방법

---

## 1. useRef - 렌더링 없이 값 유지

### 기본 개념

```jsx
const ref = useRef(initialValue);
// ref = { current: initialValue }

ref.current = newValue; // 변경해도 리렌더링 없음
```

### useState vs useRef

```jsx
function Comparison() {
  const [count1, setCount1] = useState(0);
  const count2 = useRef(0);

  const handleClick = () => {
    setCount1(count1 + 1); // 리렌더링 발생 ✅
    count2.current = count2.current + 1; // 리렌더링 없음 ⚠️

    console.log("State:", count1); // 화면에 표시됨
    console.log("Ref:", count2.current); // 화면에 표시 안됨
  };

  return (
    <div>
      <p>State: {count1}</p>
      <p>Ref: {count2.current}</p>
      <button onClick={handleClick}>증가</button>
    </div>
  );
}
```

**출력 결과**:

- `count1`은 화면에 즉시 반영됨
- `count2.current`는 다음 리렌더링 때까지 화면에 반영 안 됨

### 사용 사례 1: DOM 요소 참조

```jsx
function TextInputWithFocus() {
  const inputRef = useRef(null);

  const handleFocus = () => {
    inputRef.current?.focus();
  };

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={handleFocus}>포커스</button>
    </>
  );
}
```

**Android 비교**:

```kotlin
class MyFragment : Fragment() {
  private lateinit var inputField: EditText

  override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    inputField = view.findViewById(R.id.input_field)
  }

  fun focusInput() {
    inputField.requestFocus()
  }
}
```

### 사용 사례 2: 이전 값 저장

```jsx
function PreviousValue({ value }) {
  const prevValue = useRef();

  useEffect(() => {
    prevValue.current = value;
  });

  return (
    <div>
      <p>현재 값: {value}</p>
      <p>이전 값: {prevValue.current}</p>
    </div>
  );
}
```

### 사용 사례 3: 타이머 ID 저장

```jsx
function Timer() {
  const [seconds, setSeconds] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    if (intervalRef.current !== null) return; // 이미 실행 중

    intervalRef.current = setInterval(() => {
      setSeconds((s) => s + 1);
    }, 1000);
  };

  const stop = () => {
    if (intervalRef.current === null) return;

    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  const reset = () => {
    stop();
    setSeconds(0);
  };

  // Cleanup
  useEffect(() => {
    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);

  return (
    <div>
      <p>{seconds}초</p>
      <button onClick={start}>시작</button>
      <button onClick={stop}>정지</button>
      <button onClick={reset}>리셋</button>
    </div>
  );
}
```

### 사용 사례 4: 렌더링 횟수 추적

```jsx
function RenderCounter() {
  const renderCount = useRef(0);

  // 매 렌더링마다 증가 (리렌더링 유발 없이)
  renderCount.current += 1;

  return <div>렌더링 횟수: {renderCount.current}</div>;
}
```

---

## 2. useMemo - 값 메모이제이션

### 기본 개념

```jsx
const memoizedValue = useMemo(() => {
  return expensiveCalculation(a, b);
}, [a, b]); // a 또는 b가 변경될 때만 재계산
```

### 문제 상황: 불필요한 재계산

```jsx
// ❌ 매 렌더링마다 정렬 실행
function ProductList({ products, filter }) {
  // filter 변경 시뿐만 아니라 부모 리렌더링 시에도 정렬!
  const sortedProducts = [...products].filter((p) => p.category === filter).sort((a, b) => b.price - a.price);

  return (
    <div>
      {sortedProducts.map((p) => (
        <ProductItem key={p.id} product={p} />
      ))}
    </div>
  );
}
```

### 해결: useMemo 적용

```jsx
// ✅ products 또는 filter 변경 시에만 정렬
function ProductList({ products, filter }) {
  const sortedProducts = useMemo(() => {
    console.log("정렬 실행"); // 의존성 변경 시에만 출력
    return [...products].filter((p) => p.category === filter).sort((a, b) => b.price - a.price);
  }, [products, filter]);

  return (
    <div>
      {sortedProducts.map((p) => (
        <ProductItem key={p.id} product={p} />
      ))}
    </div>
  );
}
```

### 실전 예제 1: 복잡한 계산

```jsx
function DataAnalysis({ data }) {
  const statistics = useMemo(() => {
    console.log("통계 계산 중...");

    const sum = data.reduce((acc, val) => acc + val, 0);
    const avg = sum / data.length;
    const max = Math.max(...data);
    const min = Math.min(...data);

    return { sum, avg, max, min };
  }, [data]);

  return (
    <div>
      <p>합계: {statistics.sum}</p>
      <p>평균: {statistics.avg}</p>
      <p>최대: {statistics.max}</p>
      <p>최소: {statistics.min}</p>
    </div>
  );
}
```

### 실전 예제 2: 참조 안정성 유지

```jsx
function UserList({ users }) {
  // ❌ 매 렌더링마다 새 배열 생성 → 자식 컴포넌트 불필요한 리렌더링
  const activeUsers = users.filter((u) => u.active);

  // ✅ users 변경 시에만 새 배열 생성
  const activeUsers = useMemo(() => {
    return users.filter((u) => u.active);
  }, [users]);

  return <MemoizedUserTable users={activeUsers} />;
}

const MemoizedUserTable = React.memo(UserTable);
```

---

## 3. useCallback - 함수 메모이제이션

### 기본 개념

```jsx
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]); // a 또는 b가 변경될 때만 함수 재생성
```

### 문제 상황: 불필요한 자식 리렌더링

```jsx
// ❌ 부모가 리렌더링될 때마다 자식도 리렌더링
function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState("");

  // 매 렌더링마다 새 함수 생성!
  const handleClick = () => {
    console.log("Clicked");
  };

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <p>Count: {count}</p>

      {/* text 변경 시에도 Child가 리렌더링됨 */}
      <Child onClick={handleClick} />
    </div>
  );
}

const Child = React.memo(({ onClick }) => {
  console.log("Child 렌더링");
  return <button onClick={onClick}>클릭</button>;
});
```

**문제**: `handleClick`이 매번 새로 생성되므로 `React.memo`가 무용지물!

### 해결: useCallback 적용

```jsx
// ✅ 함수를 메모이제이션
function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState("");

  // 함수를 한 번만 생성
  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []); // 의존성 없음

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <p>Count: {count}</p>

      {/* text 변경 시 Child는 리렌더링 안 됨! */}
      <Child onClick={handleClick} />
    </div>
  );
}
```

### 실전 예제 1: State 업데이트 함수

```jsx
function TodoList() {
  const [todos, setTodos] = useState([]);

  // ✅ 함수형 업데이트 사용 → 의존성 없음
  const addTodo = useCallback((text) => {
    setTodos((prev) => [...prev, { id: Date.now(), text }]);
  }, []);

  const deleteTodo = useCallback((id) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  }, []);

  const toggleTodo = useCallback((id) => {
    setTodos((prev) => prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t)));
  }, []);

  return (
    <div>
      <TodoForm onSubmit={addTodo} />
      {todos.map((todo) => (
        <TodoItem key={todo.id} todo={todo} onDelete={deleteTodo} onToggle={toggleTodo} />
      ))}
    </div>
  );
}
```

### 실전 예제 2: 외부 값 사용

```jsx
function SearchBox({ searchAPI }) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  // query가 변경될 때만 함수 재생성
  const handleSearch = useCallback(async () => {
    const data = await searchAPI(query);
    setResults(data);
  }, [query, searchAPI]);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button onClick={handleSearch}>검색</button>
      <ResultList results={results} />
    </div>
  );
}
```

---

## 4. React.memo와 조합

### React.memo 기본

```jsx
// 컴포넌트를 메모이제이션: props가 동일하면 리렌더링 스킵
const MemoizedComponent = React.memo(function Component({ name, age }) {
  console.log("렌더링:", name);
  return (
    <div>
      {name} ({age})
    </div>
  );
});
```

### 얕은 비교 (Shallow Comparison)

```jsx
// React.memo는 props를 얕은 비교
<MemoizedComponent
  name="John" // 문자열: 값 비교 ✅
  age={30} // 숫자: 값 비교 ✅
  user={{ name: "John" }} // 객체: 참조 비교 ⚠️
  onClick={() => {}} // 함수: 참조 비교 ⚠️
/>
```

### 완벽한 최적화 조합

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState("");

  // 객체 메모이제이션
  const user = useMemo(
    () => ({
      name: "John",
      age: 30,
    }),
    []
  ); // 한 번만 생성

  // 함수 메모이제이션
  const handleClick = useCallback(() => {
    console.log("Clicked", user.name);
  }, [user]);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>증가</button>

      <input value={text} onChange={(e) => setText(e.target.value)} />

      {/* count/text 변경 시에도 Child는 리렌더링 안 됨! */}
      <MemoizedChild user={user} onClick={handleClick} />
    </div>
  );
}

const MemoizedChild = React.memo(function Child({ user, onClick }) {
  console.log("Child 렌더링");
  return (
    <div>
      <p>
        {user.name} ({user.age})
      </p>
      <button onClick={onClick}>클릭</button>
    </div>
  );
});
```

---

## 5. 언제 최적화해야 하는가?

### 최적화가 필요한 경우 ✅

#### 1. 리스트 아이템 컴포넌트

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo) => (
        // ✅ 리스트가 크면 최적화 필수
        <MemoizedTodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

const MemoizedTodoItem = React.memo(TodoItem);
```

#### 2. 무거운 계산

```jsx
function DataVisualization({ data }) {
  // ✅ 복잡한 계산은 메모이제이션
  const processedData = useMemo(() => {
    return data.map((item) => ({
      ...item,
      normalized: (item.value - min) / (max - min),
      trend: calculateTrend(item.history),
    }));
  }, [data]);
}
```

#### 3. 자주 렌더링되는 부모의 자식

```jsx
function ChatApp() {
  const [messages, setMessages] = useState([]);
  const [typingUsers, setTypingUsers] = useState([]);

  // ✅ typingUsers가 자주 변경되어도 UserList는 안정적
  const users = useMemo(() => getUsers(), []);

  return (
    <div>
      <UserList users={users} />
      <MessageList messages={messages} />
      <TypingIndicator users={typingUsers} />
    </div>
  );
}
```

### 최적화가 불필요한 경우 ❌

#### 1. 단순한 컴포넌트

```jsx
// ❌ 불필요한 최적화
function SimpleComponent({ text }) {
  return <div>{text}</div>;
}

// useMemo를 쓰면 오히려 비용 증가
```

#### 2. 이미 빠른 연산

```jsx
// ❌ 불필요
const doubled = useMemo(() => count * 2, [count]);

// ✅ 그냥 계산하는 게 빠름
const doubled = count * 2;
```

#### 3. Props가 자주 변경되는 컴포넌트

```jsx
// ❌ 불필요 (filter가 매번 바뀜)
const FilteredList = React.memo(function ({ items, filter }) {
  return items.filter((i) => i.includes(filter));
});
```

### 최적화 판단 기준

```
최적화하기 전에 물어보기:

1. 실제로 성능 문제가 있는가?
   → React DevTools Profiler로 측정

2. 컴포넌트가 자주 리렌더링되는가?
   → console.log로 확인

3. 계산이 무거운가?
   → console.time으로 측정

4. 리스트가 큰가? (100개 이상)
   → 최적화 고려

없으면 최적화하지 말 것!
"조기 최적화는 모든 악의 근원" - Donald Knuth
```

---

## 6. 성능 측정 방법

### React DevTools Profiler

```jsx
// 1. React DevTools 설치
// 2. Profiler 탭에서 기록 시작
// 3. 액션 수행
// 4. 기록 정지
// 5. Flame 차트에서 렌더링 시간 확인

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Component />
    </Profiler>
  );
}

function onRenderCallback(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}
```

### console.time으로 계산 측정

```jsx
function ExpensiveComponent({ data }) {
  console.time("calculation");
  const result = complexCalculation(data);
  console.timeEnd("calculation"); // 출력: calculation: 245ms

  return <div>{result}</div>;
}
```

### 렌더링 횟수 추적

```jsx
function Component() {
  const renderCount = useRef(0);
  renderCount.current += 1;

  console.log("렌더링 횟수:", renderCount.current);
}
```

---

## 7. 실전 최적화 예제

### Before: 최적화 전

```jsx
// ❌ 성능 문제
function ProductCatalog({ products, category }) {
  const [sortBy, setSortBy] = useState("name");

  // 매 렌더링마다 필터링+정렬 (느림!)
  const displayProducts = products
    .filter((p) => p.category === category)
    .sort((a, b) => (a[sortBy] > b[sortBy] ? 1 : -1));

  const handleSort = (field) => {
    setSortBy(field);
  };

  return (
    <div>
      <button onClick={() => handleSort("name")}>이름순</button>
      <button onClick={() => handleSort("price")}>가격순</button>

      {displayProducts.map((p) => (
        <ProductCard key={p.id} product={p} onAddToCart={() => addToCart(p)} />
      ))}
    </div>
  );
}
```

### After: 최적화 후

```jsx
// ✅ 성능 최적화
function ProductCatalog({ products, category }) {
  const [sortBy, setSortBy] = useState("name");

  // 필터링+정렬 메모이제이션
  const displayProducts = useMemo(() => {
    console.log("필터링+정렬 실행");
    return products.filter((p) => p.category === category).sort((a, b) => (a[sortBy] > b[sortBy] ? 1 : -1));
  }, [products, category, sortBy]);

  // 핸들러 메모이제이션
  const handleSort = useCallback((field) => {
    setSortBy(field);
  }, []);

  const handleAddToCart = useCallback((product) => {
    addToCart(product);
  }, []);

  return (
    <div>
      <button onClick={() => handleSort("name")}>이름순</button>
      <button onClick={() => handleSort("price")}>가격순</button>

      {displayProducts.map((p) => (
        <MemoizedProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
      ))}
    </div>
  );
}

const MemoizedProductCard = React.memo(ProductCard);
```

---

## 8. 정리

### 최적화 Hooks 비교표

| Hook        | 용도                  | 반환값   | 언제 사용?                      |
| ----------- | --------------------- | -------- | ------------------------------- |
| useRef      | 값 유지               | ref      | 리렌더링 없이 값 저장/DOM 참조  |
| useMemo     | 값 메모이제이션       | 값       | 무거운 계산, 참조 안정성        |
| useCallback | 함수 메모이제이션     | 함수     | 자식 컴포넌트 props로 함수 전달 |
| React.memo  | 컴포넌트 메모이제이션 | 컴포넌트 | 리스트 아이템, 무거운 컴포넌트  |

### 최적화 체크리스트

- [ ] 실제 성능 문제가 있는지 측정했는가?
- [ ] 무거운 계산을 useMemo로 메모이제이션했는가?
- [ ] 자식에게 전달하는 함수를 useCallback으로 메모이제이션했는가?
- [ ] React.memo를 적절히 사용했는가?
- [ ] 의존성 배열이 정확한가?
- [ ] 과도한 최적화를 하지 않았는가?

**다음**: [03-Custom-Hooks.md](./03-Custom-Hooks.md)
