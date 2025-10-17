# Props와 State

## 1. Props (Properties)

### Props란?

부모 컴포넌트가 자식 컴포넌트에게 전달하는 **읽기 전용 데이터**입니다.

```jsx
// 부모 컴포넌트
function App() {
  return <Greeting name="철수" age={25} />;
}

// 자식 컴포넌트
function Greeting(props) {
  return (
    <View>
      <Text>안녕하세요, {props.name}님!</Text>
      <Text>나이: {props.age}세</Text>
    </View>
  );
}
```

### 구조 분해로 간결하게

```jsx
// props 객체로 받기
function Greeting(props) {
  return <Text>{props.name}</Text>;
}

// 구조 분해 (권장)
function Greeting({ name, age }) {
  return (
    <View>
      <Text>{name}</Text>
      <Text>{age}</Text>
    </View>
  );
}

// 기본값 설정
function Greeting({ name = "게스트", age = 0 }) {
  return <Text>{name}</Text>;
}
```

### Props의 특징

#### 1. 읽기 전용 (Immutable)

```jsx
function Greeting({ name }) {
  // ❌ 안됨: Props는 수정 불가
  name = "영희";

  return <Text>{name}</Text>;
}
```

**이유**:

- **단방향 데이터 흐름** 유지 (One-way data flow)
- Props를 수정하면 부모와 자식의 상태가 불일치
- 순수 함수 원칙 (같은 입력 → 같은 출력)

#### 2. 단방향 흐름

```
부모 (App)
  ↓ props 전달
자식 (Greeting)
```

자식이 데이터를 변경하려면? → **콜백 함수**를 props로 전달!

```jsx
function Parent() {
  const [name, setName] = useState("철수");

  return <Child name={name} onChangeName={(newName) => setName(newName)} />;
}

function Child({ name, onChangeName }) {
  return (
    <View>
      <Text>{name}</Text>
      <Button onPress={() => onChangeName("영희")} />
    </View>
  );
}
```

### Android와 비교

#### Android: Intent/Bundle

```kotlin
// 부모 Activity
val intent = Intent(this, ChildActivity::class.java)
intent.putExtra("name", "철수")
intent.putExtra("age", 25)
startActivity(intent)

// 자식 Activity
class ChildActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val name = intent.getStringExtra("name")
        val age = intent.getIntExtra("age", 0)
    }
}
```

#### Android: Custom View Attributes

```xml
<!-- attrs.xml -->
<declare-styleable name="GreetingView">
    <attr name="name" format="string" />
    <attr name="age" format="integer" />
</declare-styleable>

<!-- layout.xml -->
<com.example.GreetingView
    app:name="철수"
    app:age="25" />
```

```kotlin
class GreetingView(context: Context, attrs: AttributeSet) : View(context, attrs) {
    private val name: String
    private val age: Int

    init {
        val typedArray = context.obtainStyledAttributes(attrs, R.styleable.GreetingView)
        name = typedArray.getString(R.styleable.GreetingView_name) ?: ""
        age = typedArray.getInt(R.styleable.GreetingView_age, 0)
        typedArray.recycle()
    }
}
```

#### React: 훨씬 간단!

```jsx
<Greeting name="철수" age={25} />;

function Greeting({ name, age }) {
  return (
    <Text>
      {name}, {age}
    </Text>
  );
}
```

### Props의 다양한 타입

```jsx
function Component({
  // 기본 타입
  title, // string
  count, // number
  isActive, // boolean

  // 객체
  user, // { name: string, age: number }

  // 배열
  items, // string[]

  // 함수 (이벤트 핸들러)
  onPress, // () => void
  onChangeName, // (name: string) => void

  // JSX 요소
  children, // ReactNode
  header, // ReactNode
}) {
  return (
    <View>
      <Text>{title}</Text>
      <Text>{user.name}</Text>
      {items.map((item) => (
        <Text key={item}>{item}</Text>
      ))}
      <Button onPress={onPress} />
      {header}
      {children}
    </View>
  );
}

// 사용
<Component
  title="제목"
  count={10}
  isActive={true}
  user={{ name: "철수", age: 25 }}
  items={["a", "b", "c"]}
  onPress={() => console.log("clicked")}
  header={<Text>Header</Text>}
>
  <Text>Children content</Text>
</Component>;
```

### Children Props

특별한 prop: 컴포넌트 사이에 있는 내용

```jsx
function Card({ children }) {
  return <View style={styles.card}>{children}</View>;
}

// 사용
<Card>
  <Text>제목</Text>
  <Text>내용</Text>
  <Button title="확인" />
</Card>;
```

