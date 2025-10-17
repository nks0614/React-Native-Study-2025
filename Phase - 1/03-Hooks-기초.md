# Hooks 기초

## 1. Hooks의 탄생 배경

### Class Component의 문제점

#### 1. 로직 재사용의 어려움

```jsx
// Higher-Order Component (HOC) - 복잡함
function withUser(Component) {
  return class extends React.Component {
    state = { user: null };

    componentDidMount() {
      fetchUser().then((user) => this.setState({ user }));
    }

    render() {
      return <Component user={this.state.user} {...this.props} />;
    }
  };
}

// Render Props - 중첩 지옥
<UserProvider>
  {(user) => (
    <ThemeProvider>
      {(theme) => (
        <LanguageProvider>{(language) => <Component user={user} theme={theme} language={language} />}</LanguageProvider>
      )}
    </ThemeProvider>
  )}
</UserProvider>;
```

#### 2. 복잡한 Lifecycle 메서드

```jsx
class UserProfile extends React.Component {
  componentDidMount() {
    // 데이터 fetch
    fetchUser(this.props.userId).then((user) => this.setState({ user }));

    // 이벤트 리스너 등록
    window.addEventListener("resize", this.handleResize);

    // 타이머 시작
    this.timer = setInterval(this.updateTime, 1000);
  }

  componentDidUpdate(prevProps) {
    // userId 변경 시 다시 fetch
    if (prevProps.userId !== this.props.userId) {
      fetchUser(this.props.userId).then((user) => this.setState({ user }));
    }
  }

  componentWillUnmount() {
    // 정리 작업이 여기저기 흩어짐
    window.removeEventListener("resize", this.handleResize);
    clearInterval(this.timer);
  }
}
```

**문제**: 관련 있는 로직이 **여러 메서드에 분산**되어 있음!

#### 3. this의 혼란

```jsx
class Counter extends React.Component {
  state = { count: 0 };

  handleClick() {
    // ❌ this가 undefined!
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}

// 해결책 1: bind
<button onClick={this.handleClick.bind(this)}>

// 해결책 2: arrow function
<button onClick={() => this.handleClick()}>

// 해결책 3: class field
handleClick = () => {
  this.setState({ count: this.state.count + 1 });
}
```

### Hooks가 해결한 것들

```jsx
// ✅ Hooks: 간결하고 명확
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  // 모든 관련 로직이 한 곳에!
  useEffect(() => {
    fetchUser(userId).then(setUser);

    const handleResize = () => console.log("resized");
    window.addEventListener("resize", handleResize);

    const timer = setInterval(() => console.log("tick"), 1000);

    // 정리 작업도 함께
    return () => {
      window.removeEventListener("resize", handleResize);
      clearInterval(timer);
    };
  }, [userId]); // userId 변경 시 재실행

  return <Text>{user?.name}</Text>;
}
```

---

## 2. useState

### 기본 사용법

```jsx
import { useState } from "react";

function Counter() {
  // [현재 값, 설정 함수] = useState(초기값)
  const [count, setCount] = useState(0);

  return (
    <View>
      <Text>Count: {count}</Text>
      <Button onPress={() => setCount(count + 1)} />
    </View>
  );
}
```

### 지연 초기화 (Lazy Initialization)

```jsx
// ❌ 매 렌더링마다 실행
const [state, setState] = useState(expensiveCalculation());

// ✅ 첫 렌더링에만 실행
const [state, setState] = useState(() => expensiveCalculation());
```

**사용 예**:

```jsx
function TodoList() {
  // localStorage에서 읽기 - 무거운 작업
  const [todos, setTodos] = useState(() => {
    const saved = localStorage.getItem("todos");
    return saved ? JSON.parse(saved) : [];
  });
}
```

### 함수형 업데이트

```jsx
// ❌ 동시 업데이트 시 문제
setCount(count + 1);
setCount(count + 1); // 여전히 count + 1

// ✅ 이전 값 기반 업데이트
setCount((prev) => prev + 1);
setCount((prev) => prev + 1); // prev는 최신 값
```

**실제 예제**: 좋아요 버튼

```jsx
function LikeButton() {
  const [likes, setLikes] = useState(0);

  const handleLike = async () => {
    // API 호출 (비동기)
    await api.like();

    // ❌ 안됨: 여러 번 클릭 시 likes가 오래된 값
    setLikes(likes + 1);

    // ✅ 함수형 업데이트
    setLikes((prev) => prev + 1);
  };

  return <Button onPress={handleLike}>{likes}</Button>;
}
```

---

## 3. useEffect

### 기본 개념

**Side Effect**(부수 효과): 컴포넌트 렌더링 외부에 영향을 주는 작업

- API 호출
- DOM 조작
- 타이머 설정
- 이벤트 리스너 등록
- 로깅

