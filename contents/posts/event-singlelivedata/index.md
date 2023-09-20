---
title: "LiveData로 이벤트 처리하기: SingleLiveData"
description: "페스타고 앱의 이벤트 처리에 대해 설명합니다."
date: 2023-09-19
update: 2023-09-19
tags:
- Android
- Event
- SingleLiveData
---

안녕하세요. 페스타고의 해시입니다.

페스타고 앱을 개발하면서 다른 액티비티로 넘어가거나 토스트를 띄우는 등의 이벤트를 어떻게 처리할지를 고민하게 되었고, 이러한 이벤트를 처리하기 위해 시도했던 방법과 겪었던 문제점, 그리고 해결 방법에 대해 다루려 합니다.

---

먼저 어떤 상황에서 이벤트가 발생하는지 코드를 통해 살펴보자.

뷰모델에서 로그인 여부를 확인해 로그인된 사용자가 아닌 경우 SignInActivity로 이동한다.

```kotlin
// MyPageViewModel

fun loadUserInfo() {
    if (!authRepository.isSigned) {
        // SignInActivity로 이동!
    }
    viewModelScope.launch {
        // 유저 정보 불러오기
    }
}
```

사용자가 마이페이지에서 회원 탈퇴를 한 경우 첫 화면으로 이동해 앱의 첫 화면에서 재시작 하도록 한다.

```kotlin
// MyPageViewModel

fun deleteAccount() {
    viewModelScope.launch {
        authRepository.deleteAccount()
            .onSuccess {
                // 첫 화면으로 이동
            }.onFailure {
                // ...
            }
    }
}
```

위 두 예시처럼 뷰모델에서 이벤트가 발생하는 경우가 있다. 이벤트는 뷰모델의 메서드에서 발생하지만 다른 액티비티를 열거나 다이얼로그를 띄우는 등 이벤트에 대한 처리는 액티비티나 프래그먼트에서 이루어져야 한다.

이런 이벤트를 처리하기 위해 LiveData를 활용할 수 있다. Flow를 사용해 뷰모델에서 안드로이드 의존성을 제거하는 것도 좋은 방법이지만 Flow는 다음 포스트에서 다루도록 하고, 이번 포스트에서는 LiveData를 다루려 한다.

## LiveData로 이벤트 처리하기

뷰모델에 라이브데이터를 두고, 액티비티나 프래그먼트에서 이를 observe 하여 이벤트를 처리할 수 있다.

```kotlin
class MyPageViewModel(
    // ...
) : ViewModel() {

    private val _showSignInEvent = MutableLiveData<Boolean>()
    val showSignInEvent: LiveData<Boolean> get() = _showSignInEvent // 1

		// ...

		fun showSignIn() { // 3
        _showSignInEvent.value = true // 4
    }
}
```

```kotlin
// MyPageFragment

vm.showSignInEvent.observe(viewLifecycleOwner) { event -> // 2
    if (event) {
        startActivity(SignInActivity.getIntent(requireContext())) // 5
    }
}
```

이벤트가 처리되는 플로우를 나열해 보면 다음과 같다.

1. 뷰모델에 LiveData<Boolean> 타입의 showSignInEvent가 있다.
2. 프래그먼트에서 showSignInEvent를 observe 한다.
3. 뷰모델의 showSignIn()이 호출된다.
4. showSignInEvent의 값이 true로 변경된다.
5. showSignInEvent 값이 true로 변경되어 이를 observe 하는 프래그먼트에서 startActivity()가 실행된다.

### 문제점

그냥 LiveData로 이벤트를 처리하면 이벤트가 중복으로 발생되는 문제가 있다. 액티비티 이동은 이벤트가 발생했을 때 한 번 이루어져야 하는데 showSignInEvent를 observe할 때마다 이전 이벤트가 중복으로 발생하는 것이다.

예를 들어, 토스트를 띄우는 이벤트가 발생했을 때 화면 회전하는 경우를 생각해 보자.

1. LiveData를 observe한다.
2. LiveData 값이 변경되면 LiveData는 옵저버에게 데이터를 emit하여 토스트가 뜬다.
3. 화면 회전한다.
4. 화면 회전에 의해 라이프사이클이 돌면서 액티비티가 재생성되고 LiveData를 다시 observe한다.
5. LiveData는 observe할 때 마지막 값을 emit하여 토스트가 다시 뜬다.

### ❌ 해결 1: LiveData의 값 되돌리기 (1)

이벤트 발생 후에 LiveData의 value를 변경하는 식으로 중복 이벤트를 방지할 수 있다.

```kotlin
fun showSignIn() { 
    _showSignInEvent.value = true
    _showSignInEvent.value = false
}
```