**Layout 컴포넌트에 유용**:

```jsx
function Container({ children }) {
  return <View style={styles.container}>{children}</View>;
}

function App() {
  return (
    <Container>
      <Header />
      <Content />
      <Footer />
    </Container>
  );
}
```

---

## 2. State

### State란?

컴포넌트 **내부에서 관리**하는 **변경 가능한** 데이터입니다.

```jsx
import { useState } from "react";

function Counter() {
  // [상태 값, 상태 변경 함수] = useState(초기값)
  const [count, setCount] = useState(0);

  return (
    <View>
      <Text>카운트: {count}</Text>
      <Button title="증가" onPress={() => setCount(count + 1)} />
    </View>
  );
}
```

### useState 사용법

#### 기본 사용

```jsx
const [count, setCount] = useState(0);

// 상태 변경
setCount(10); // 직접 값 설정
setCount(count + 1); // 이전 값 기반 업데이트
```

#### 함수형 업데이트

```jsx
// ❌ 비동기 이슈 가능
setCount(count + 1);
setCount(count + 1); // 여전히 count + 1

// ✅ 함수형 업데이트 (권장)
setCount((prev) => prev + 1);
setCount((prev) => prev + 1); // prev는 최신 값
```

**이유**: setState는 **비동기**로 동작!

```jsx
const [count, setCount] = useState(0);

function handleClick() {
  setCount(count + 1);
  console.log(count); // ❌ 여전히 0! (즉시 반영 안됨)
}

// 최신 값 사용하려면
function handleClick() {
  setCount((prev) => {
    console.log(prev); // ✅ 최신 값
    return prev + 1;
  });
}
```

#### 여러 State 사용

```jsx
function Form() {
  const [name, setName] = useState("");
  const [age, setAge] = useState(0);
  const [isActive, setIsActive] = useState(false);

  return (
    <View>
      <TextInput value={name} onChangeText={setName} />
      <Text>{age}</Text>
      <Switch value={isActive} onValueChange={setIsActive} />
    </View>
  );
}
```

#### 객체 State

```jsx
const [user, setUser] = useState({
  name: "철수",
  age: 25,
  city: "서울",
});

// ❌ 안됨: 직접 수정
user.age = 26;
setUser(user); // React가 변경 감지 못함!

// ✅ 새 객체 생성 (불변성 유지)
setUser({ ...user, age: 26 });

// 또는
setUser((prev) => ({
  ...prev,
  age: prev.age + 1,
}));
```

#### 배열 State

```jsx
const [items, setItems] = useState(["A", "B", "C"]);

// 추가
setItems([...items, "D"]);
setItems((prev) => [...prev, "D"]);

// 삭제 (인덱스 1 제거)
setItems(items.filter((_, index) => index !== 1));

// 수정 (인덱스 1을 "X"로)
setItems(items.map((item, index) => (index === 1 ? "X" : item)));

// ❌ 안됨: 원본 배열 변경
items.push("D");
setItems(items); // React가 변경 감지 못함!
```

### Android와 비교

#### Android: ViewModel + LiveData

```kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData(0)
    val count: LiveData<Int> = _count

    fun increment() {
        _count.value = (_count.value ?: 0) + 1
    }
}

// Activity/Fragment
class MainActivity : AppCompatActivity() {
    private val viewModel: CounterViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        viewModel.count.observe(this) { count ->
            binding.textView.text = "Count: $count"
        }

        binding.button.setOnClickListener {
            viewModel.increment()
        }
    }
}
```

#### React: 훨씬 간단!

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <View>
      <Text>Count: {count}</Text>
      <Button onPress={() => setCount(count + 1)} />
    </View>
  );
}
```

### iOS SwiftUI와 비교

```swift
struct Counter: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") {
                count += 1
            }
        }
    }
}
```

---

## 3. 불변성 (Immutability)

### 왜 불변성이 중요한가?

React는 **참조(reference)를 비교**해서 변경을 감지합니다.

```jsx
const [user, setUser] = useState({ name: "철수", age: 25 });

// ❌ 안됨: 같은 참조
user.age = 26;
setUser(user);
// React: "참조가 같네? 변경 없음!" → 리렌더링 안 함

// ✅ 새 객체: 다른 참조
setUser({ ...user, age: 26 });
// React: "참조가 다르네! 변경됨!" → 리렌더링
```

### 불변성 유지 패턴

#### 객체 업데이트

```jsx
const [user, setUser] = useState({
  name: "철수",
  age: 25,
  address: {
    city: "서울",
    district: "강남",
  },
});

