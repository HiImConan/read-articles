# 무지성 useEffect 남발을 멈춰!

> 출처 : https://velog.io/@jay/you-might-need-useEffect-diet#react-18--strict-mode

1. useEffect Hook은 언제 사용하나요?
   useEffect는 컴포넌트의 상태 값이 변화할 때마다 특정 동작을 수행시키고 싶을 때 사용하는 Hook이다. deps에 어떤 값이 들어가는지에 따라 컴포넌트 생성 후 상태 값의 변화에 맞춰 수행하거나 딱 한 번만 수행할 수 있게 할 수 있다.

2. React18버전 이상부터는 useEffect 렌더링이 두 번 일어남(+ StrictMode 활성화 시)
<aside>
💡 왜 Why? :  *리엑트18은 페이지 이동 이후 다시 돌아왔을때 앱이 망가지는 부분이 없는지 확인하기 위해 개발모드(process.env.NODE_ENV === development)에서 한 컴포넌트를 두번 렌더링합니다. 따라서 useEffect가 두번 호출되어 위와 같이 Connecting이 두 번씩 기록되는 것인데요,*
</aside>

3. 그렇기 때문에 생기는 비효율도 많다.

- data fetching을 비롯한 useEffect 내부 코드들이 전부 두 번씩 실행됨
- 서버 비용 증가는 물론, dependency array에 삽입된 state가 바뀔 때마다 리렌더링은 곱절로 늘어남
- 만약 서로 다른 두 data fetching이 발생했을 때 메타데이터 간의 순서를 보장할 수 없음(race condition)

4. 그러면 StrictMode를 꺼놓으면 되는거 아닌가요?

- ㄴㄴ. StrictMode를 없애 두 번씩 렌더링하는 과정이 일어나지 않게 할 수 있지만 production 환경에서 일어날 수 있는 오류를 리엑트에서 잡아주지 못하므로 되도록이면 StrictMode에서 개발하는 것이 좋음.

<br/>

그럼 해결 방법은 뭔가요?