하지만 이벤트 발생 후 값을 변경해 주는 것은 번거롭기도 하고, 개발자의 실수로 처리를 하지 않는 경우가 발생하기도 쉽다. 또한 LiveData는 값이 변경되었을 때 모든 경우에 emit 하지 않는다. showSignInEvent 값이 true로 변경되어도 활성 상태의 옵저버가 없다면 변경되었다는 것을 누구에게도 알리지 않고 다시 false로 바뀌게 될 수 있다.

### ❌ 해결 2: LiveData의 값 되돌리기 (2)

showSignInEvent 값을 false로 되돌리는 함수를 뷰모델에 추가하고, startActivity() 이후에 호출하는 방법도 있을 것이다. 하지만 이 방식도 개발자의 실수로 LiveData값을 되돌리지 않는 경우가 생길 수 있다. 게다가 한 화면에 여러 이벤트가 존재한다면 각 이벤트마다 함수를 만들어야한다. (보일러플레이트!)

### ⭕️ 해결 3: Event Wrapper

이를 해결하기 위해 많이 사용되는 방법은 Event Wrapper를 사용하는 것이다. 안드로이드 샘플에서도 사용하는 널리 쓰이는 방식으로, 값을 Event로 감싸 이벤트를 한 번만 handle 하도록 한다.

```kotlin
/**
 * Used as a wrapper for data that is exposed via a LiveData that represents an event.
 */
open class Event<out T>(private val content: T) {

    var hasBeenHandled = false
        private set // Allow external read but not write

    /**
     * Returns the content and prevents its use again.
     */
    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) {
            null
        } else {
            hasBeenHandled = true
            content
        }
    }

    /**
     * Returns the content, even if it's already been handled.
     */
    fun peekContent(): T = content
}
```

showSignInEvent는 이제 LiveData<Event<Unit>> 타입이 된다.

```kotlin
class MyPageViewModel(
    // ...
) : ViewModel() {

    private val _showSignInEvent = MutableLiveData<Event<Unit>>()
    val showSignInEvent: LiveData<Event<Unit>> get() = _showSignInEvent

		// ...

		fun showSignIn() {
        _showSignInEvent.value = Event(Unit)
    }
}
```

이벤트가 발생한 후에는 getContentIfNotHandled()가 null을 리턴하므로 아래 코드처럼 처리하면 startActivity()가 한번 실행되도록 할 수 있다.

```kotlin
// MyPageFragment

vm.showSignInEvent.observe(viewLifecycleOwner) { event ->
    event.getContentIfNotHandled()?.let {
        startActivity(SignInActivity.getIntent(requireContext()))
    }
}
```

### 🚀 개선 1: SingleLiveData

위에 Event Wrapper 코드에서 확인할 수 있듯이 타입을 모두 Event로 감싸고, 이벤트를 생성할 때마다 Event로 감싸서 처리하는 것은 매우 귀찮은 일이다. 이를 개선하기 위해 SingleLiveData 내부에서 이를 처리한다.

```kotlin
abstract class SingleLiveData<T> {

    private val liveData = MutableLiveData<Event<T>>()

    protected constructor()

    protected constructor(value: T) {
        liveData.value = Event(value)
    }

    protected open fun setValue(value: T) {
        liveData.value = Event(value)
    }

    protected open fun postValue(value: T) {
        liveData.postValue(Event(value))
    }

    fun getValue() = liveData.value?.peekContent()

    fun observe(owner: LifecycleOwner, onResult: (T) -> Unit) {
        liveData.observe(owner) { it.getContentIfNotHandled()?.let(onResult) }
    }

    fun observePeek(owner: LifecycleOwner, onResult: (T) -> Unit) {
        liveData.observe(owner) { onResult(it.peekContent()) }
    }
}
```

```kotlin
class MutableSingleLiveData<T> : SingleLiveData<T> {

    constructor() : super()

    constructor(value: T) : super(value)

    public override fun postValue(value: T) {
        super.postValue(value)
    }

    public override fun setValue(value: T) {
        super.setValue(value)
    }
}
```

이제 SingleLiveData를 LiveData와 유사하게 사용하면서 이벤트가 한 번만 처리되도록 할 수 있다!

```kotlin
class MyPageViewModel(
    // ...
) : ViewModel() {

    private val _showSignInEvent = MutableSingleLiveData<Unit>()
    val showSignInEvent: SingleLiveData<Unit> get() = _showSignInEvent

		// ...

		fun showSignIn() {
        _showSignInEvent.setValue(Unit)
    }
}
```

```kotlin
vm.showSignInEvent.observe(viewLifecycleOwner) {
    startActivity(SignInActivity.getIntent(requireContext()))
}
```

### 🚀 개선 2: sealed interface 활용하기

페스타고 앱의 마이페이지에서는 5개의 이벤트가 발생할 수 있다.

1. 로그인 화면으로 이동 (ShowSignIn)
2. 로그아웃 성공 (SignOutSuccess)
3. 회원 탈퇴 성공 (DeleteAccountSuccess)
4. 과거 예매 내역 보기 (ShowTicketHistory)
5. 정말 탈퇴하시겠어요? 다이얼로그 띄우기 (ShowConfirmDelete)