```jsx
useEffect(() => {
  // 실행할 작업
  console.log("Effect runs!");

  // 정리 함수 (cleanup)
  return () => {
    console.log("Cleanup!");
  };
}, [dependencies]); // 의존성 배열
```

### 실행 시점

#### 1. 의존성 배열 없음 - 매 렌더링마다

```jsx
useEffect(() => {
  console.log("매 렌더링마다 실행");
});
```

#### 2. 빈 배열 - 마운트/언마운트 시만

```jsx
useEffect(() => {
  console.log("마운트 시 실행");

  return () => {
    console.log("언마운트 시 실행");
  };
}, []); // ← 빈 배열
```

**Android 비유**:

```kotlin
// onCreate
override fun onCreate(savedInstanceState: Bundle?) {
    console.log('마운트 시 실행')
}

// onDestroy
override fun onDestroy() {
    console.log('언마운트 시 실행')
}
```

#### 3. 의존성 배열 - 특정 값 변경 시

```jsx
useEffect(() => {
  console.log(`userId가 ${userId}로 변경됨`);
}, [userId]); // userId 변경 시만 실행
```

### 실제 예제

#### 1. API 호출

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // 로딩 시작
    setLoading(true);
    setError(null);

    // API 호출
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => {
        setUser(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err.message);
        setLoading(false);
      });
  }, [userId]); // userId 변경 시 재호출

  if (loading) return <Text>로딩 중...</Text>;
  if (error) return <Text>에러: {error}</Text>;
  return <Text>{user?.name}</Text>;
}
```

#### 2. 이벤트 리스너

```jsx
function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };

    // 등록
    window.addEventListener("resize", handleResize);

    // 정리: 언마운트 시 제거
    return () => {
      window.removeEventListener("resize", handleResize);
    };
  }, []); // 한 번만 실행

  return (
    <Text>
      {size.width} x {size.height}
    </Text>
  );
}
```

#### 3. 타이머

```jsx
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds((prev) => prev + 1);
    }, 1000);

    // 정리: interval 제거
    return () => clearInterval(interval);
  }, []);

  return <Text>{seconds}초</Text>;
}
```

#### 4. 구독 (Subscription)

```jsx
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    // 구독 시작
    const subscription = chatAPI.subscribe(roomId, (message) => {
      setMessages((prev) => [...prev, message]);
    });

    // 정리: 구독 해제
    return () => {
      subscription.unsubscribe();
    };
  }, [roomId]); // roomId 변경 시 재구독

  return (
    <View>
      {messages.map((msg) => (
        <Text key={msg.id}>{msg.text}</Text>
      ))}
    </View>
  );
}
```

### Cleanup 함수의 중요성

```jsx
// ❌ Cleanup 없음 - 메모리 누수!
useEffect(() => {
  const interval = setInterval(() => {
    console.log("tick");
  }, 1000);
  // 컴포넌트가 언마운트되어도 interval은 계속 실행!
}, []);

// ✅ Cleanup 있음
useEffect(() => {
  const interval = setInterval(() => {
    console.log("tick");
  }, 1000);

  return () => clearInterval(interval); // 정리!
}, []);
```

### 의존성 배열 규칙

#### 규칙: Effect 내부에서 사용하는 모든 값은 의존성 배열에 포함!

```jsx
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    // query를 사용함
    fetch(`/api/search?q=${query}`)
      .then((res) => res.json())
      .then(setResults);
  }, [query]); // ← query를 의존성에 포함!
}
```

#### 린터 경고 무시하지 말기

```jsx
// ❌ 안됨: 린터 경고 무시
useEffect(() => {
  doSomething(props.value);
}, []); // ← props.value가 변경되어도 실행 안됨!

// ✅ 의존성 추가
useEffect(() => {
  doSomething(props.value);
}, [props.value]);
```

#### 함수를 의존성에서 제외하기

```jsx
// ❌ 매 렌더링마다 fetchData가 새로 생성됨
function SearchResults({ query }) {
  const fetchData = () => {
    fetch(`/api/search?q=${query}`).then(...)
  };

  useEffect(() => {
    fetchData();
  }, [fetchData]); // fetchData가 매번 바뀜 → 무한 루프!
}

// ✅ 해결 1: Effect 내부로 이동
useEffect(() => {
  const fetchData = () => {
    fetch(`/api/search?q=${query}`).then(...)
  };
  fetchData();
}, [query]);

// ✅ 해결 2: useCallback 사용 (나중에 배움)
const fetchData = useCallback(() => {
  fetch(`/api/search?q=${query}`).then(...)
}, [query]);

useEffect(() => {
  fetchData();
}, [fetchData]);
```

---

## 4. Hooks의 규칙

### 규칙 1: 최상위에서만 호출

```jsx
// ❌ 안됨: 조건문 안에서 Hook 호출
function Component({ condition }) {
  if (condition) {
    const [state, setState] = useState(0); // Error!
  }
}