// 1단계 업데이트
setUser({ ...user, age: 26 });

// 중첩 객체 업데이트
setUser({
  ...user,
  address: {
    ...user.address,
    city: "부산",
  },
});
```

#### 배열 업데이트

```jsx
const [items, setItems] = useState([1, 2, 3]);

// 추가 (끝)
setItems([...items, 4]);

// 추가 (앞)
setItems([0, ...items]);

// 삭제 (ID 기준)
setItems(items.filter((item) => item.id !== deleteId));

// 수정 (ID 기준)
setItems(items.map((item) => (item.id === targetId ? { ...item, completed: true } : item)));

// 정렬 (원본 변경!)
setItems([...items].sort((a, b) => a - b));
```

#### 복잡한 State: Immer 라이브러리

```jsx
import { useImmer } from "use-immer";

function TodoList() {
  const [todos, setTodos] = useImmer([{ id: 1, text: "Buy milk", done: false }]);

  // 불변성 신경 안 써도 됨!
  function toggleTodo(id) {
    setTodos((draft) => {
      const todo = draft.find((t) => t.id === id);
      todo.done = !todo.done; // 직접 수정 OK!
    });
  }
}
```

---

## 4. Props vs State

### 비교표

| 구분             | Props         | State              |
| ---------------- | ------------- | ------------------ |
| **소유자**       | 부모 컴포넌트 | 해당 컴포넌트      |
| **변경 가능**    | ❌ 읽기 전용  | ✅ setState로 변경 |
| **전달**         | 부모 → 자식   | 해당 컴포넌트 내부 |
| **리렌더링**     | Props 변경 시 | State 변경 시      |
| **용도**         | 컴포넌트 설정 | 동적 데이터 관리   |
| **Android 비유** | Intent/Bundle | ViewModel/LiveData |
| **iOS 비유**     | init 파라미터 | @State             |

### 언제 Props를 사용?

- 부모로부터 받은 데이터를 표시
- 재사용 가능한 컴포넌트 (Button, Card 등)
- 설정 값 (색상, 크기 등)

```jsx
function Button({ title, color, onPress }) {
  return (
    <TouchableOpacity
      onPress={onPress}
      style={{ backgroundColor: color }}
    >
      <Text>{title}</Text>
    </TouchableOpacity>
  );
}