1. cleanup 함수를 꼭 작성해주기

   - cleanup 함수가 뭔가요?

     React의 useEffect Hook은 class 생명주기 메서드에서 `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` 세 가지 시점에 특정 side effect를 실행시키기 위해 조합된 훅이다([React useEffect 공식문서](https://ko.reactjs.org/docs/hooks-effect.html) 내용이 완벽하니 꼭 한 번씩 읽어볼 것).

     이 때, Component의 unmount 이전 / update 직전에 어떠한 작업을 수행하고 싶다면 `Clean-up 함수`를 반환해 주어야 한다.

     Clean-up 함수를 사용하게 되면 수행 순서는 `re-render -> 이전 effect clean-up -> effect`
     이다.

   ***

   <br/>

   ### [설명: effect가 업데이트 시마다 실행되는 이유](https://ko.reactjs.org/docs/hooks-effect.html#:~:text=%EC%84%A4%EB%AA%85%3A%20effect%EA%B0%80%20%EC%97%85%EB%8D%B0%EC%9D%B4%ED%8A%B8%20%EC%8B%9C%EB%A7%88%EB%8B%A4%20%EC%8B%A4%ED%96%89%EB%90%98%EB%8A%94%20%EC%9D%B4%EC%9C%A0)

   친구가 온라인인지 아닌지 표시하는 `FriendStatus` 컴포넌트 예시를 생각해봅시다. class는 `this.props`로부터 `friend.id`를 읽어내고 컴포넌트가 마운트된 이후에 친구의 상태를 구독하며 컴포넌트가 마운트를 해제할 때에 구독을 해지합니다.

   ```jsx
     componentDidMount() {
       ChatAPI.subscribeToFriendStatus(
         this.props.friend.id,
         this.handleStatusChange
       );
     }

     componentWillUnmount() {
       ChatAPI.unsubscribeFromFriendStatus(
         this.props.friend.id,
         this.handleStatusChange
       );
     }
   ```

   **그런데 컴포넌트가 화면에 표시되어있는 동안 `friend` prop이 변한다면** 무슨 일이 일어날까요? 컴포넌트는 다른 친구의 온라인 상태를 계속 표시할 것입니다. 버그인 거죠. 또한 마운트 해제가 일어날 동안에는 구독 해지 호출이 다른 친구 ID를 사용하여 메모리 누수나 충돌이 발생할 수도 있습니다.

   클래스 컴포넌트에서는 이런 경우들을 다루기 위해 `componentDidUpdate`를 사용합니다.

   ```jsx
     componentDidMount() {
       ChatAPI.subscribeToFriendStatus(
         this.props.friend.id,
         this.handleStatusChange
       );
     }

     componentDidUpdate(prevProps) {
       // 이전 friend.id에서 구독을 해지합니다.
       ChatAPI.unsubscribeFromFriendStatus(
         prevProps.friend.id,
         this.handleStatusChange
       );
       // 다음 friend.id를 구독합니다.
       ChatAPI.subscribeToFriendStatus(
         this.props.friend.id,
         this.handleStatusChange
       );
     }
   	componentWillUnmount() {
       ChatAPI.unsubscribeFromFriendStatus(
         this.props.friend.id,
         this.handleStatusChange
       );
     }
   ```

   React 애플리케이션의 흔한 버그 중의 하나가 `componentDidUpdate`를 제대로 다루지 않는 것입니다.

   이번에는 Hook을 사용하는 컴포넌트를 생각해봅시다.

   ```jsx
   function FriendStatus(props) {
     // ...
     useEffect(() => {
       // ...
       ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
       return () => {
         ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
       };
     });
   ```

   이 경우에는 버그에 시달리지 않습니다.(달리 바꾼 것도 없는데 말이죠.)

   `useEffect`가 *기본적으로* 업데이트를 다루기 때문에 더는 업데이트를 위한 특별한 코드가 필요 없습니다. 다음의 effect를 적용하기 전에 이전의 effect는 정리(clean-up)합니다.

   ***

   위 예시처럼 컴포넌트가 마운트 해제되는 순간 뿐만 아니라 리렌더링되는 모든 순간에도 cleanup함수가 사용되며, cleanup 함수를 잘 활용해야 사용자가 리렌더링이 두 번 일어나는지 느끼지 못하게 된다.

2. props, state 변경에 따라 또 다른 state를 업데이트해야 하는 경우

   ```jsx
   function TodoList({ todos, filter }) {
     const [newTodo, setNewTodo] = useState("");

     // 🔴 Avoid: redundant state and unnecessary Effect
     const [visibleTodos, setVisibleTodos] = useState([]);
     useEffect(() => {
       setVisibleTodos(getFilteredTodos(todos, filter));
     }, [todos, filter]);
     // ...
   }
   ```

   위 코드에서 todos, filter 둘 중 하나의 데이터만 변경되더라도 ui가 업데이트되어야 하기 때문에 visibleTodos state는 변경되어야 한다. 그래서 useEffect를 사용하는 것은 문제가 없어보인다.

   그러나 이는 불필요한 리렌더링을 발생시킨다. 상태 변경이 일어나면 리엑트는 돔에 변경된 state를 commit하고 그 이후에 ui를 업데이트 한다. 위 경우에는 todos(or filter) 데이터가 변경 → 리렌더링, getFilteredTodos 변경 → 리렌더링, setVisibleTodos 변경 → 리렌더링 총 3번의 리렌더링이 발생하게 되며 불필요한 렌더링이 총 2(3-1)x2번 일어나는 것이다.

   ```jsx
   import { useMemo, useState } from "react";

   function TodoList({ todos, filter }) {
     const [newTodo, setNewTodo] = useState("");
     // ✅ Does not re-run getFilteredTodos() unless todos or filter change
     const visibleTodos = useMemo(
       () => getFilteredTodos(todos, filter),
       [todos, filter]
     );
     // ...
   }
   ```

   위 코드는 ui 업데이트를 위해 `setVisibleTodos`를 호출해주지 않아도 컴포넌트가 리렌더링 될 때마다 ui가 의존하고 있는 visibleTodos가 업데이트 되기 때문에 ui는 최신 데이터 반영을 보장할 수 있다.

3. props 변경에 따라 상태가 리셋되어야 하는 경우

   ```jsx
   export default function ProfilePage({ userId }) {
     const [comment, setComment] = useState("");

     // 🔴 Avoid: Resetting state on prop change in an Effect
     useEffect(() => {
       setComment("");
     }, [userId]);
     // ...
   }
   ```

   관리자 페이지에서 유저들에 대한 코멘트를 작성한다고 가정, 유저 1에 대한 정보를 작성하다 유저 2로 이동할 경우 위 처럼 comment state를 명시적으로 useEffect 내부에서 리셋해주는 방식을 떠올릴 수 있다. 그러나 이 방식은 마찬가지로 userId가 변경될 때 한 번, setComment가 실행된 이후 한 번 총 두 번의 리렌더링이 일어나는 비효율적인 방법이다.

   아래와 같이 컴포넌트 내부 useEffect 사용이 아닌 페이지단에서 컴포넌트 자체를 리셋하는 방식으로 접근한다면 더 효율적인 리엑트 사용이 가능할 것이다.

   ```jsx
   export default function ProfilePage({ userId }) {
     return <Profile userId={userId} key={userId} />;
   }

   function Profile({ userId }) {
     // ✅ This and any other state below will reset on key change automatically
     const [comment, setComment] = useState("");
     // ...
   }
   ```

4. props 변경에 따라 특정 상태만 업데이트 해야 하는 경우
   ```jsx
   function List({ items }) {
     const [isReverse, setIsReverse] = useState(false);
     const [selection, setSelection] = useState(null);

     useEffect(() => {
       setSelection(null);
     }, [items]);
     // ...
   }
   ```
   위 코드의 경우 아래와 같이 바꿀 수 있다.
   ```jsx
   function List({ items }) {
     const [isReverse, setIsReverse] = useState(false);
     const [selection, setSelection] = useState(null);
     // Better: Adjust the state while rendering
     const [prevItems, setPrevItems] = useState(items);
     if (items !== prevItems) {
       setPrevItems(items);
       setSelection(null);
     }
     // ...
   }
   ```
   useEffect 대신 useState와 조건문을 사용하여 state를 set시켜주는 방법이다. 두 방법의 차이로는 우선 useEffect의 태생적 실행 순서가 있다. useEffect는 컴포넌트의 렌더링이 끝난 뒤 effect가 실행된다. 아래 방법은 컴포넌트의 렌더링 과정 중에 update가 이루어진다. useEffect로 인한 불필요한 리렌더링으로 낭비되는 메모리보다 훨씬 적은 메모리 낭비를 꾀할 수 있다.
5. data fetching의 경우
   ```jsx
   useEffect(() => {
     let ignore = false;

     async function startFetching() {
       const json = await fetchTodos(userId);
       if (!ignore) {
         setTodos(json);
       }
     }

     startFetching();

     return () => {
       ignore = true;
     };
   }, [userId]);
   ```
   위 코드와 같이 clean-up 함수를 통해 data fetching 이후 수행될 effect의 불필요한 반복을 토글할 수 있는 2진 변수를 활용하는 방법이 있다.
   data fetching 횟수가 적지 않을 경우, 또는 데이터의 캐싱이 필요할 경우에는 `react-query`와 같은 data fetch 최적화 라이브러리를 활용하는 것이 좋다.