위에서 살펴본 SingleLiveData를 사용해 뷰모델에 각 이벤트를 선언하면 어떻게 될까? MyPageViewModel에 Mutable 값을 포함해서 10개 프로퍼티가 추가될 것이다. MyPageFragment에서는 5개의 이벤트를 모두 따로 observe 해야 한다.

화면에서 발생할 수 있는 이벤트들을 sealed interface로 묶어보자. 이제 뷰모델에서 _event, event 두 개의 프로퍼티로 화면 내 모든 이벤트를 관리할 수 있다.

```kotlin
sealed interface MyPageEvent {
    object ShowSignIn : MyPageEvent
    object SignOutSuccess : MyPageEvent
    object DeleteAccountSuccess : MyPageEvent
    object ShowTicketHistory : MyPageEvent
    object ShowConfirmDelete : MyPageEvent
}
```

```kotlin
class MyPageViewModel(
    // ...
) : ViewModel() {

    private val _event = MutableSingleLiveData<MyPageEvent>()
    val event: SingleLiveData<MyPageEvent> = _event

}
```

이 글의 맨 처음에 언급했던 예시를 다시 떠올려보자.

- 뷰모델에서 로그인 여부를 확인해 로그인된 사용자가 아닌 경우 SignInActivity로 이동한다.
- 사용자가 마이페이지에서 회원 탈퇴를 한 경우 첫 화면으로 이동해 앱의 첫 화면에서 재시작 하도록 한다.

두 예시는 다음과 같이 처리할 수 있다.

```kotlin
class MyPageViewModel(
    // ...
) : ViewModel() {

    private val _event = MutableSingleLiveData<MyPageEvent>()
    val event: SingleLiveData<MyPageEvent> = _event

		fun loadUserInfo() {
        if (!authRepository.isSigned) {
            _event.setValue(MyPageEvent.ShowSignIn)
            // ...
        }
        viewModelScope.launch {
            // 유저 정보 불러오기
        }
    }

		fun deleteAccount() {
        viewModelScope.launch {
            authRepository.deleteAccount()
                .onSuccess {
                    _event.setValue(MyPageEvent.DeleteAccountSuccess)
                    // ...
                }.onFailure {
                    // ...
                }
        }
    }
}
```

MyPageFragment에서는 event 하나만 observe하고 event가 MyPageEvent 중 어느 것인지 분기하여 이벤트를 처리한다.

```kotlin
// MyPageFragment

vm.event.observe(viewLifecycleOwner) { event ->
    when (event) {
        is MyPageEvent.ShowSignIn -> handleShowSignInEvent()
        is MyPageEvent.SignOutSuccess -> handleSignOutSuccessEvent()
        is MyPageEvent.DeleteAccountSuccess -> handleDeleteAccountSuccess()
        is MyPageEvent.ShowTicketHistory -> handleShowTicketHistory()
        is MyPageEvent.ShowConfirmDelete -> handleShowConfirmDelete()
    }
}

private fun handleShowSignInEvent() {
    startActivity(SignInActivity.getIntent(requireContext()))
}

// ...
```

이벤트에 어떤 값을 전달하는 경우도 있을 수 있다. 예를 들어 페스타고 앱의 축제 목록 화면에서 축제 하나를 선택하면 티켓 예매 화면으로 넘어간다. 이때 축제 ID가 필요하다. 이런 이벤트는 다음과 같이 class로 이벤트를 정의할 수 있다.

```kotlin
sealed interface FestivalListEvent {
    class ShowTicketReserve(val festivalId: Long) : FestivalListEvent
}
```

```kotlin
// FestivalListViewModel

_event.setValue(ShowTicketReserve(festivalId))
```

```kotlin
// FestivalListFragment

private fun handleEvent(event: FestivalListEvent) {
    when (event) {
        is FestivalListEvent.ShowTicketReserve -> {
            startActivity(TicketReserveActivity.getIntent(requireContext(), event.festivalId))
        }
    }
}
```

**sealed interface 사용했을 때 장점**

- 각 이벤트가 모두 MyPageEvent를 상속하기에 MyPageEvent이라는 하나의 타입으로 관리할 수 있다.
- 각 이벤트를 뷰모델에 모두 따로 선언하고, 액티비티에서 각 이벤트를 observe 할 필요가 없다.
- MyPage에서 발생할 수 있는 이벤트를 한곳에서 확인할 수 있다는 장점도 있다. 일종의 명세 역할을 한다고 볼 수 있다.

**sealed interface 사용했을 때 단점**

- 일관성 있게 화면마다 Event sealed interface를 정의한다면 이벤트가 하나만 있어도 sealed interface를 따로 만들어야한다. 관점에 따라 보일러플레이트라고 생각할 수 있다.
