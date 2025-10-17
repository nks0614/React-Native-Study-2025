# Phase 1: React 기초와 내부 동작 원리

## 📌 학습 목표

Native 개발자(Android/iOS)가 React의 철학과 동작 원리를 **깊이 있게** 이해하고, 단순히 사용법을 넘어 "왜 이렇게 동작하는지"를 파악하는 것이 목표입니다.

## 🗂️ 학습 구조

### 1. [React 기초](./01-React-기초.md)

- React의 철학: 선언형 vs 명령형
- JavaScript/TypeScript 최신 문법 (ES6+)
- JSX와 컴포넌트의 본질
- Native 개발과의 비교 분석

### 2. [Props와 State](./02-Props-State.md)

- Props: 컴포넌트 간 데이터 전달
- State: 컴포넌트 내부 상태 관리
- 불변성(Immutability)의 중요성
- Native 개발 패턴과의 비교

### 3. [Hooks 기초](./03-Hooks-기초.md)

- Hooks가 등장한 배경
- useState, useEffect 기본 사용법
- Hooks의 규칙과 그 이유
- Class Component vs Hooks

### 4. [Hooks 내부 구조 Deep Dive](./04-Hooks-내부구조.md)

- React 소스코드 분석
- useState가 실제로 동작하는 방식
- Fiber 아키텍처 기초
- Dispatcher 메커니즘
- 직접 구현해보는 간단한 Hooks

### 5. [Virtual DOM과 Reconciliation](./05-Virtual-DOM-Reconciliation.md)

- Virtual DOM이 필요한 이유
- Diffing 알고리즘
- Reconciliation 과정
- 성능 최적화 기법
- React가 리렌더링을 최소화하는 방법

### 6. [실습 프로젝트](./06-실습-프로젝트.md)

- Todo 앱 만들기 (기초)
- 카운터 앱 (State 관리)
- 사용자 목록 (List 렌더링)
- 코드베이스 분석 실습

## 🎯 학습 접근 방식

### 이론 학습 (40%)

- React의 핵심 개념 이해
- 왜 이런 설계가 되었는지 철학 파악

### 코드베이스 분석 (40%)

- React 소스코드 직접 읽기
- 디버거로 실제 동작 확인
- 중요 함수 flow 추적

### 실습 (20%)

- 간단한 컴포넌트 직접 구현
- Hooks를 직접 만들어보기
- 리렌더링 최적화 실습

## 📚 Native 개발자를 위한 비유표

| React 개념  | Android              | iOS                     | 설명                    |
| ----------- | -------------------- | ----------------------- | ----------------------- |
| Component   | View/Fragment        | UIView/UIViewController | UI의 재사용 가능한 단위 |
| Props       | Bundle/Intent extras | init 파라미터           | 부모→자식 데이터 전달   |
| State       | ViewModel/LiveData   | @State/@Observable      | 컴포넌트 내부 상태      |
| useEffect   | Lifecycle callbacks  | viewDidLoad/onAppear    | 부수 효과 처리          |
| Virtual DOM | View hierarchy diff  | -                       | 효율적인 UI 업데이트    |
| Hooks       | -                    | SwiftUI Modifiers       | 로직 재사용 메커니즘    |

## ⚡ 선수 지식

- JavaScript 기본 문법 (변수, 함수, 배열, 객체)
- ES6+ 문법 (화살표 함수, 구조 분해, Spread 연산자)
- 비동기 처리 기본 개념 (Promise, async/await)

> 💡 **Tip**: 각 섹션을 순서대로 학습하되, 코드베이스 분석은 VS Code에서 실제로 React 소스코드를 열어보며 따라하는 것을 추천합니다!

## 🔗 다음 단계

Phase 1을 완료하면:

- React의 기본 동작 원리를 이해하게 됩니다
- Hooks를 자신있게 사용할 수 있습니다
- 렌더링 최적화를 위한 기초 지식을 갖추게 됩니다

**다음**: [01-React-기초.md](./01-React-기초.md)로 시작하세요!
