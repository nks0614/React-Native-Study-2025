# React 기초

## 1. React란 무엇인가?

### 공식 정의

> "사용자 인터페이스를 만들기 위한 JavaScript 라이브러리"

### 더 정확한 설명

React는 **컴포넌트 기반**의 **선언형** UI 라이브러리입니다. 상태(State)가 변경되면 자동으로 UI를 업데이트하며, Virtual DOM을 통해 효율적인 렌더링을 수행합니다.

### 핵심 철학: 선언형 vs 명령형

#### 명령형 프로그래밍 (Android)

"어떻게(How)" UI를 변경할지 **단계별로 명령**합니다.

```kotlin
// Android (명령형)
val textView = findViewById<TextView>(R.id.title)
textView.text = "Hello"
textView.setTextColor(Color.BLUE)
textView.textSize = 16f

button.setOnClickListener {
    textView.text = "Clicked!" // UI를 직접 변경
}
```

**문제점**:

- 상태가 복잡해지면 UI 동기화가 어려움
- 버그 발생 시 추적이 힘듦 (어디서 textView를 변경했는지?)
- 테스트가 어려움

#### 선언형 프로그래밍 (React)

"무엇을(What)" 보여줄지 **상태에 따라 선언**합니다.

```jsx
// React (선언형)
function Greeting() {
  const [text, setText] = useState("Hello");

  return (
    <View>
      <Text style={{ color: "blue", fontSize: 16 }}>{text}</Text>
      <Button onPress={() => setText("Clicked!")} />
    </View>
  );
}
```

**장점**:

- 상태(text)만 변경하면 UI는 자동으로 업데이트
- UI는 항상 현재 상태를 반영 (UI = f(state))
- 테스트가 쉬움 (상태만 변경해서 확인)

### UI = f(state) 공식

React의 핵심을 수식으로 표현하면:

```
UI = f(state)
```

- **state**: 현재 데이터 상태
- **f**: React 컴포넌트 (순수 함수)
- **UI**: 렌더링 결과

같은 state면 항상 같은 UI가 나옵니다 (예측 가능성).

---

## 2. 왜 React를 배워야 하나?

### Native 개발자 관점에서의 이점

#### 1. 생산성 향상

```kotlin
// Android: 각 View마다 수동 업데이트
class CounterActivity : AppCompatActivity() {
    var count = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        updateUI() // 초기 렌더링
    }

    fun increment() {
        count++
        updateUI() // 수동 업데이트
    }

    fun updateUI() {
        findViewById<TextView>(R.id.count).text = "Count: $count"
        findViewById<TextView>(R.id.double).text = "Double: ${count * 2}"
        findViewById<TextView>(R.id.square).text = "Square: ${count * count}"
    }
}
```

```jsx
// React: 상태만 변경
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <View>
      <Text>Count: {count}</Text>
      <Text>Double: {count * 2}</Text>
      <Text>Square: {count * count}</Text>
      <Button onPress={() => setCount(count + 1)} />
    </View>
  );
  // UI는 자동으로 업데이트!
}
```

#### 2. 크로스 플랫폼

- **React Native**: iOS + Android 동시 개발
- **React Native Web**: Web까지 확장 가능
- 하나의 코드베이스로 여러 플랫폼 지원

#### 3. 컴포넌트 재사용

```jsx
// 한 번 만들면 어디서든 재사용
function UserCard({ name, avatar }) {
  return (
    <View>
      <Image source={{ uri: avatar }} />
      <Text>{name}</Text>
    </View>
  );
}

// 사용
<UserCard name="철수" avatar="..." />
<UserCard name="영희" avatar="..." />
```

#### 4. 거대한 생태계

- npm 패키지: 수백만 개
- React Native 라이브러리: Navigation, Animation, UI 등
- 활발한 커뮤니티

---

## 3. JSX (JavaScript XML)

### JSX란?

JavaScript 안에 HTML처럼 보이는 문법을 작성할 수 있게 해주는 **문법 확장**입니다.

```jsx
const element = <h1>Hello, World!</h1>;
```

**중요**: JSX는 JavaScript가 아닙니다! **Babel**이 JavaScript로 변환합니다.

### JSX → JavaScript 변환

