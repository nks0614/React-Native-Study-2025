# Custom Hooks 마스터

## 📌 학습 목표

- Custom Hooks의 개념과 설계 원칙 이해
- 실무에서 자주 사용하는 Custom Hooks 패턴
- 재사용 가능한 Hooks 작성 방법
- Hooks 조합과 의존성 관리
- 테스트 가능한 Hooks 설계

---

## 1. Custom Hooks란?

### 정의

> **Custom Hook**: `use`로 시작하는 이름을 가진 함수로, 내부에서 다른 Hooks를 호출할 수 있는 재사용 가능한 로직

```jsx
// Custom Hook
function useWindowSize() {
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

    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return size;
}

// 사용
function MyComponent() {
  const { width, height } = useWindowSize();
  return (
    <div>
      {width} x {height}
    </div>
  );
}
```

### 일반 함수 vs Custom Hook

```jsx
// ❌ 일반 함수 - Hooks 사용 불가
function getWindowSize() {
  const [size, setSize] = useState(...); // Error!
  return size;
}

// ✅ Custom Hook - Hooks 사용 가능
function useWindowSize() {
  const [size, setSize] = useState(...); // OK!
  return size;
}
```

### Android와 비교

```kotlin
// Android - Extension Function/Helper
fun Context.showToast(message: String) {
    Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
}

// 사용
context.showToast("Hello")
```

```jsx
// React - Custom Hook
function useToast() {
  const [message, setMessage] = useState(null);

  const showToast = useCallback((msg) => {
    setMessage(msg);
    setTimeout(() => setMessage(null), 3000);
  }, []);

  return { message, showToast };
}

// 사용
function MyComponent() {
  const { message, showToast } = useToast();
  return (
    <>
      {message && <Toast>{message}</Toast>}
      <button onClick={() => showToast("Hello")}>Show</button>
    </>
  );
}
```

---

## 2. Custom Hooks 설계 원칙

### 원칙 1: 명확한 책임

```jsx
// ❌ 너무 많은 책임
function useEverything() {
  const user = useUser();
  const theme = useTheme();
  const locale = useLocale();
  return { user, theme, locale };
}

// ✅ 단일 책임
function useUser() {
  // 사용자 관련 로직만
}

function useTheme() {
  // 테마 관련 로직만
}
```

### 원칙 2: 명확한 인터페이스

```jsx
// ❌ 불명확한 반환값
function useData() {
  return [data, loading, error, refetch, cancel];
}

// ✅ 객체로 명확하게
function useData() {
  return {
    data,
    loading,
    error,
    refetch,
    cancel,
  };
}

// ✅ 또는 표준 패턴 따르기 (useState처럼)
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle];
}
```

### 원칙 3: 의존성 명확히

```jsx
// ❌ 숨겨진 의존성
function useFetch(url) {
  const token = getToken(); // 외부 함수 호출
  // ...
}

// ✅ 의존성을 파라미터로
function useFetch(url, token) {
  // ...
}
```

---

## 3. 실무 필수 Custom Hooks

### 1. useFetch - API 호출

```jsx
function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let ignore = false;

    const fetchData = async () => {
      try {
        setLoading(true);
        setError(null);

        const response = await fetch(url, options);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const json = await response.json();

        if (!ignore) {
          setData(json);
        }
      } catch (e) {
        if (!ignore) {
          setError(e.message);
        }
      } finally {
        if (!ignore) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      ignore = true;
    };
  }, [url]);

  return { data, loading, error };
}

// 사용
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <div>로딩 중...</div>;
  if (error) return <div>에러: {error}</div>;
  return <div>{user.name}</div>;
}
```

### 2. useDebounce - 입력 지연

```jsx
function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// 사용
function SearchBox() {
  const [searchTerm, setSearchTerm] = useState("");
  const debouncedSearchTerm = useDebounce(searchTerm, 500);

  useEffect(() => {
    if (debouncedSearchTerm) {
      // API 호출 (타이핑 멈춘 후 500ms 뒤)
      searchAPI(debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);

  return <input value={searchTerm} onChange={(e) => setSearchTerm(e.target.value)} placeholder="검색..." />;
}
```

### 3. useLocalStorage - 로컬 저장소

