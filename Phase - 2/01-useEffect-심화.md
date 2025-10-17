# useEffect 심화

## 📌 학습 목표

- useEffect의 실행 타이밍을 정확히 이해
- Cleanup 함수가 언제, 왜 실행되는지 파악
- 의존성 배열 관련 일반적인 함정 회피
- useLayoutEffect와의 차이점 이해
- Race Condition 해결 방법

---

## 1. useEffect 실행 타이밍 상세 분석

### 기본 실행 순서

```jsx
function Component() {
  console.log("1. 렌더링 시작");

  useEffect(() => {
    console.log("3. useEffect 실행");

    return () => {
      console.log("4. Cleanup 실행 (다음 effect 전 또는 언마운트 시)");
    };
  });

  console.log("2. 렌더링 완료 (return 전)");

  return <div>Component</div>;
}
```

**출력 순서**:

```
1. 렌더링 시작
2. 렌더링 완료 (return 전)
[DOM 업데이트]
[브라우저 paint]
3. useEffect 실행
```

### 비동기 실행의 의미

```jsx
// useEffect는 브라우저 paint 이후 실행
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // 이 코드는 화면이 업데이트된 후 실행됨
    console.log("Effect:", count);
  }, [count]);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**타임라인**:

```
1. setCount(1) 호출
2. 컴포넌트 리렌더링
3. Virtual DOM 생성
4. DOM 업데이트
5. 브라우저가 화면에 그림 ← 사용자가 즉시 볼 수 있음
6. useEffect 실행 ← 비동기
```

### Android와 비교

```kotlin
// Android - Lifecycle
class MyFragment : Fragment() {
    override fun onResume() {
        super.onResume()
        // 화면이 보이는 시점에 동기적으로 실행
        updateUI()
    }
}
```

```jsx
// React - useEffect
function MyComponent() {
  useEffect(() => {
    // 화면이 업데이트된 후 비동기로 실행
    updateUI();
  }, []);
}
```

---

## 2. useLayoutEffect - 동기 실행

### useEffect vs useLayoutEffect

```jsx
function EffectComparison() {
  const [width, setWidth] = useState(0);
  const ref = useRef(null);

  // ❌ useEffect: 화면 깜빡임 발생 가능
  useEffect(() => {
    setWidth(ref.current.offsetWidth);
  }, []);

  // ✅ useLayoutEffect: 화면에 그려지기 전 실행
  useLayoutEffect(() => {
    setWidth(ref.current.offsetWidth);
  }, []);

  return <div ref={ref}>Width: {width}</div>;
}
```

**실행 타임라인**:

```
[useEffect]
1. 렌더링
2. DOM 업데이트
3. 브라우저 paint ← 사용자가 width: 0을 봄 (깜빡임)
4. useEffect 실행
5. setWidth 호출
6. 리렌더링
7. 브라우저 paint ← width: 300

[useLayoutEffect]
1. 렌더링
2. DOM 업데이트
3. useLayoutEffect 실행 (paint 전)
4. setWidth 호출
5. 리렌더링
6. 브라우저 paint ← 사용자는 width: 300만 봄 (깜빡임 없음)
```

### 언제 useLayoutEffect를 사용?

```jsx
// ✅ 사용해야 하는 경우
useLayoutEffect(() => {
  // 1. DOM 측정 (offsetWidth, getBoundingClientRect 등)
  const rect = elementRef.current.getBoundingClientRect();

  // 2. DOM 조작 (스크롤 위치, 포커스 등)
  elementRef.current.scrollTo(0, 100);

  // 3. 애니메이션 시작 위치 설정
  elementRef.current.style.transform = "translateX(0)";
}, []);

// ❌ 대부분의 경우는 useEffect로 충분
useEffect(() => {
  // API 호출
  fetchData();

  // 이벤트 리스너 등록
  window.addEventListener("resize", handler);

  // 로깅
  analytics.track("page_view");
}, []);
```

**주의**: useLayoutEffect는 **동기적으로** 실행되어 화면 업데이트를 차단하므로, 무거운 작업은 피해야 합니다!

---

## 3. Cleanup 함수 상세 분석

### Cleanup이 실행되는 시점

```jsx
function Timer({ interval }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log(`Effect 실행 (interval: ${interval})`);

    const timer = setInterval(() => {
      setCount((c) => c + 1);
    }, interval);

    return () => {
      console.log(`Cleanup 실행 (interval: ${interval})`);
      clearInterval(timer);
    };
  }, [interval]);

  return <div>{count}</div>;
}
```

**시나리오 1: interval prop 변경**

```
// interval: 1000
Effect 실행 (interval: 1000)

