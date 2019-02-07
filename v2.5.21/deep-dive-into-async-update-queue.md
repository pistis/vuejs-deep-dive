> _이 문서는 [vue.js v2.5.21][git-1] 코드를 기반으로 작성되었습니다._  
> _이 문서는 [영문 공식가이드][guide-1]와 [한국어 공식가이드][guide-2]를 참고하였습니다._
---

선행 학습
- *[Deep dive into Reactivity in Depth][reactivity-in-depth] Vue가 반응형 구조를 위해 변경내용을 어떻게 추적하는지에 대한 아티클*
- '*자바스크립트는 싱글 쓰레드 라면서 도대체 어떻게 비동기를 처리하는가?*' 
  1. [What the heck is the event loop anyway?][youtube-Philip-Roberts] : Philip Roberts라는 개발자의 유투브 강연 영상이며 한글 자막이 제공된다. 매우 쉽게 설명한다.
  2. [자바스크립트와 이벤트 루프](https://meetup.toast.com/posts/89) : NHN Enter FE 기술블로그에 있는 글이며 매우 자세하고 친절하게 설명한 글이다.
  3. [tasks-microtasks-queues-and-schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) : stackoverflow던 기술 블로그의 아티클 글이던 많은 글들이 지금 이 링크를 참조한다. 실제 예제 코드와 애니메이션으로 이해가 쏙쏙~
  4. [difference-between-microtask-and-macrotask-within-an-event-loop-context](https://stackoverflow.com/questions/25915634/difference-between-microtask-and-macrotask-within-an-event-loop-context) : Stackoverflow 글이다. 한번 읽어보자.

# 프롤로그
## Async Update Queue
[한국어 공식가이드][guide-2]의 번역된 뜻으로는 "비동기 갱신 큐"이다.  

우리는 왜 "비동기 갱신 큐"에 대해서 이해해야 하는가?  
기본적으로 data를 수정하면 Vue는 자동으로 DOM을 업데이트 한다.  
특별한 일이 없는 이상 DOM을 직접 참조할 일이 드물다.  

다만, 간혹 데이터를 수정한 직후에 DOM에 직접 엑세스 해야만 할 때가 있다.  
그때 사용되는 것이 "비동기 갱신 큐"이며 API는 [nextTick](https://vuejs.org/v2/api/#vm-nextTick)이다.  

데이터를 변경한 직후에 변경된 DOM에 엑세스할 때 어떠한 메커니즘으로 동작하는지 이제부터 살펴본다.  

[한국어 공식가이드][guide-2]상의 내용 중 핵심은 다음과 같다.  
1. Vue는 데이터 변경이 되면 DOM을 비동기로 업데이트 한다.
2. 데이터 변경이 발견 될 때마다 큐를 열고 같은 이벤트 루프에서 발생하는 모든 데이터 변경을 버퍼에 담는다.
   - *Whenever a data change is observed, it will open a queue and buffer all the data changes that happen in the same event loop.*
4. 그 다음, 이벤트 루프 “tick”에서 Vue는 대기열을 비우고 실제 (이미 중복 제거 된) 작업을 수행한다.
5. 내부적으로 Vue는 비동기 큐를 위해 네이티브 Promise.then 와 MessageChannel를 시도하고 setTimeout (fn, 0)으로 돌아간다.
   - *Internally Vue tries native Promise.then and MessageChannel for the asynchronous queuing and falls back to setTimeout(fn, 0).*

일단, 위 2번은 [Deep dive into Reactivity in Depth][reactivity-in-depth] 이 문서에서 자세히 다루고 있다.  

3번과 4번의 실체를 알기 위해 vue.js 소스코드에 deep dive 해보자.  


## 자바스크립트의 비동기 처리
선행 학습을 했다면(또는, 학습전에 이미 알고 있었다면) 우리는 이벤트 루프와 task/micro task queue, 그리고 자바스크립트 런타임 call stack과 webapi가 무엇인지 알고 있다.  

먼저, task와 micro task는 다음과 같은 것들이 있다.  

### task(a.k.a macro task라고도 부른다.)
1. setTimeout
2. setInterval
3. setImmediate(IE 전용)
4. requestAnimationFrame
5. I/O Task
6. UI rendering
7. DOM Event

### micro task
1. Promise
2. Object.observe
3. MutationObserver

우리가 기억해야 할 것은 다음과 같다.  
1. 브라우저에서 DOM의 click 이벤트등이 발생했을 때 실행되는 이벤트 핸들러는 macro task queue에 들어간다.(이벤트 전파를 통해 수행되는 이벤트 핸들러도 동일하다.)
2. 이벤트 루프를 통해 queue의 task들이 비워지는 순서는 micro task -> macro task 순서이다.
   - micro task가 수행되는 동안 micro task가 새로 추가되면 macro task가 이미 존재하더라도 새로 추가된 micro task가 macro task보다 먼저 수행된다.  

## 데이터의 변경과 DOM 업데이트
[Deep dive into Reactivity in Depth][reactivity-in-depth] 이 문서에서 이 동작에 대해 이미 자세하게 정리하였으니 먼저 문서의 'change data and re-render' 섹션을 다시 한번 확인하자.  

[grid example][git-2]의 스크린샷은 다음과 같다.  
![grid-example-screenshot][grid-example-screenshot]

요약해보면 [grid example][git-2]에서 테이블 헤더를 클릭하면 다음과 같은 순서로 데이터가 변경되고 DOM이 업데이트 된다.  
1. sortBy function (examples/grid/grid.js) : sortKey와 sortOrder data를 변경한다.
2. notify function (src/core/observer/dep.js) : 반응형 객체의 속성이 변경되면서 속성과 의존관계의 Watcher에 변경을 통지한다.(watcher.update())
3. update function (src/core/observer/watcher.js) : Watcher의 update함수는 Scheduler의 queueWatcher를 호출하며 Scheduler는 Watcher를 업데이트 대기열에 등록한다.
4. queueWatcher function (src/core/observer/scheduler.js) : 대기열에 Watcher를 등록(중복 검사)한 후 queue를 비우는 flushSchedulerQueue 함수를 **nextTick**으로 실행한다.

grid component의 `sortBy` 함수가 종료되기까지 1 ~ 3번까지는 동기로 수행된다.  
즉, call stack에 쌓여서 순차적으로 처리된다.  
단, 4번의 `flushSchedulerQueue` 함수는 `sortBy` 함수가 종료되기전에 `nextTick` 에 의해서 비동기로 처리 되도록 task queue에 task가 등록 되며 1 ~ 3번 task가 수행되는 call stack에 추가되지 않는다.  

결국 call stack이 비워지고 다음 이벤트 루프에서 실행된다.  
그로 인해, sortBy 함수내에서 data를 수정하고 바로 DOM에 엑세스를 하면 변경된 DOM을 참조할 수 없다.  

코드를 보자.  
다음은 `queueWatcher` 함수이다.  
![core observer scheduler](http://drive.google.com/uc?export=view&id=1vF66TtQO10RmTB-AwUWSeqmQmpXbwuch)

25줄 : `nextTick(flushSchedulerQueue)` 코드를 확인할 수 있다.  

여기까지 확인한 결과 우리는 [영문 공식가이드][guide-1]에 왜 이런 내용이 있는지 정확히 이해할 수 있다.  
> For example, when you set vm.someData = 'new value', the component will not re-render immediately. It will update in the next “tick”, when the queue is flushed. 

여기서 만족할 수 있나?  
nextTick의 내부동작을 좀 더 파헤쳐보자.  

## nextTick
먼저 `nextTick` 함수의 코드를 보자. (src/core/util/next-tick.js)  
![nexttick function][nexttick function]

`nextTick`은 자신이 비동기로 수행할 callback 함수들을 관리하는 `callbacks` 큐를 가지고 있다.  
3줄 : `nextTick`이 호출되면 callback 함수를 수행하는 함수를 만들어 큐에 삽입한다. (위 sortBy 함수 동작시 4번 step의 `flushSchedulerQueue`함수가 callback이 된다.)  
11, 24 ~ 26줄 : `nextTick`를 호출하며 인자로 전달하는 것이 아닌 Promise의 then으로 등록하는 형태의 callback도 허용한다.([v2.1부터 지원](https://vuejs.org/v2/api/#vm-nextTick)), 단 이 방식을 사용하면 인자로 callback을 전달하는 방식과 실행의 우선순위가 다르다.(이 부분은 나중에 살펴 보자.)  
16 ~ 20줄 : 여기가 중요한 부분이다. macro task와 micro task가 언급되며 useMacroTask변수의 값의 참/거짓 유무에 따라 `callbacks`큐를 비우는 두 타입의 함수를 호출한다.  

왜 nextTick은 두개의 방식중에 하나를 선택하여 callback을 수행할까?    

결론을 먼저 말하면 v2.4까지는 항상 micro task 방식을 사용했었다.  
하지만, 이는 너무 높은 우선순위를 가지기에 그로 인해 Vue를 사용할 때 의도치 않은 버그가 발생하였다.  


관련된 버그들은 다음과 같다.  

* https://github.com/vuejs/vue/issues/4521
* https://github.com/vuejs/vue/issues/6690
* https://github.com/vuejs/vue/issues/6566

이중에 [@click would trigger event other vnode @click event. · Issue #6566 · vuejs/vue · GitHub](https://github.com/vuejs/vue/issues/6566) 이슈를 살펴보자.  

![issue6566 code](http://drive.google.com/uc?export=view&id=1DuHGAJqbaiouYm04BzJTiqD84Mb62Cls)

코드의 작성자는 의도는 다음과 같다.( 이 코드는 버그를 보여주려는 의도로 작성된 코드이다.)  
1. header class로 정의된 div > i 태그를 클릭하면 expand를 false로 변경하고 countA값을 증가시킨다. 이후 expand class로 정의된 div가 렌더링되고 header class div는 사라진다.
2. expand class로 정의된 div 태그(i tag가 아니다)를 클릭하면 expand를 true로 변경하고 countB값을 증가시킨다. 이후 header class로 정의된 div가 렌더링되고 expand class div는 사라진다.

의도는 아주 명확하다. header와 expand가 공존할 수 없으며 toggle되면서 count가 증가하고 둘중에 하나만 노출되는 예제이다.  

근데 이게 2.4 버전에서 비정상적으로 동작한다.  

[issues/6566 버그 수행코드](https://jsbin.com/qejofexedo/edit?html,js,output) 이 링크를 클릭하면 버그를 확인할 수 있다.  

nextTick에서 우선순위가 높은 micro task만 사용했던 vue.js v2.4의 동작 순서이다.  
1. 'Expand is True' text를 포함하는 header class로 정의된 div > i 를 클릭
2. expand = false로 데이터 변경, countA++로 데이터 변경, 이후 data변경으로 컴포넌트의 watcher에 notify된다.
3. queueWatcher에의해 watcher가 업데이트 예약된다.
4. nextTick(flushSchedulerQueue)으로 watcher 대기열을 flush하도록 등록된다.
5. nextTick은 micro task(Promise)로 예약되어 수행된다.
6. 이후 이벤트 전파를 통해 수행될 모든 event listener보다 먼저 DOM을 업데이트 하는 watcher.run()이 수행된다.
7. i에서 발생한 click 이벤트가 버블링된다.
8. 변경된 DOM구조에서 div태그로 이벤트가 전파되고 이 시점에서 div는 expand class div가 된다.
9. expand class div에 click event listener(macro task)가 수행된다.
10. expand = true로 data를 변경, countB++로 데이터 변경, 이후 data변경으로 컴포넌트의 watcher에 notify된다.
11. 이후 3~6번의 과정이 동일하게 반복된다.

결론적으로 1~11번 step의 과정이 끝나면 사용자 입장에서는 'Expand is False' 텍스트를 볼 수 없고 countA와 countB가 모두 증가된 상태로 'Expand is True'의 DOM을 보게 된다.  

문제는 무엇인가?  
> 이것을 다시 기억하자.
> 'micro task가 수행되는 동안 micro task가 새로 추가되면 macro task가 이미 존재하더라도 새로 추가된 micro task가 macro task보다 먼저 수행된다.'

사실상 기대되는 동작은 'Expand is True' 텍스트를 포함하는 i 태그의 click event listener만 동작하고 DOM이 업데이트된 후 더이상 동작할 함수는 없어야 한다.  
하지만, 처음 클릭시 data변경 후 micro task인 render함수(watcher.run())가 수행되고, 그로인해 expand class div가 렌더링된다.  
이 시점에서 이벤트가 전파되었을때 원래는 동작할 함수가 없었으나 expand class div의 click event listener가 동작하게 된다.  

결국, 이 문제를 해결하기 위해 nextTick의 동작을 항상 micro task로 수행하지 않고 특정 상황에서는 macro task로 수행되게 하는 수정이 2.5에서 진행되었다.  
 
이벤트 버블링을 통한 event listener들의 수행과 연속적인 이벤트(window resize event 또는 scroll 이벤트 등)의 event listener의 수행보다 DOM을 업데이트 하는 re-render 동작의 우선순위를 낮추도록 macro task로 nextTick을 수행하는 것이다.  

관련해서 Vue에서 event listener는 어떻게 구성되는지 코드를 보자.  

![macro task event listener](http://drive.google.com/uc?export=view&id=1NWeiXP8RlYxGzMOH3OKM_fgLhnFtJm8C)

24줄 : next-tick 모듈에 있는 withMacroTask함수는 event listener를 호출하는 wrapping 함수를 반환한다.  
8줄 : add함수는 wrapping된 함수를 events listener로 등록한다.
25줄 : 이벤트가 발생하면 먼저 next-tick의 전략을 macro task로 변경하고  
27줄 : 실제 event발생시 수행될 함수를 수행한다.  
29줄 : 이후 next-tick의 전략을 다시 micro task로 변경한다.  

결국 27줄이 수행되는 와중에 (여기서의 fn은 data를 변경하는 event listener이다.) 등록되는 next-tick에 전달되는 함수는 macro task로 수행된다.(watcher의 run(render 함수))  


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

[reactivity-in-depth]: deep-dive-into-reactivity-in-depth.md

[youtube-Philip-Roberts]: https://www.youtube.com/watch?v=8aGhZQkoFbQ

[grid-example-screenshot]: http://drive.google.com/uc?export=view&id=1bj6bFwAwGOgIB2V__tL5b7xqB6uO1PmL

[nexttick function]: http://drive.google.com/uc?export=view&id=1Dgul0JyrBXjQVScjTP5OUHH2uY8maon3