```jsx
// JSX 작성
const element = <h1 className="title">Hello</h1>;

// Babel이 변환
const element = React.createElement("h1", { className: "title" }, "Hello");
```

### React.createElement 분석

```javascript
function createElement(type, props, ...children) {
  return {
    type, // 'h1', 'div' 등
    props: {
      ...props,
      children, // 자식 요소들
    },
  };
}
```

결과는 **JavaScript 객체** (Virtual DOM 노드):

```javascript
{
  type: 'h1',
  props: {
    className: 'title',
    children: 'Hello'
  }
}
```

### Android XML과의 비교

#### Android (정적 XML)

```xml
<LinearLayout
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/title"
        android:text="Hello"
        android:textColor="#FF0000" />
</LinearLayout>
```

**특징**:

- 정적: 조건문, 반복문 사용 불가
- 런타임에 inflate 필요
- findViewById로 참조

#### React (동적 JSX)

```jsx
<View style={{ flexDirection: "column" }}>
  <Text style={{ color: "#FF0000" }}>{title}</Text>
</View>
```

**특징**:

- 동적: JavaScript 표현식 사용 가능
- 조건문, 반복문 자유롭게 사용
- 직접 참조 가능 (ref 사용)

### JSX 핵심 규칙

#### 1. 반드시 하나의 부모 요소 반환

```jsx
// ❌ 에러: Adjacent JSX elements must be wrapped
function Component() {
  return (
    <Text>First</Text>
    <Text>Second</Text>
  );
}

// ✅ Fragment 사용
function Component() {
  return (
    <>
      <Text>First</Text>
      <Text>Second</Text>
    </>
  );
}

// 또는
function Component() {
  return (
    <React.Fragment>
      <Text>First</Text>
      <Text>Second</Text>
    </React.Fragment>
  );
}
```

**이유**: `React.createElement`는 하나의 객체만 반환하기 때문!

#### 2. JavaScript 표현식은 `{}` 안에

```jsx
const name = "철수";
const age = 25;

return (
  <View>
    <Text>이름: {name}</Text>
    <Text>나이: {age}</Text>
    <Text>내년 나이: {age + 1}</Text>
    <Text>{age >= 20 ? "성인" : "미성년자"}</Text>
  </View>
);
```

**주의**: `{}` 안에는 **표현식(expression)**만 가능!

```jsx
// ❌ 안됨: if는 statement
{if (condition) { ... }}

// ✅ 삼항 연산자 사용
{condition ? <Text>True</Text> : <Text>False</Text>}

// ✅ 또는 && 연산자
{condition && <Text>True</Text>}
```

#### 3. 스타일은 객체로

```jsx
// Android
android:textColor="#FF0000"
android:textSize="16sp"

// React Native
<Text style={{
  color: '#FF0000',
  fontSize: 16,
}}>
  Hello
</Text>

// 또는 분리
const styles = {
  text: {
    color: '#FF0000',
    fontSize: 16,
  }
};

<Text style={styles.text}>Hello</Text>
```

#### 4. 예약어 주의

```jsx
// ❌ 안됨: class는 JavaScript 예약어
<div class="container">

// ✅ className 사용 (Web)
<div className="container">

// React Native는 style만 사용
<View style={styles.container}>
```

---

## 4. 컴포넌트

### 컴포넌트란?

재사용 가능한 UI 조각. React 앱은 **컴포넌트의 트리**로 구성됩니다.

```
App
├─ Header
│  ├─ Logo
│  └─ Navigation
├─ Content
│  ├─ Sidebar
│  └─ MainContent
└─ Footer
```

### 함수 컴포넌트 (권장)

```jsx
function Greeting(props) {
  return <Text>Hello, {props.name}!</Text>;
}

// 또는 Arrow Function
const Greeting = (props) => {
  return <Text>Hello, {props.name}!</Text>;
};

// 사용
<Greeting name="철수" />;
```

### 클래스 컴포넌트 (레거시)

```jsx
class Greeting extends React.Component {
  render() {
    return <Text>Hello, {this.props.name}!</Text>;
  }
}
```

**참고**: 현대 React는 **함수 컴포넌트 + Hooks**를 권장합니다.

### Android와 비교

```kotlin
// Android: Custom View
class GreetingView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : TextView(context, attrs) {

    fun setName(name: String) {
        text = "Hello, $name!"
    }
}
```