// interval: 500으로 변경
Cleanup 실행 (interval: 1000) ← 이전 effect 정리
Effect 실행 (interval: 500)    ← 새 effect 실행
```

**시나리오 2: 컴포넌트 언마운트**

```
Cleanup 실행 (interval: 500)
[컴포넌트 제거]
```

### Cleanup이 필요한 경우들

#### 1. 타이머

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log("tick");
  }, 1000);

  // ✅ 반드시 정리
  return () => clearInterval(timer);
}, []);
```

#### 2. 이벤트 리스너

```jsx
useEffect(() => {
  const handleScroll = () => console.log("scrolled");
  window.addEventListener("scroll", handleScroll);

  // ✅ 메모리 누수 방지
  return () => window.removeEventListener("scroll", handleScroll);
}, []);
```

#### 3. 구독 (Subscription)

```jsx
useEffect(() => {
  const subscription = dataSource.subscribe((data) => {
    setData(data);
  });

  // ✅ 구독 해제
  return () => subscription.unsubscribe();
}, []);
```

#### 4. API 요청 취소

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch("/api/data", { signal: controller.signal })
    .then((res) => res.json())
    .then(setData)
    .catch((err) => {
      if (err.name !== "AbortError") {
        console.error(err);
      }
    });

  // ✅ 컴포넌트 언마운트 시 요청 취소
  return () => controller.abort();
}, []);
```

---

## 4. 의존성 배열 완벽 가이드

### 문제 1: 객체/배열을 의존성에 넣으면?

```jsx
// ❌ 매번 새 객체 생성 → 무한 루프!
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const filters = { active: true }; // 매 렌더링마다 새 객체

  useEffect(() => {
    fetchUser(userId, filters).then(setUser);
  }, [filters]); // filters가 매번 다름!
}
```

**해결 방법들**:

```jsx
// ✅ 해결 1: 원시값으로 분리
const isActive = true;
useEffect(() => {
  fetchUser(userId, { active: isActive });
}, [userId, isActive]);

// ✅ 해결 2: useMemo로 메모이제이션
const filters = useMemo(() => ({ active: true }), []);
useEffect(() => {
  fetchUser(userId, filters);
}, [userId, filters]);

// ✅ 해결 3: effect 내부에서 생성
useEffect(() => {
  const filters = { active: true };
  fetchUser(userId, filters);
}, [userId]);
```

### 문제 2: 함수를 의존성에 넣으면?

```jsx
// ❌ 매번 새 함수 생성 → 무한 루프!
function SearchBox() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  const search = () => {
    api.search(query).then(setResults);
  };

  useEffect(() => {
    search();
  }, [search]); // search가 매번 다름!
}
```

**해결 방법들**:

```jsx
// ✅ 해결 1: effect 내부로 이동
useEffect(() => {
  const search = () => {
    api.search(query).then(setResults);
  };
  search();
}, [query]);

// ✅ 해결 2: useCallback 사용
const search = useCallback(() => {
  api.search(query).then(setResults);
}, [query]);

useEffect(() => {
  search();
}, [search]); // search는 query 변경 시에만 바뀜
```

### 문제 3: State를 읽지만 의존성에 없으면?

```jsx
// ❌ count가 항상 0
function BrokenCounter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      console.log("Count:", count); // 항상 0!
      setCount(count + 1); // 0 + 1 = 1만 반복
    }, 1000);

    return () => clearInterval(timer);
  }, []); // count를 의존성에 안 넣음
}
```

**해결 방법**:

```jsx
// ✅ 해결 1: 함수형 업데이트
useEffect(() => {
  const timer = setInterval(() => {
    setCount((c) => c + 1); // 최신 count 사용
  }, 1000);

  return () => clearInterval(timer);
}, []); // 의존성 불필요