<Button title="확인" color="blue" onPress={handleSubmit} />
<Button title="취소" color="gray" onPress={handleCancel} />
```

### 언제 State를 사용?

- 사용자 입력 (폼, 검색어 등)
- 토글 상태 (모달, 탭 등)
- API로 받은 데이터
- 시간에 따라 변하는 값 (타이머 등)

```jsx
function SearchBar() {
  const [query, setQuery] = useState(""); // 입력값
  const [results, setResults] = useState([]); // API 결과

  return (
    <View>
      <TextInput value={query} onChangeText={setQuery} />
      {results.map((result) => (
        <Text key={result.id}>{result.name}</Text>
      ))}
    </View>
  );
}
```

### Props를 State로 변환하기

```jsx
// ❌ 안티패턴: Props를 State 초기값으로만 사용
function Counter({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  // initialCount가 변경되어도 count는 업데이트 안됨!

  return <Text>{count}</Text>;
}

// ✅ 패턴 1: Props 직접 사용
function Counter({ count }) {
  return <Text>{count}</Text>;
}

// ✅ 패턴 2: 명확하게 의도 표현
function Counter({ defaultCount }) {
  const [count, setCount] = useState(defaultCount);
  // "default"라는 이름으로 초기값임을 명시

  return <Text>{count}</Text>;
}

// ✅ 패턴 3: useEffect로 동기화
function Counter({ count: propCount }) {
  const [count, setCount] = useState(propCount);

  useEffect(() => {
    setCount(propCount);
  }, [propCount]);

  return <Text>{count}</Text>;
}
```

---

## 5. 상태 끌어올리기 (Lifting State Up)

### 문제 상황

두 컴포넌트가 **같은 데이터를 공유**해야 할 때?

```jsx
// ❌ 각자 State를 가지면 동기화 안됨
function InputA() {
  const [value, setValue] = useState("");
  return <TextInput value={value} onChangeText={setValue} />;
}

function InputB() {
  const [value, setValue] = useState("");
  return <TextInput value={value} onChangeText={setValue} />;
}
```

### 해결: 공통 부모로 State 이동

```jsx
function Parent() {
  const [value, setValue] = useState(""); // State를 부모로!

  return (
    <View>
      <InputA value={value} onChange={setValue} />
      <InputB value={value} onChange={setValue} />
    </View>
  );
}

function InputA({ value, onChange }) {
  return <TextInput value={value} onChangeText={onChange} />;
}

function InputB({ value, onChange }) {
  return <TextInput value={value} onChangeText={onChange} />;
}
```

### 실제 예제: 온도 변환기

```jsx
function TemperatureConverter() {
  const [celsius, setCelsius] = useState(0);

  const fahrenheit = (celsius * 9) / 5 + 32;

  return (
    <View>
      <TemperatureInput label="Celsius" value={celsius} onChange={setCelsius} />
      <TemperatureInput label="Fahrenheit" value={fahrenheit} onChange={(f) => setCelsius(((f - 32) * 5) / 9)} />
    </View>
  );
}

function TemperatureInput({ label, value, onChange }) {
  return (
    <View>
      <Text>{label}</Text>
      <TextInput value={String(value)} onChangeText={(text) => onChange(Number(text))} />
    </View>
  );
}
```

---

## 6. 조건부 렌더링

### if/else

```jsx
function Greeting({ isLoggedIn }) {
  if (isLoggedIn) {
    return <Text>환영합니다!</Text>;
  }
  return <Text>로그인하세요</Text>;
}
```

### 삼항 연산자

```jsx
function Greeting({ isLoggedIn }) {
  return <Text>{isLoggedIn ? "환영합니다!" : "로그인하세요"}</Text>;
}
```

### && 연산자

```jsx
function Notification({ hasUnread, count }) {
  return (
    <View>
      <Text>알림</Text>
      {hasUnread && <Text>읽지 않은 알림: {count}개</Text>}
    </View>
  );
}
```

**주의**: falsy 값 렌더링

```jsx
const count = 0;

// ❌ 0이 화면에 표시됨!
{
  count && <Text>Count: {count}</Text>;
}

// ✅ boolean으로 변환
{
  count > 0 && <Text>Count: {count}</Text>;
}
{
  Boolean(count) && <Text>Count: {count}</Text>;
}
```

---

## 7. 리스트 렌더링

### map() 사용

```jsx
function UserList() {
  const users = [
    { id: 1, name: "철수" },
    { id: 2, name: "영희" },
    { id: 3, name: "민수" },
  ];

  return (
    <View>
      {users.map((user) => (
        <Text key={user.id}>{user.name}</Text>
      ))}
    </View>
  );
}
```

### key prop의 중요성

```jsx
// ❌ 안됨: key 없음
{
  users.map((user) => <Text>{user.name}</Text>);
}
// Warning: Each child should have a unique "key" prop

// ❌ 안티패턴: index를 key로
{
  users.map((user, index) => <Text key={index}>{user.name}</Text>);
}
// 순서가 바뀌면 버그!

// ✅ 고유한 ID를 key로
{
  users.map((user) => <Text key={user.id}>{user.name}</Text>);
}
```

**왜 key가 필요한가?**

React가 어떤 항목이 **변경/추가/삭제**되었는지 추적하기 위해!

```jsx
// 초기 리스트
[
  { id: 1, name: "A" },
  { id: 2, name: "B" },
  { id: 3, name: "C" },
]

// 항목 삭제 후
[
  { id: 1, name: "A" },
  { id: 3, name: "C" },
]

// key가 있으면: id=2 삭제됨을 인식 → B만 제거
// key가 없으면: 모든 항목을 다시 렌더링
```

### FlatList (React Native)

```jsx
import { FlatList } from 'react-native';

function UserList() {
  const users = [...];

  return (
    <FlatList
      data={users}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => (
        <Text>{item.name}</Text>
      )}
    />
  );
}
```

---

## 8. 정리

### 핵심 개념

1. **Props**: 부모 → 자식 데이터 전달 (읽기 전용)
2. **State**: 컴포넌트 내부 상태 관리 (변경 가능)
3. **불변성**: State 변경 시 새 객체/배열 생성
4. **Lifting State Up**: 공유 State는 공통 부모로

### Props와 State 선택 가이드

```
데이터가 변경되나요?
├─ NO → Props 사용
└─ YES → State 사용
    │
    └─ 여러 컴포넌트가 공유하나요?
        ├─ NO → 해당 컴포넌트의 State
        └─ YES → 공통 부모의 State + Props로 전달
```

### 다음 단계

- Hooks 심화 (useEffect, useCallback, useMemo)
- Context API로 깊은 Props 전달 문제 해결
- State 관리 라이브러리 (Redux, Zustand)

**다음**: [03-Hooks-기초.md](./03-Hooks-기초.md)