// ❌ 안됨: 반복문 안에서 Hook 호출
function Component() {
  for (let i = 0; i < 10; i++) {
    const [state, setState] = useState(i); // Error!
  }
}

// ✅ 최상위에서 호출
function Component({ condition }) {
  const [state, setState] = useState(0);

  if (condition) {
    // state 사용은 OK
    setState(10);
  }
}
```

**이유**: React는 **Hook 호출 순서**로 State를 추적하기 때문!

```jsx
// 첫 번째 렌더링
useState(0); // Hook 1
useState(10); // Hook 2
useState(20); // Hook 3

// 두 번째 렌더링
useState(0); // Hook 1 - 이전 값 가져옴
useState(10); // Hook 2 - 이전 값 가져옴
useState(20); // Hook 3 - 이전 값 가져옴

// 조건문에서 Hook을 호출하면?
useState(0); // Hook 1
if (condition) {
  useState(10); // Hook 2 - 조건에 따라 실행 안될 수도!
}
useState(20); // Hook 2 또는 3? - 순서 꼬임!
```

### 규칙 2: React 함수에서만 호출

```jsx
// ❌ 일반 JavaScript 함수
function calculateTotal() {
  const [total, setTotal] = useState(0); // Error!
}

// ✅ React 함수 컴포넌트
function Component() {
  const [total, setTotal] = useState(0); // OK
}

// ✅ Custom Hook (이름이 use로 시작)
function useTotal() {
  const [total, setTotal] = useState(0); // OK
  return total;
}
```

### ESLint 플러그인

```bash
npm install eslint-plugin-react-hooks
```

```json
// .eslintrc
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

---

## 5. Class Component vs Hooks

### 동일한 컴포넌트 비교

#### Class Component

```jsx
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0,
      name: "Guest",
    };
  }

  componentDidMount() {
    document.title = `Count: ${this.state.count}`;
  }

  componentDidUpdate() {
    document.title = `Count: ${this.state.count}`;
  }

  componentWillUnmount() {
    console.log("Unmounting...");
  }

  handleIncrement = () => {
    this.setState({ count: this.state.count + 1 });
  };

  handleNameChange = (e) => {
    this.setState({ name: e.target.value });
  };

  render() {
    return (
      <div>
        <input value={this.state.name} onChange={this.handleNameChange} />
        <p>Count: {this.state.count}</p>
        <button onClick={this.handleIncrement}>+</button>
      </div>
    );
  }
}
```

#### Hooks (함수 컴포넌트)

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("Guest");

  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);

  useEffect(() => {
    return () => {
      console.log("Unmounting...");
    };
  }, []);

  return (
    <div>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}
```

**차이점**:

- 코드 줄 수: **40줄 → 20줄**
- this 바인딩 불필요
- 관련 로직이 함께 위치
- 더 읽기 쉬움

---

## 6. 자주 사용하는 Hooks

### useContext

전역 상태 공유

```jsx
const ThemeContext = React.createContext("light");

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Component />
    </ThemeContext.Provider>
  );
}

function Component() {
  const theme = useContext(ThemeContext); // "dark"
  return <Text>Current theme: {theme}</Text>;
}
```

### useRef

DOM 참조 또는 값 유지

```jsx
function TextInputWithFocus() {
  const inputRef = useRef(null);

  const handleFocus = () => {
    inputRef.current.focus();
  };

  return (
    <View>
      <TextInput ref={inputRef} />
      <Button onPress={handleFocus} title="포커스" />
    </View>
  );
}
```

**useRef vs useState**:

```jsx
// useRef: 변경해도 리렌더링 안 함
const countRef = useRef(0);
countRef.current += 1; // 리렌더링 없음

// useState: 변경 시 리렌더링
const [count, setCount] = useState(0);
setCount(count + 1); // 리렌더링!
```

### useReducer

복잡한 State 관리

```jsx
const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <View>
      <Text>Count: {state.count}</Text>
      <Button onPress={() => dispatch({ type: "increment" })} />
      <Button onPress={() => dispatch({ type: "decrement" })} />
    </View>
  );
}
```

---

## 7. 정리

### Hooks의 장점

1. **간결함**: Class보다 적은 코드
2. **로직 재사용**: Custom Hooks
3. **관련 로직 그룹화**: useEffect로 모음
4. **this 없음**: 더 직관적

### Hooks 사용 체크리스트

- [ ] 최상위에서만 호출했는가?
- [ ] React 함수에서만 호출했는가?
- [ ] useEffect의 의존성 배열이 올바른가?
- [ ] Cleanup 함수가 필요한가?
- [ ] 린터 경고를 무시하지 않았는가?

### 다음 단계

- useState/useEffect 내부 구조 이해
- Custom Hooks 만들기
- 성능 최적화 Hooks (useMemo, useCallback)

**다음**: [04-Hooks-내부구조.md](./04-Hooks-내부구조.md)
