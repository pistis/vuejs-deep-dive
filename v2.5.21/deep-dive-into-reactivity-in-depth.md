> _이 문서는 [vue.js v2.5.21][git-1] 코드를 기반으로 작성되었습니다._  
> _이 문서는 [영문 공식가이드][guide-1]와 [한국어 공식가이드][guide-2]를 참고하였습니다._
---

시작하기 전에...
> [Build a reactivity system](https://www.vuemastery.com/courses/advanced-components/build-a-reactivity-system) <- Dependence와 Watcher 개념을 이용한 reactivity system 만들기  
> 이 문서를 모두 작성한 후 이후에 뒤늦게 발견한 비디오 영상이다. (OTL...역시 검색을 생활화 해야...)  
> 이 16분 짜리 영상을 보면 Dep과 Watcher가 무엇인지 기본적인 개념을 명확하게 이해할 수 있다.  
> 먼저 이 영상을 보고 이 문서를 읽는 것을 추천한다.  
> 물론, 이 문서에서는 이 내용을 vue.js 소스코드를 기반으로 살펴본다.   

# 프롤로그
## Reactivity in Depth
[한국어 공식가이드][guide-2]의 번역된 뜻으로는 "반응형에 대해 깊이 알아보기"이다.

해당 가이드에는 Vue.js가 어떻게 데이터의 변경을 추적하고 DOM을 업데이트 하는지에 대해서 설명한다.

공식가이드의 이미지를 잠깐 보자. 매우 심플하다.
![공식가이드_data][img-1]

이 그림을 보고 이런 의문들이 생겼다.
> 1. "Touch" 가 뭐지? getter에 화살표가 있는것을 보니 데이터를 참조하는 것인가?
> 2. Data에 getter, setter라는 무언가가 있는데 저게 뭘까?
> 3. Collect as Dependency? getter가 Watcher와 연결되어 있는데 어떤 형태로 연결되어 있을까?
> 4. Notify? 아 뭔가 데이터가 변경되면 Watcher라는 모듈에 데이터가 변경되었다고 알려준다는 뜻이겠구나..어떻게? 
> 5. 그리고 그다음에 Watcher가 컴포넌트의 render 함수를 직접 호출 하는건가? 데이터가 변경될때마다 rendering을 하는건 아니겠지?? 최적화는 하겠지? 근데 어떻게? Watcher가 어떻게 생겼는지 궁금하다.

위 의문들을 해결하기 위해 지금부터 vue.js 소스코드에 **deep dive** 해보자.

# Deep dive into 'Reactivity in depth'
## 예제로 시작하자
vue.js의 examples 폴더에 보면 예제들이 많이 있다.  
[examples/grid][git-2] 예제로 Vue를 탐험해보자.    

아래 그림은 예제가 실행된 모습이다.  
![grid-example-screenshot][grid-example-screenshot]

예제 코드가 길지 않으니 어떤 데이터가 있는지 하나씩 짚어보자.  

일단 UI상으로는 테이블에 Name과 Power 정보를 표현한다.
입력창 하나를 제공하고 키워드 입력시 간단한 검색(필터링) 기능을 제공한다.  
또한, 테이블 컬럼의 각 헤더를 클릭하면 정렬이 된다.  

반응형, 즉 데이터가 변경되었을때 자동으로 DOM에 반영되는 것을 확인할 수 있는 충분한 기능이 있다.  

이 화면을 어떻게 구성하는지 grid의 template을 보자.  
![examples grid html 1](http://drive.google.com/uc?export=view&id=1_GODhq2M_G-okG7f4e0A92TyJIblbxfi)
demo는 **Root Vue 컴포넌트**이며 form element와 **demo-grid 컴포넌트**를 가지고 있다.  
4줄 : input element는 **searchQuery** 데이터와 **v-model direvtice를 통해 양방향 연결**되어 있다.  
7~9줄 : demo-grid 컴포넌트에는 **gridData, gridColumns, searchQuery를 props로 전달**한다.  

demo Root 컴포넌트의 javascript 코드를 보자.  
![core instance index js](http://drive.google.com/uc?export=view&id=1nCcknrdX8oewLTg7-gvlHJ6fEe6eoJZX)
3줄 : 템플릿의 id '#demo'를 전달되는 options의 el로 설정한다.  
4줄 : 그리고 위 template에서 렌더링 할때 참조하는 data를 볼 수 있다. Primitive Type인 String, 그리고 Array와 Array의 요소로 Object를 가지고 있는 구조이다.  

이어서 사용자 정의된 demo-grid 컴포넌트의 template을 보자.  

![examples grid html 2](http://drive.google.com/uc?export=view&id=1IEMkRoFmvyiCrg3GZy7ulaLUX-4eXMBo)
4줄 : 루트 element인 table이 **filteredData 배열의 값이 없는 경우 렌더링 되지 않도록 v-if 처리**되어 있다. (뒤에서 filteredData.length가 0이되어 **table이 렌더링 되지 않을때 반응형은 어떻게 동작하는지 살펴보자.**)  

마지막으로 demo-grid 컴포넌트의 javascript 코드이다.  
![examples grid grid js](http://drive.google.com/uc?export=view&id=13cTBQsteXLHTkvB3DsVEEwzjMUq7ge8E)
2줄 : Vue.component로 demo-grid 전역 컴포넌트를 등록한다.  
8줄 : 이 컴포넌트는 input의 키워드를 받아 필터링 키로 사용하고  
49줄 : 테이블 헤더를 클릭하여 정렬 기준을 바꿀 수 있는 기능을 제공한다.  
10줄 : 여기서도 data 함수가 반환하는 값과  
21줄 : computed로 정의된 filteredData 값을 기억하자. 이 데이터들이 반응형 데이터가 된다.  

간단히 우리가 살펴볼 예제가 어떤 형태로 되어 있는지 살펴보았다.

## 라이프사이클내에서 반응형의 동작 이해하기
컴포넌트가 생성된 후 렌더링되고 이벤트에 반응하여 업데이트되며 파괴되기 까지의 일련의 과정을 컴포넌트의 라이프사이클이라고 한다.  
공식가이드에 있는 라이프사이클을 설명하는 그림이다.
<img src="https://vuejs.org/images/lifecycle.png" width="600"/>


**[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)은 위 라이프사이클과 함께 반응형 동작 방식을 좀 더 구체적으로 도식화 한것이다.**
### 그림-1
[![deep-dive-vue.js-v2.5.21-deep-dive-lifecycle][deep-dive-vue.js-v2.5.21-deep-dive-lifecycle]](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)

모든 코드를 다 보다보면 우리 모두 같이 빠져나올 수 없는 미로에 빠질 수 있으므로 라이프사이클을 기반으로 반응형이 어떻게 동작하는지에 대한 핵심적인 부분을 도식화하였다.
언뜻 보면 그림이 매우 복잡해 보이지만 아래 친절한 가이드와 함께 Step을 하나씩 따라 가면서 관련된 코드를 보면 이해하기 어렵지 않을 것이다.


다음은 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)에 대한 가이드이다.  
- 라이프사이클
  - **실선화살표로 표현**되며 라이프사이클의 flow를 나타낸다.
  - 그림 우상단의 start _init() 부터 따라가며 코드를 살펴보자
    - 순서도 상 빨간색 프로세스 네모는 라이프사이클 hook을 나타낸다.
- 의존성 연결
  - **파란색 도형/점선화살표/텍스트로 표현**되며 데이터를 참조할 때 의존성이 맺어지는 flow를 나타낸다.
- 변경 알림 및 업데이트(렌더링)
  - **빨간색 도형/점선화살표/텍스트로 표현**되며 데이터가 변경될 때 반응적으로 동작하는 flow를 나타낸다.


> **[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)은 이 문서가 전달하고자 하는 반응형 동작을 간단하지만 모두 설명하는 그림이다.**
> **이제부터는 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)을 클릭하여 따로 브라우저창을 띄워놓고 문서와 같이 보면 이해하는데에 많은 도움이 될 것이다.**

**[그림-2](http://drive.google.com/uc?export=view&id=1fmUBwe0QCVk5Cqh8lZyRzlydSim8L0MD)는 우리가 주로 살펴볼 소스코드들이다.**
### 그림-2
<img src="http://drive.google.com/uc?export=view&id=1fmUBwe0QCVk5Cqh8lZyRzlydSim8L0MD" width="600"/>

## Let's follow the '_init'
Vue를 생성하면 el과 data가 정의된 options객체를 _init함수의 인자로 전달하며 라이프사이클이 시작된다.  
![let's init](http://drive.google.com/uc?export=view&id=15ZoVUUGsvKYXZz5BsjMl3qy1TuXVCT6o)

![core instance init js](http://drive.google.com/uc?export=view&id=1ZaHuKE7AeNQVSLAUtlcHltmAIXxpV535)
`_init` 함수를 보자.  
3줄 : vm은 ViewModel의 약자이며 Vue 인스턴스 자신이다.  
6~17줄 : 전달받은 options을 Vue 기본 옵션정책을 적용하여 최종 vm.$options을 구성하고  
21~28줄 : 'init' prefix가 붙은 여러 초기화 작업들을 수행한다.  

반응형 구조를 이해하기 위해서 `_init` 함수에서 `initState`(26줄)와 `vm.$mount`(31줄)를 살펴본다  

## initState -> initData
먼저 `initState`로 들어가자.  
![core instance state js](http://drive.google.com/uc?export=view&id=1WPuYll5Bs9c-LA1h4bPqwiBGgGmZTw0J)
props와 methods, computed, watch는 건너뛴다.(demo Root Vue Component에 정의되지 않음)  
바로 data를 위한 `initData`(8줄)로 들어가자. 

![core instance initData](http://drive.google.com/uc?export=view&id=1_mTtoULTCmedihp9r7cEKaHQDqpsiILh)
3~12줄 : 아래 코드의 initData 함수에서 반응형으로 탈바꿈할 demo Vue 인스턴스의 data 구조도 같이 보자.  
15~18줄 : _init함수내에서 options를 재구성하면서  vm.$options.data는 함수로 변경되어 있다. 반환값은 3~12줄의 data 객체이다.  
32~47줄 : data의 모든 속성(`searchQuery, gridColumns, gridData`)들이 data 옵션보다 먼저 초기화되는 props/methods와 key가 중복 되는지를 체크한다. (`initState`에서 `initData`보다 `initProps`와 `initMethods`를 먼저 실행하기 때문)  

여기까지는 특별한 것이 없다. 
initData의 핵심은 49, 53줄이다.  

49줄 : `proxy(vm, '_data', key)`
- `proxy(vm, '_data', key)` 코드가 실행된 이후부터는 `vm.{key}`로 직접 접근이 가능해진다. 
- 예를 들어 Vue 인스턴스 내부 methods, computed, render 함수등에서 `this.searchQuery or this.searchQuery = 'any query'`등의 코드를 실행할 때 실제로는 vm._data 객체에 접근하여 값을 반환하거나 값을 변경한다.
- 이는 `this._data.searchQuery` 로 참조하는 것과 동일하다.  

아래는 proxy함수의 코드이다.  
![core instance proxy](http://drive.google.com/uc?export=view&id=1Hhps-X5CuhtqdpTK6F-QTdPlSgP5F2Cp)
먼저 이쯤에서 [Object.defineProperty][define-property]가 무엇인지 한번 보고 오자.  

2~7줄 : Object.defineProperty에 전달되는 descriptor는 공용으로 사용할 하나의 객체가 정의되어 있다.
10~15줄 : getter, setter를 재정의한다. `return this[sourceKey][key]` 이 코드는 결과적으로 `vm.searchQuery`의 값을 참조하면 `vm._data.searchQuery`의 값을 반환해주는 것이다. setter도 같은 방식이다.

data의 모든 속성을 순회하며 proxy 함수의 실행이 완료되면 `vm.searchQuery`, `vm.gridColumns`, `vm.gridData`와 같은 형태로 vm._data에 접근이 가능해진다.  


initData 53줄 : `observe(data, true /* asRootData */)`
- initData 함수의 마지막 작업이 data 객체를 반응형 객체로 탈바꿈 시키는 것이다. 그렇다, 여기서도 **Object.defineProperty**를 사용한다.  
- 전달되는 인자 data는 vm._data이다.

## observe
observe함수는 기본적으로 재귀적으로 동작한다. 전체 과정을 그림으로 먼저 이해해보자.  
### 그림-3
[![deep-dive-vue.js-v2.5.21-deep-dive-observe][deep-dive-vue.js-v2.5.21-deep-dive-observe]](http://drive.google.com/uc?export=view&id=1KCzjQe3uvx7oib8742TALgenC8xsIf-v)

[그림-3](http://drive.google.com/uc?export=view&id=1KCzjQe3uvx7oib8742TALgenC8xsIf-v)에서 빨간 부분이 가장 중요한 부분이다.  

![core observer index js](http://drive.google.com/uc?export=view&id=1loEGiVY9aDYHyMT1rPL0n7BSUuXHr_hf)
`observe` 함수는 객체 또는 배열 데이터(3~5줄)에 대해서만 Observer를 만들어 준다.  
16줄 `ob = new Observer(value)`를 따라가보자.  

아래는 Observer 클래스의 생성자 코드이다.  
![core observer constructor](http://drive.google.com/uc?export=view&id=1f8UnahQyu6E5QLVHx-Xgl4A0Fp-iTS4-)
9줄에 보면 Dep Class의 인스턴스를 하나 생성한다. Dep은 뭘까?  
[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO) 좌측에 Dep과 Watcher 클래스의 관계도가 있다.  

잠시 의문을 접어두고 일단 다음을 보자.  
11줄의 `def(value, '__ob__', this)`를 통해 vm._data객체에 `__ob__` 속성을 정의하고 Observer 인스턴스를 연결한다.  
`vm._data.__ob__ = {Observer 인스턴스}` 이다.  
`observe` 함수의 7줄에서 이미 Observer가 생성되었는지 체크하는 코드도 볼 수 있다.  

`vm._data` 를 반응형 객체로 만드는 과정의 첫 순서는 `vm._data.__ob__`에 Observer를 생성하여 연결하는 것이고,  
이후 12~20줄의 `observeArray`와 `walk` 함수를 통해 `vm._data`의 하위 속성들을 순회하면서 하위의 모든 배열과 객체에, 그리고 배열안의 객체들까지 재귀적으로 Observer를 생성하고 할당한다.(재귀적인 `observe` 함수 호출)  


Observer와 함께 `walk` 함수내에서는 이 과정에서 배열과 객체의 각 속성들을 위한 getter/setter들도 설정된다.  

먼저, 배열인 경우를 살펴보자.

### methodsToPatch for Array
value가 배열인 경우와 객체인 경우 서로 다른 방식으로 getter/setter를 정의한다.  
13~17줄 : 먼저, 배열인 경우 `protoAugment`또는 `copyAugment` 함수를 통해 setter를 정의한다.  

[그림-3](http://drive.google.com/uc?export=view&id=1KCzjQe3uvx7oib8742TALgenC8xsIf-v)의 `methodsToPatch` 를 아래 코드를 통해 살펴보자.  
![core observer array js](http://drive.google.com/uc?export=view&id=140fwROtswfeP04iZvCgT7KJaTgQk8hRx)

12~13줄 : Array.prototype을 prototype으로하는 새로운 객체인 arrayMethods를 생성한다.  
31줄
- arrayMethods의 배열 조작함수들을 모두 재정의한다. (Object.defineProperty 사용-def 함수)  
- 이렇게 재정의된 배열 조작함수들은 결국 반응형으로 동작하는데에 쓰일 setter들이다.(mutator function)  

#### setter의 역할
1. 32줄 : Array.prototype의 배열 조작함수를 original이란 이름으로 먼저 호출한다.
2. 35~44줄 : 새로운 요소를 배열에 추가하는 함수가 호출되었다면 `observeArray` -> `observe` 함수를 호출 하여 추가된 요소(들)를 반응형으로 구성해준다. 
2. 33줄,46줄 : 배열의 Observer 인스턴스`__ob__`의 dep 인스턴스를 통해 Watcher에 변경을 알려준다.(이 부분의 정확한 이해는 이 문서를 읽다 보면 아~ 하는 순간이 올 것이다. [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO) 좌측에 Dep과 Watcher 클래스의 관계도가 있다.)

여기까지가 배열의 setter를 정의하는 부분이다.  

`protoAugment` 함수는 이렇게 정의된 하나의 arrayMethods 인스턴스를 전달된 배열 data만을 위해 `{배열data}.__proto__`에 연결한다.  

`__proto__`속성을 지원하지 않는 환경에서는 `copyAugment` 함수를 통해 재정의한다. (def 함수 -> 내부적으로 Object.defineProperty 사용하여 재정의)  

### defineReactive for Object
이제 value가 객체인 경우 getter/setter를 정의하는 코드를 보자.
![core observer object walk](http://drive.google.com/uc?export=view&id=13UUhcIl-8iNsplaDHwIzk3lYMXB0xa8y)
15줄에서 전달된 값이 객체인 경우 walk함수를 호출한다.  

walk함수는 객체의 모든 속성을 defineReactive함수를 통해 반응형 객체로 만든다.  

defineReactive 코드를 보자.
![core observer definereactive](http://drive.google.com/uc?export=view&id=1D5NPNrLc3TIIt-Wn9mO_Wg6QYk6DmP9q)
기본적으로 reactivityGetter, reactivitySetter는 내부함수(Closure)로 선언되어 defineReactive가 호출되는 시점에 생성되는 지역변수 dep, val, childOb등을 private하게 접근한다.  
12줄 : Dep Class의 인스턴스를 반응형 속성을 위해서 생성한다.  
이쯤에서 아까 보았던 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)의 Dep과 Watcher의 관계도를 다시 보고 오자.

#### getter의 역할(27~37줄)
1. 기본적으로 vm._data의 어떤 속성에 접근했을 때 getter가 호출된다.
2. 28줄 : Dep과 연결이 가능한 Dep.target(watcher 인스턴스)이 존재한다면, 
   - 컴포넌트의 `render`함수가 호출되는 시점에 Dep.target에 컴포넌트 Watcher인스턴스가 설정된다.
3. 29줄 : `dep.depend()`를 호출함으로서 **dep.subs에는 의존되는 Dep.target(watcher 인스턴스)이 저장** 된다.
   - [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)의 라이프사이클 중 옅은 주황색 둥근 사각형으로 강조 표시한 곳부터 의존성 연결 Step1~6까지를 살펴보자.
4. props등을 통해 한 컴포넌트의 data가 다른 컴포넌트에 전달될 수 있고, 하나의 속성을 참조하는 여러개의 computed가 만들어질 수 있다. 즉, 하나의 속성을 참조하는 Watcher는 여러개가 될 수 있다.
5. computed는 컴포넌트 Watcher가 아니라 computed만을 위한 Watcher가 따로 생성된다.
6. 최종적으로는 속성값을 반환한다.

#### setter의 역할(40~57줄)
1. DOM 이벤트나 그 외 기타 여러가지 방법들을 통해 vm._data의 속성을 변경할때 setter가 호출된다.
2. 54줄 : 속성의 값(val)에 새로운 값(newVal)을 적용한다.
3. 56줄 : 만약 newVal이 Object 또는 Array라면 `observe` 함수를 통해 다시 반응형으로 구성해준다.
3. 57줄 : `dep.notify()`를 호출함으로서 **dep.subs에 저장된 모든 Watcher들에 변경을 알려주는 역할**을 한다.
   - `dep.subs`는 `vm._render` 함수가 속성을 참조할때 각 속성들의 getter의 의해서 의존성이 맺어져 연결된다.


#### Array의 내부 요소를 위한 getter
- 위에서 Array는 setter만 보았는데 이유는 다음과 같다.
- Array의 내부 요소를 접근하는 것을 Proxy할 수 있는 방법은 없다. (Proxy객체를 사용한 감지는 Vue.js 3.0 로드맵이다.)
- 그래서 다음 `dependArray`함수를 통해 Array의 접근을 감지한다.
- 이 메서드의 호출은 객체의 속성 getter에서 호출된다.
  - 객체의 속성에 접근할때 속성의 값이 배열이면 `dependArray`를 통해 `배열.__ob__.dep.depend()`를 사용하여 Watcher와 의존성을 연결한다.
  - `배열.__ob__.dep`은 배열의 setter에서 사용하는 동일한 dep 인스턴스이다.

![core observer array dependarray](http://drive.google.com/uc?export=view&id=17RdSynZEtfQfbvaRK2ZyI091z7VV-Orb)

여기까지 `initData` 함수의 수행이 완료되면 반응형으로 동작할 준비과정이 완료된다.
지금까지 살펴본 코드에서 Dep이 생성되는 곳과 사용되는 곳을 알 수 있었다.  

## Dep과 Watcher
[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)을 다시 한번 보자.  
[![deep-dive-vue.js-v2.5.21-deep-dive-lifecycle][deep-dive-vue.js-v2.5.21-deep-dive-lifecycle]](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)

Dep과 Watcher는 서로 1...n의 관계를 가진다.  
data가 반응형 객체가 되는 과정에서의 Dep 생성과 사용은 다음과 같다.  
1. Object.defineProperty로 data의 각 속성별 getter/setter를 정의할 때 해당 속성을 특정 Watcher와 의존시키기 위해 생성된다.
2. 객체 및 배열을 위해 Observer(내부 속성 `__ob__`으로 참조)를 생성하는데 Observer마다 하나의 Dep을 생성한다.
   - 이는 배열의 요소별 getter를 구현할 수 없어서 배열 자체에 접근시 배열을 특정 Watcher와 의존시키기 위해 사용된다.

지금까지 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)의 우측 라이프사이클의 `initState`안에서 어떤 일이 일어나는지 살펴보았다.(정확히는 `initData` 이다.)  
 
## $mount
mount 단계의 flow는 다음과 같다.  
`$mount -> mountComponent -> beforeMount -> new Watcher -> render -> update DOM -> mounted`  

render나 update DOM부분은 가볍게 살펴보자.  

[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)을 같이 보면서 `_init` 코드를 다시 보면 
![core instance init js](http://drive.google.com/uc?export=view&id=1ZaHuKE7AeNQVSLAUtlcHltmAIXxpV535)
27줄 : `initProvide`(이 부분은 플러그인등을 만드는 것과 관련 있는 부분으로 차후에 기회가 생기면 다루도록 한다.)까지 완료 하면  
28줄 : `created` 라이프사이클 훅이 호출된다.  
여기까지가 create 단계이다.  

31줄부터 `vm.$mount`를 따라가보자.  
![runtime mount](http://drive.google.com/uc?export=view&id=1HiAz7Oq3fcG01FwGf4ydF_m0A8JYeFcI)

위 코드를 포함하는 모듈 `entry-runtime-with-compiler.js`는 runtime에 템플릿을 컴파일하여 `render`함수를 생성하는 작업을 책임지는 모듈이다.  

일단 런타임에 컴파일을 템플릿하는 모듈이 포함된 vue bundle을 사용하는 경우 
2줄에서 처럼 기존에 정의된 `$mount` 함수는 따로 참조를 해놓는다.  
이후 3줄에서 `Vue.prototype.$mount`를 새로 정의한다.  
그리고 render함수를 생성한 후 이전에 따로 참조해놓은 `$mount`함수를 호출한다.  


19줄 : render 함수가 없다면(runtime에 render함수를 만들어야 하는 경우)
24,34,42줄 : template과 el을 통해 HTML 문자열을 얻어온다.(innerHTML,outerHTML)  
50줄 : 이후 `compileToFunctions` 함수를 통해 render 함수를 생성한다.
- `compileToFunctions` 함수의 동작 방식은 결국 이 문서의 주제와 관련이 없으므로 생략한다.

66줄 : 기존 정의된 `$mount` 함수를 최종적으로 호출한다. `mount.call(this, el, hydrating)`


위 코드 66줄에서 호출하는 mount함수의 코드를 보자.
![public mount](http://drive.google.com/uc?export=view&id=1l5ZUzhXB2oHq210h_4Oo5T0nLASz14XV)
7줄: browser에서 실행되는 경우 el을 설정한 후 `mountComponent` 함수를 호출한다.  


아래 `mountComponent`함수를 보자.  
![core instance lifecycle mountcomponent](http://drive.google.com/uc?export=view&id=1QfQnYc9qFyb8YpJRi_86qp8k8VFSV4nB)
28줄 : `beforeMount` 라이프사이클 훅을 호출한다. created와 beforeMount 사이에는 render함수가 만들어지는 과정이 있었다.  
50줄 : `updateComponent` 함수를 정의한다. `vm._render`는 vnode를 생성하며 `vm._update`는 vnode를 실제 DOM에 patch한다. 이러한 작업을 하나의 task로 묶어서 `updateComponent` 라는 이름의 함수로 정의한다.  
58줄 : Vue 인스턴스를 위한 Watcher를 생성하면서 두번째 인자로 `updateComponent`를 전달한다.  


## watcher
위 코드 58줄의 `new Watcher(vm, updateComponent, ...`를 따라가보자.  
![core observer watcher](http://drive.google.com/uc?export=view&id=1hvoFY1y703rmbYNY9PoRFvURCh7ISbQA)
지면을 위해 지금 살펴보지 않을 코드들은 생략하였다.  
24줄 : Watcher의 생성자에서는 두번째 인자로 받은 `updateComponent` 함수를 내부 변수 `getter`로 설정한다.  
39줄 : 마지막에 내부 함수 `get`을 호출한다. `get` 함수는 `getter`함수를 사용하여 Watcher와 Watcher가 필요한 컴포넌트의 data와 의존성을 맺는 과정을 진행한다.  


또 한번, 이쯤에서 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)을 다시 한번 보자.  
[![deep-dive-vue.js-v2.5.21-deep-dive-lifecycle][deep-dive-vue.js-v2.5.21-deep-dive-lifecycle]](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)

그림 우측 LifeCycle영역에 보면 주황색으로 일부 Flow에 대해서 강조표시를 해놓았다.  
'Watcher를 Dep.target으로 등록' 이라는 문구부터 의존성 연결 작업이 시작된다.  
이 Flow의 코드가 `get` 함수이다.  
코드를 보자
![core observer watcher get](http://drive.google.com/uc?export=view&id=1oxac3lSI0s2uQd1ShAHI5UgA-4hQkQyh)

[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)과 위 코드를 같이 보며 따라가보자.
[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO) 파란색 step 1
- 6줄 : pushTarget(this)를 호출하여 Dep.target값을 현재의 Watcher로 설정한다.  

[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO) 파란색 step 2
- 10줄 : this.getter를 호출하면 전달받은 `updateComponent` 가 호출된다.  
![core instance lifecycle updatecomponent](http://drive.google.com/uc?export=view&id=1c0XJM8crXoRSQOWf4w3KsGHvsQqkUe8-)

updateComponent 동작
- vm._render를 통해 vnode를 생성한다.  
- vnode를 생성하기 위해 **render함수가 평가될 때 data의 값을 참조**하게 된다.  
- **data의 값을 참조할 때 속성 getter가 호출**된다.  

[그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO) 파란색 step 3 ~ 5
- **getter는 속성 dep 인스턴스를 통해 현재의 Watcher(Dep.target)와 의존성 연결**이 된다.

컴포넌트의 `render`함수가 수행되면서 vnode가 만들어질때 `render` 함수는 data를 참조하게 된다. 이때 참조되는 data속성의 getter 함수에서 Dep(dep 변수)과 Watcher(Dep.target)가 연결되어 의존 관계가 된다.  

이후, `vm._update`함수에 vnode가 전달된다.  
`_update` 함수도 내부적인 동작은 차후 다른 주제로 다룰 예정이나, 일단 현재는 간략히 개념만 본다면 이 함수는 두가지 역할을 한다.
1. vnode를 실제 DOM에 적용한다.
2. vnode안에 child component가 있다면 다시 해당 child component의 라이프사이클을 시작한다.(모든 최하위 child component까지 라이프사이클을 시작하며 역순으로 mounted 된다.)
   - 예제를 기준으로 vm._update함수에서 demo-grid child 컴포넌트를 생성한다.
   - 이후 demo-grid child 컴포넌트가 지금까지 살펴본 라이프사이클을 수행하고 mounted가 먼저 된다.

23줄 : `popTarget`함수를 통해 Dep.target을 다시 이전 상태로 원복한다.  
24줄 : `cleanupDeps`를 통해 Watcher의 기존 dep(의존된 속성들)들을 모두 정리하고 새로 의존된 dep들로 교체한다. 말 그대로 cleanup!!  

여기까지 진행되면 위 `mountComponent` 함수의 71줄 `callHook(vm, 'mounted')`를 통해 demo vm의 mounted 훅이 호출되고 `_init`함수가 종료된다.

# 되돌아보기
간단한 예제였지만, 꽤나 긴 여정이었다.  

> demo root는 **rootVm**, demo-grid child 컴포넌트는 **childVm**이라 칭한다.
> ![examples grid html 1](http://drive.google.com/uc?export=view&id=1_GODhq2M_G-okG7f4e0A92TyJIblbxfi)

지금까지의 과정을 다시 한번 요약해보자.  
- rootVm._init
   - rootVm's beforeCreate
   - rootVm.initState -> rootVm.initData -> observe(rootVm._data) -> getter/setter 정의
   - rootVm's created
   - rootVm.$mount -> rootVm.$options.render 함수 생성
   - rootVm's beforeMount
   - rootVm._render
     - **rootVm._data와 rootVm's Watcher 연결/의존 관계 수립**
   - rootVm._update
     - childVm._init
       - childVm's beforeCreate
       - childVm.initState -> childVm.initData -> observe(childVm._data) -> getter/setter 정의
       - childVm's created
       - childVm.$mount -> childVm.$options.render 함수 생성
       - childVm's beforeMount
       - childVm._render
         - **childVm._data와 childVm's Watcher 연결/의존 관계 수립**
       - childVm._update
         - childVm의 Real DOM patch
       - **childVm's mounted**
     - rootVm의 Real DOM patch
   - **rootVm's mounted**

**의존 관계 수립 : parent -> child의 순서로 완료된다.**  
**Mounted : child -> parent의 순서로 완료된다.**  

아직 끝나지 않았다.  
지금까지 이 긴 여정을 통해 반응형으로 동작하기 위한 기반이 마련된 것을 볼 수 있다.  

그럼 실제 반응형 동작은 어떻게 동작하는지 보자.  

## change data and re-render
기본적으로 이 과정은 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)의 빨간색으로 강조된 Step 1 ~ 9 flow를 코드로 설명한다.  
demo-grid 컴포넌트의 코드를 보자.
![examplges grid js sortBy](http://drive.google.com/uc?export=view&id=1qla8HjnHtf_hPVNNYHJDmQQFTik8d-Ll)

`sortBy` 메서드는 테이블 헤더를 클릭했을때 정렬이 되도록 하는 클릭 이벤트 핸들러이다.  
테이블 헤더를 클릭해서 `sortBy` 메서드가 호출되면 
1. `this.sortKey = key`를 통해 `this._data.sortKey`의 setter가 호출된다.
2. setter에서 sortKey를 위한 dep인스턴스에 연결된 Watcher(dep.subs)에 변경을 알린다.(dep.nofity() -> Watcher's update)
   - 실제 sortKey속성은 두개의 Watcher와 의존 관계로 연결되어 있다.
   - 하나는 demo-grid component를 위한 updateComponent용 Watcher
   - 하나는 demo-grid내부의 `filteredData` computed Watcher

Watcher의 update함수를 보자.
![core observer watcher update](http://drive.google.com/uc?export=view&id=1urFoMhUD2hdN9PDnTgnCtI8AeSrUtHOn)
4줄 : lazy가 true인 경우는 computed를 위한 Watcher인 경우이다. `filteredData` computed Watcher는 lazy가 true이므로 향후 computed의 값을 참조하는 타이밍에 Watcher의 update가 수행된다.(`filteredData` 재계산)  
6줄 : sync는 vue-test-utils등 테스트 환경등에서 동기적으로 렌더링하기 위한 속성이며 production환경에서는 사용되지 않는다.  
9줄 : `queueWatcher`는 컴포넌트용 Watcher(DOM을 업데이트 하는 Watcher)인 경우 수행되는 코드이다.  

`queueWatcher`를 보자.  
![core observer scheduler](http://drive.google.com/uc?export=view&id=1vF66TtQO10RmTB-AwUWSeqmQmpXbwuch)
4줄 : `if (has[id] == null) ` 코드를 통해 queue에 등록된 Watcher인지를 판단하는데, 이는 하나의 Watcher에 여러개의 변경 알림이 통지됐을 경우 단 한번만 Watcher를 업데이트 하기 위해서 queue에 추가하는 조건이다.  
7줄 : `queue.push(watcher)` 코드로 queue에 Watcher를 등록
25줄 : `nextTick(flushSchedulerQueue)` 코드로 nextTick, 즉 다음 이벤트 루프일때 수행되도록 현재 수행 Task와 분리한다.


`flushSchedulerQueue`함수의 코드를 보자.  
이 함수는 속성 변경 이벤트 핸들러(sortBy)가 종료된 후 nextTick을 통해 그 다음 tick에 수행된다.
![core observer scheduler flush queue](http://drive.google.com/uc?export=view&id=1D1CpXWNeOJYOuHw6aVZO1qOFAsaHQrpS)
17줄 : `sortBy` 함수에서 sortKey와 sortOrders를 모두 변경하면 다음 이벤트 루프에서 `flushSchedulerQueue`이 수행될 때 queue에 존재하는 Watcher는 단 한개이다.
- 두 속성 모두 demo-grid 컴포넌트 Watcher와만 의존관계를 수립하기 때문이다.
- computed 속성인 `filteredData`의 Watcher는 queue에 등록되는 Watcher가 아니다. computed 값이 참조되는 순간에 재평가가 된다.

24줄 : `flushSchedulerQueue`는 queue에 등록된 Watcher들의 `run`함수를 호출한다.  


![core observer watcher run](http://drive.google.com/uc?export=view&id=1QWxMfPEisG1U5UtFAyjhaBffvITps7vm)

Watcher의 run함수는 최종적으로 Watcher의 `get` 함수, 즉 updateComponent(컴포넌트 라이프사이클 상에서 `mounted`가 되기전에 Watcher의 getter속성에 연결된다.)를 수행하는 함수를 호출한다.  

![core instance lifecycle updatecomponent](http://drive.google.com/uc?export=view&id=1c0XJM8crXoRSQOWf4w3KsGHvsQqkUe8-)

또한, 이때 vm._render를 수행하는 과정에서 참조되는 data와 컴포넌트 Watcher는 다시 의존 관계를 갱신하게 된다. (이유는 기존에는 의존관계에 있었지만 값의 변경과 조건으로 인해 render시에 참조하지 않게되는 경우가 생길 수 있다. 또는 그 반대의 경우도 생길 수 있기 때문에 **data와 Watcher간의 의존 관계는 render가 수행될 때 마다 갱신되어야 한다**. 예를 들면 v-if등을 통해 특정 조건에 의해 특정 data가 참조되거나 참조되지 않을 수 있다.)

최종적으로 **vm._render -> vnode 전달 -> vm._update가 수행되어 DOM이 업데이트** 된다.  

여기서 **왜 data를 변경했을때 바로 DOM에는 반영되지 않는지**에 대한 답도 나와 있다.  
그렇다. nextTick을 통해 `updateComponent`가 task queue에 추가되고 다음 이벤트 루프에서 task queue에 존재하는 updateComponent함수를 call stack으로 가져와 실행하기 때문이다.  
(deep dive의 주제 중 'nextTick 깊게 알아보기'를 진행할 때 task queue와 이벤트 루프도 같이 알아보자.)  

지금까지 Vue의 반응형이 어떻게 동작하는지 vue.js의 코드와 함께 살펴보았다.  
이 동작 flow와 관계를 최대한 하나의 그림에 담은게 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)이다.  
마지막으로 [그림-1](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)을 다시 한번 보자.  
[![deep-dive-vue.js-v2.5.21-deep-dive-lifecycle][deep-dive-vue.js-v2.5.21-deep-dive-lifecycle]](http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO)


## 마치며
처음에는 Vue.js의 반응형의 실체를 이해하기 위해 코드 분석을 시작했지만 공유하면 조금이라도 다른 개발자들에게 도움이 될 수 있지 않을까 하는 마음이 들어 문서로 정리하였다.  


### P.S
문서의 내용중 잘못된 부분이나 개선이 필요한 부분이 있다면 피드백 해주시면 좋을 거 같습니다.
vamalboro@gmail.com


Written by 피스티스.

---

[git-1]: https://github.com/vuejs/vue/tree/v2.5.21

[git-2]: https://github.com/vuejs/vue/tree/v2.5.21/examples/grid

[guide-1]: https://vuejs.org/v2/guide/reactivity.html

[guide-2]: https://kr.vuejs.org/v2/guide/reactivity.html

[guide-3]: https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function

[define-property]: https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

[img-1]: https://vuejs.org/images/data.png

[grid-example-screenshot]: http://drive.google.com/uc?export=view&id=1bj6bFwAwGOgIB2V__tL5b7xqB6uO1PmL

[deep-dive-vue.js-v2.5.21-deep-dive-lifecycle]: http://drive.google.com/uc?export=view&id=1N0VqBqEfSvByYmcIGzSqhVvx6SOHyRVO

[deep-dive-vue.js-v2.5.21-deep-dive-observe]: http://drive.google.com/uc?export=view&id=1KCzjQe3uvx7oib8742TALgenC8xsIf-v

[lifecycle]: https://vuejs.org/images/lifecycle.png