```jsx
// React: 함수형 컴포넌트
function Greeting({ name }) {
  return <Text>Hello, {name}!</Text>;
}
```

### 컴포넌트 명명 규칙

- **반드시 대문자로 시작**: `Greeting`, `UserCard`
- 소문자는 HTML 태그로 인식: `<div>`, `<span>`

```jsx
// ❌ 안됨
function greeting() {
  return <Text>Hello</Text>;
}
<greeting />; // HTML <greeting> 태그로 인식

// ✅ 대문자로 시작
function Greeting() {
  return <Text>Hello</Text>;
}
<Greeting />;
```

---

## 5. JavaScript/TypeScript 필수 문법

### const, let (var 사용 금지)

```javascript
const name = "철수"; // 재할당 불가
let age = 25; // 재할당 가능

name = "영희"; // ❌ TypeError
age = 26; // ✅
```

### Arrow Function

```javascript
// 전통적 함수
function add(a, b) {
  return a + b;
}

// Arrow Function
const add = (a, b) => a + b;

// 중괄호 있으면 return 명시
const add = (a, b) => {
  console.log("Adding...");
  return a + b;
};
```

**주의**: `this` 바인딩이 다릅니다!

```javascript
// 전통적 함수: this가 동적
function Counter() {
  this.count = 0;
  setTimeout(function () {
    this.count++; // ❌ this는 window
  }, 1000);
}

// Arrow Function: this가 렉시컬
function Counter() {
  this.count = 0;
  setTimeout(() => {
    this.count++; // ✅ this는 Counter
  }, 1000);
}
```

### 구조 분해 (Destructuring)

```javascript
// 객체
const user = { name: "철수", age: 25, city: "서울" };
const { name, age } = user;
console.log(name); // "철수"

// 배열
const colors = ["red", "green", "blue"];
const [first, second] = colors;
console.log(first); // "red"

// React에서 자주 사용
const [count, setCount] = useState(0);
const { name, age } = props;
```

### Spread/Rest 연산자

```javascript
// Spread: 배열/객체 펼치기
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5]; // [1, 2, 3, 4, 5]

const user = { name: "철수", age: 25 };
const updatedUser = { ...user, age: 26 }; // { name: "철수", age: 26 }

// Rest: 나머지 수집
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b);
}
sum(1, 2, 3, 4); // 10
```

**React에서의 활용**:

```jsx
// Props spreading
const props = { name: "철수", age: 25 };
<Greeting {...props} />;

// State 업데이트 (불변성 유지)
setUser({ ...user, age: 26 });
```

### Template Literal

```javascript
const name = "철수";
const age = 25;

// 기존 방식
const message = "안녕하세요, " + name + "입니다. 나이는 " + age + "살입니다.";

// Template Literal
const message = `안녕하세요, ${name}입니다. 나이는 ${age}살입니다.`;

// 여러 줄
const html = `
  <div>
    <h1>${title}</h1>
    <p>${content}</p>
  </div>
`;
```

### Optional Chaining & Nullish Coalescing

```javascript
const user = {
  profile: {
    name: "철수",
  },
};

// 기존 방식: 번거로움
const name = user && user.profile && user.profile.name;

// Optional Chaining
const name = user?.profile?.name; // "철수"
const email = user?.profile?.email; // undefined (에러 안남)

// Nullish Coalescing: null/undefined만 체크
const age = user?.age ?? 18; // user.age가 0이어도 0 반환
const age = user?.age || 18; // user.age가 0이면 18 반환 (falsy 체크)
```

**Kotlin 비교**:

```kotlin
val name = user?.profile?.name  // Optional Chaining
val age = user?.age ?: 18       // Elvis Operator
```

---

## 6. 정리

### 핵심 개념

1. **선언형 UI**: UI = f(state)
2. **JSX**: JavaScript + XML-like syntax
3. **컴포넌트**: 재사용 가능한 UI 단위
4. **함수형 프로그래밍**: 순수 함수, 불변성

### 다음 단계

- Props와 State를 통한 데이터 관리
- 컴포넌트 간 통신 방법
- 렌더링 최적화 기초

**다음**: [02-Props-State.md](./02-Props-State.md)