// ✅ 해결 2: 의존성에 추가 (매번 재생성되지만 동작함)
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(timer);
}, [count]);
```

---

## 5. Race Condition 해결

### 문제 상황

```jsx
// ❌ Race Condition 발생 가능
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then((data) => {
      setUser(data); // 이전 요청이 늦게 도착하면?
    });
  }, [userId]);
}
```

**시나리오**:

```
1. userId = 1 → fetchUser(1) 시작
2. userId = 2 → fetchUser(2) 시작
3. fetchUser(2) 완료 → setUser(user2) ✅
4. fetchUser(1) 완료 → setUser(user1) ❌ (잘못된 사용자!)
```

### 해결 방법 1: Cleanup으로 무시

```jsx
// ✅ 이전 요청 무시
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let ignore = false;

    fetchUser(userId).then((data) => {
      if (!ignore) {
        setUser(data); // 최신 요청만 반영
      }
    });

    return () => {
      ignore = true; // 이전 요청 무시
    };
  }, [userId]);
}
```

### 해결 방법 2: AbortController

```jsx
// ✅ 이전 요청 취소
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => res.json())
      .then(setUser)
      .catch((err) => {
        if (err.name !== "AbortError") {
          console.error(err);
        }
      });

    return () => controller.abort(); // 요청 취소
  }, [userId]);
}
```

---

## 6. useEffect 디버깅 팁

### 의존성 배열 디버깅

```jsx
function useTraceUpdate(props) {
  const prev = useRef(props);

  useEffect(() => {
    const changedProps = Object.entries(props).reduce((acc, [key, val]) => {
      if (prev.current[key] !== val) {
        acc[key] = {
          from: prev.current[key],
          to: val,
        };
      }
      return acc;
    }, {});

    if (Object.keys(changedProps).length > 0) {
      console.log("Changed props:", changedProps);
    }

    prev.current = props;
  });
}

// 사용
function MyComponent(props) {
  useTraceUpdate(props);
  // ...
}
```

### Effect 실행 횟수 추적

```jsx
function useEffectDebugger(effectName, effect, deps) {
  const renderCount = useRef(0);

  useEffect(() => {
    renderCount.current += 1;
    console.log(`[${effectName}] 실행 횟수: ${renderCount.current}`);
    return effect();
  }, deps);
}

// 사용
useEffectDebugger(
  "Fetch User",
  () => {
    fetchUser(userId).then(setUser);
  },
  [userId]
);
```

---

## 7. 실전 패턴

### 패턴 1: 디바운스 검색

```jsx
function SearchBox() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  useEffect(() => {
    // 빈 문자열이면 검색 안 함
    if (!query.trim()) {
      setResults([]);
      return;
    }

    // 500ms 지연
    const timer = setTimeout(() => {
      api.search(query).then(setResults);
    }, 500);

    // 타이핑 중에는 이전 타이머 취소
    return () => clearTimeout(timer);
  }, [query]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

### 패턴 2: 폴링 (Polling)

```jsx
function DataPolling({ interval = 5000 }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    // 즉시 한 번 실행
    fetchData().then(setData);

    // 주기적으로 실행
    const timer = setInterval(() => {
      fetchData().then(setData);
    }, interval);

    return () => clearInterval(timer);
  }, [interval]);

  return <div>{data}</div>;
}
```

### 패턴 3: 이전 값 비교

```jsx
function Component({ value }) {
  const prevValue = useRef();

  useEffect(() => {
    if (prevValue.current !== undefined && prevValue.current !== value) {
      console.log(`Changed from ${prevValue.current} to ${value}`);
    }
    prevValue.current = value;
  }, [value]);
}
```

---

## 8. 정리

### useEffect 체크리스트

- [ ] 의존성 배열에 effect에서 사용하는 모든 값을 포함했는가?
- [ ] 타이머/리스너/구독은 cleanup에서 정리하는가?
- [ ] Race Condition 가능성을 고려했는가?
- [ ] 객체/배열 의존성은 적절히 메모이제이션했는가?
- [ ] useLayoutEffect가 필요한 경우가 아닌가?

### 다음 단계

useEffect를 완벽히 이해했다면, 이제 성능 최적화 Hooks를 배울 차례입니다!

**다음**: [02-성능-최적화-Hooks.md](./02-성능-최적화-Hooks.md)