```jsx
function useLocalStorage(key, initialValue) {
  // State를 lazy initialization으로 초기화
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  // setValue: localStorage에 저장하면서 state 업데이트
  const setValue = useCallback(
    (value) => {
      try {
        // value가 함수면 현재 값으로 실행
        const valueToStore = value instanceof Function ? value(storedValue) : value;

        setStoredValue(valueToStore);
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      } catch (error) {
        console.error(error);
      }
    },
    [key, storedValue]
  );

  return [storedValue, setValue];
}

// 사용
function Settings() {
  const [darkMode, setDarkMode] = useLocalStorage("darkMode", false);
  const [language, setLanguage] = useLocalStorage("language", "ko");

  return (
    <div>
      <label>
        <input type="checkbox" checked={darkMode} onChange={(e) => setDarkMode(e.target.checked)} />
        다크모드
      </label>

      <select value={language} onChange={(e) => setLanguage(e.target.value)}>
        <option value="ko">한국어</option>
        <option value="en">English</option>
      </select>
    </div>
  );
}
```

### 4. useToggle - 토글 상태

```jsx
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => {
    setValue((v) => !v);
  }, []);

  const setTrue = useCallback(() => {
    setValue(true);
  }, []);

  const setFalse = useCallback(() => {
    setValue(false);
  }, []);

  return [value, { toggle, setTrue, setFalse }];
}

// 사용
function Modal() {
  const [isOpen, { toggle, setTrue, setFalse }] = useToggle(false);

  return (
    <>
      <button onClick={setTrue}>모달 열기</button>
      {isOpen && (
        <div className="modal">
          <p>모달 내용</p>
          <button onClick={setFalse}>닫기</button>
          <button onClick={toggle}>토글</button>
        </div>
      )}
    </>
  );
}
```

### 5. usePrevious - 이전 값 저장

```jsx
function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// 사용
function Counter({ count }) {
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>현재: {count}</p>
      <p>이전: {prevCount}</p>
      <p>변화: {count - (prevCount || 0)}</p>
    </div>
  );
}
```

### 6. useIntersectionObserver - 화면 진입 감지

```jsx
function useIntersectionObserver(ref, options = {}) {
  const [isIntersecting, setIntersecting] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      setIntersecting(entry.isIntersecting);
    }, options);

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => {
      if (ref.current) {
        observer.unobserve(ref.current);
      }
    };
  }, [ref, options]);

  return isIntersecting;
}

// 사용: 무한 스크롤
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const loadMoreRef = useRef(null);
  const isVisible = useIntersectionObserver(loadMoreRef);

  useEffect(() => {
    if (isVisible) {
      // 다음 페이지 로드
      loadMore(page).then((newItems) => {
        setItems((prev) => [...prev, ...newItems]);
        setPage((p) => p + 1);
      });
    }
  }, [isVisible]);

  return (
    <div>
      {items.map((item) => (
        <div key={item.id}>{item.name}</div>
      ))}
      <div ref={loadMoreRef}>로딩 중...</div>
    </div>
  );
}
```

### 7. useMediaQuery - 반응형 디자인

```jsx
function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => {
    return window.matchMedia(query).matches;
  });

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (e) => setMatches(e.matches);

    mediaQuery.addEventListener("change", handler);
    return () => mediaQuery.removeEventListener("change", handler);
  }, [query]);

  return matches;
}

// 사용
function ResponsiveComponent() {
  const isMobile = useMediaQuery("(max-width: 768px)");
  const isTablet = useMediaQuery("(min-width: 769px) and (max-width: 1024px)");
  const isDesktop = useMediaQuery("(min-width: 1025px)");

  return (
    <div>
      {isMobile && <MobileView />}
      {isTablet && <TabletView />}
      {isDesktop && <DesktopView />}
    </div>
  );
}
```

### 8. useOnClickOutside - 외부 클릭 감지

```jsx
function useOnClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      // ref 내부 클릭이면 무시
      if (!ref.current || ref.current.contains(event.target)) {
        return;
      }
      handler(event);
    };

    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);

    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref, handler]);
}

// 사용: 드롭다운 메뉴
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef(null);

  useOnClickOutside(dropdownRef, () => {
    setIsOpen(false);
  });

  return (
    <div ref={dropdownRef}>
      <button onClick={() => setIsOpen(!isOpen)}>메뉴</button>
      {isOpen && (
        <ul className="dropdown-menu">
          <li>항목 1</li>
          <li>항목 2</li>
        </ul>
      )}
    </div>
  );
}
```

### 9. useAsync - 비동기 작업 관리

```jsx
function useAsync(asyncFunction, immediate = true) {
  const [status, setStatus] = useState("idle");
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  const execute = useCallback(
    async (...params) => {
      setStatus("pending");
      setData(null);
      setError(null);

      try {
        const response = await asyncFunction(...params);
        setData(response);
        setStatus("success");
        return response;
      } catch (error) {
        setError(error);
        setStatus("error");
        throw error;
      }
    },
    [asyncFunction]
  );

  useEffect(() => {
    if (immediate) {
      execute();
    }
  }, [execute, immediate]);

  return {
    execute,
    status,
    data,
    error,
    isLoading: status === "pending",
    isSuccess: status === "success",
    isError: status === "error",
  };
}

// 사용
function UserProfile({ userId }) {
  const fetchUser = useCallback(() => fetch(`/api/users/${userId}`).then((r) => r.json()), [userId]);

  const { data: user, isLoading, isError, execute: refetch } = useAsync(fetchUser);

  if (isLoading) return <div>로딩 중...</div>;
  if (isError)
    return (
      <div>
        에러 발생 <button onClick={refetch}>재시도</button>
      </div>
    );

  return <div>{user.name}</div>;
}
```

### 10. useForm - 폼 관리

```jsx
function useForm(initialValues, onSubmit) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((event) => {
    const { name, value } = event.target;
    setValues((prev) => ({ ...prev, [name]: value }));
  }, []);

  const handleSubmit = useCallback(
    async (event) => {
      event.preventDefault();
      setIsSubmitting(true);
      setErrors({});

      try {
        await onSubmit(values);
      } catch (error) {
        setErrors(error.errors || {});
      } finally {
        setIsSubmitting(false);
      }
    },
    [values, onSubmit]
  );

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
  }, [initialValues]);

  return {
    values,
    errors,
    isSubmitting,
    handleChange,
    handleSubmit,
    reset,
  };
}

// 사용
function LoginForm() {
  const { values, errors, isSubmitting, handleChange, handleSubmit } = useForm(
    { email: "", password: "" },
    async (values) => {
      await loginAPI(values);
    }
  );

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={values.email} onChange={handleChange} />
      {errors.email && <span>{errors.email}</span>}

      <input name="password" type="password" value={values.password} onChange={handleChange} />
      {errors.password && <span>{errors.password}</span>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "로그인 중..." : "로그인"}
      </button>
    </form>
  );
}
```

---

## 4. Hooks 조합 패턴

### 패턴 1: Hooks가 다른 Custom Hook 사용

```jsx
// 기본 Hook
function useFetch(url) {
  // ...
}

// 조합 Hook
function useUser(userId) {
  const { data, loading, error } = useFetch(`/api/users/${userId}`);

  // 추가 로직
  const isAdmin = data?.role === "admin";

  return { user: data, loading, error, isAdmin };
}
```

### 패턴 2: 여러 Hooks 조합

```jsx
function useAuthenticatedFetch(url) {
  const { token } = useAuth(); // Custom Hook 1
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!token) {
      setLoading(false);
      return;
    }

    fetch(url, {
      headers: { Authorization: `Bearer ${token}` },
    })
      .then((r) => r.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, [url, token]);

  return { data, loading };
}
```

---

## 5. 테스트 가능한 Hooks

### 잘못된 설계

```jsx
// ❌ 테스트 어려움 (외부 의존성)
function useUser(userId) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // API 직접 호출 - 테스트 어려움
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, [userId]);

  return user;
}
```

### 테스트 가능한 설계

```jsx
// ✅ 의존성 주입
function useUser(userId, fetchFn = defaultFetch) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchFn(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, [userId, fetchFn]);

  return user;
}

// 테스트
const mockFetch = jest.fn(() =>
  Promise.resolve({
    json: () => Promise.resolve({ id: 1, name: "Test" }),
  })
);

const { result } = renderHook(() => useUser(1, mockFetch));
```

---

## 6. 정리

### Custom Hooks 작성 체크리스트

- [ ] 이름이 `use`로 시작하는가?
- [ ] 단일 책임 원칙을 따르는가?
- [ ] 반환값이 명확한가?
- [ ] 의존성이 명확히 드러나는가?
- [ ] 재사용 가능한가?
- [ ] 테스트 가능한가?
- [ ] 문서화가 되어 있는가?

### 실무에서 자주 사용하는 Hooks

1. **useFetch** - API 호출
2. **useDebounce** - 입력 지연
3. **useLocalStorage** - 로컬 저장소
4. **useToggle** - 토글 상태
5. **useIntersectionObserver** - 무한 스크롤
6. **useMediaQuery** - 반응형
7. **useOnClickOutside** - 외부 클릭 감지
8. **useAsync** - 비동기 작업
9. **useForm** - 폼 관리
10. **usePrevious** - 이전 값 저장

**다음**: [04-Hooks-내부구조-심화.md](./04-Hooks-내부구조-심화.md)
