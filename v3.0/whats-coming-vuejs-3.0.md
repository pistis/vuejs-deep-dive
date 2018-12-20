# What's coming vue js 3.0

- [Summary](#summary)
- [Detail Changes](#detail-changes)
  - [High-Level API Changes Overview](#high-level-api-changes-overview)
  - [Faster](#faster)
  - [Smaller](#smaller)
  - [Other Runtime Improvement](#other-runtime-improvement)
  - [Compiler Improvements](#compiler-improvements)
  - [IE 11 Support](#ie-11-support)
  - [Easier to use and maintain](#easier-to-use-and-maintain)
  - [Roadmap](#roadmap)
- [References](#references)  


## Summary
* Proxy기반의 observation
* TypeScript로 코드 재작성
* Virtual DOM re-written(improve compiler)
* 2.x 호환
* small size(20KB -> 10KB)

## Detail Changes
### High-Level API Changes Overview
* template 구문은 99% 동일하게 유지
    * slot의 경우 약간의 API 변경 가능성 존재
* ES2015
    * class 기반으로 재 작성
    * TypeScript로 개발
* 2.x의 Object 기반의 컴포넌트도 호환 (컴파일될 때 class로 자동 변경)
* mixin 지속 지원
* vdom의 개선으로 인해 render함수를 직접 사용하는 앱의 경우 변경사항이 발생 가능성 존재

### Faster
* Virtual Dom re-written : mounting and patching 성능 100% 향상
* Optimized Slots Generation
    * 부모와 자식을 별도로 re-rendering (불필요한 parent,child를 모두 렌더링 하는 것이 제거)
* Static Props Hoisting
    * 전체 트리 패치를 skip
    * 노드 자체에 patch를 적용하지 않고 자식 노드를 직접 patch
* proxy-based observation
    * ES2015 Proxy를 사용 (Object.defineProperty의 제약을 없애고 더 많은 기능과 성능을 제공)
    * Vue의 reactivity system의 속도를 두배로 향상, 메모리는 절반을 사용 (결과적으로 부팅 속도 100% 향상)
    * 하위호환성도 지원(IE11) - Object.defineProperty로 transfiled
    * 특징
        * lazy obserbing과 독립적인 API 제공(Vue를 사용하지 않아도 사용할 수 있는 형태)
        * 최소화된 변경 알림
            * Vue.set 등 사용시 2.x는 객체를 감시하는 옵저버가 재평가, 3.x는 속성과 관련이 있는 옵저버만 재평가되도록 개선
* 디버깅
    * renderTracked, renderTriggered hook 제공 : 렌더링이 되는 시기를 추적 및 원인을 감지 가능

### Smaller
* AS-IS : 20KB, TO-BE : gzipped 10KB
    * 기본적으로 Tree Shaking이 적용

### Other Runtime Improvement
* 기본 built-in component를 위한 tree-shaking
    * keep-alive, transition, transition-group, v-model, v-show…(v2.5.21 기준)
* Fragments & Portals
    * Fragments : 여러 루트 노드를 반환하는 구성요소 (react의 Fragment와 같은 기능으로 예상)
    * Portals : 컴포넌트의 하위가 아닌 DOM의 다른 영역에 하위 트리를 렌더링하는 기능 제공
* Improved slots mechanism
    * child component의 render함수가 call될때 slot도 호출 (slot은 3.0부터는 모두 함수로 컴파일)
    * 슬롯의 종속성이 상위 -> 하위로 변경
        * 슬롯 내용이 변경되면 자식만 re-render
        * 부모가 렌더링 될때 자식은 슬롯 내용을 변경하지 않아도 된다. (종속 대상의 변경으로 인해)
        * 이로 인해 구성요소 트리수준에서 더욱 정확한 변경감지를 제공하여 쓸데없은 re-render 비용을 제거
* Custom Renderer API
    * Weex, NativeScript Vue등의 render-to-native project들의 vue의 최신 변경사항을 적용 유지하는데에 쉬워진다.

### Compiler Improvements
* tree-shaking
* 더 많은 AOT 최적화
    * Virtual Dom re-written
        * children normalization을 회피하기 위한 런타임 컴파일 힌트
        * static props hoisting, static tree hoisting과 같은 컴파일 타임 최적화 개선
        * VNode creation fast paths
* 더 나은 오류정보 및 소스맵을 지원하는 parser
    * template 컴파일 오류 위치 제공 등

### IE 11 Support
* 별도의 빌드를 통해 IE11까지 지원되는 형태 제공

### Easier to use and maintain
* Support TypeScript
* 패키지 분리 및 모듈화 (Observer 모듈 등 독립적인 패키지 및 라이브러리)
* iOS, Android, Web 모든 환경에서 사용할 수 있도록 변경(Custom Renderer API ??)

### Roadmap
* Internal feedback for the runtime prototype.
    * 런타입 프로토타입이 실제 동작하는지 확인 (현재 내부적으로 이 단계, 2018-11월)
    * v3.0를 릴리즈 하는 시점에 쉽게 마이그레이션 및 사용할 수 있는 중요한 라이브러리를 같이 준비
* Public feedback on breaking changes.
    * 공개 피드백
* Compatible features for 2.x and 2.x-next.
    * 2.x 호환성 확인
    * 2.x-next를 통해 기존 2.x로 만들어진 코드의 점진적인 마이그레이션을 지원
    * 2.x의 마지막 LTS 릴리즈는 3.0릴리즈 이후 18개월 동안 버그/보안 fix 지원
        * 현 시점 2.5.21 버전이 마지막 버전
* Alpha phase
    * 주요 목표 : 컴파일러와 SSR부분을 마무리 한 후안정성 확인 목적의 단계를 제공
* Beta phase
    * 주요 목표 : Vue Router, Vuex, Vue CLI, Vue DevTools등의 도구 업데이트 및 검증
* RC phase
    * 주요 목표 : API와 code base의 안정성 검증 이후 단계
* IE11 build
    * 2.x의 마지막 LTS 릴리즈 직전에 진행
* Final Release
    * 날짜 미정, 2019년 언젠가 될 가능성이 높다. 현재는 견고하고 안정성있는 코드를 만드는데에 집중
	
### References
- https://medium.com/the-vue-point/plans-for-the-next-iteration-of-vue-js-777ffea6fabf Evan You Blog
- https://jaxenter.com/road-vue-js-3-0-150147.html News Article
- https://jaxenter.com/whats-coming-vue-js-3-0-152107.html News Article
- https://www.youtube.com/watch?utm_campaign=Vue.js+News&utm_medium=email&utm_source=Revue+newsletter&v=XkOMOeEAFQI Evan You Presentation Youtube Video
  - https://docs.google.com/presentation/d/1yhPGyhQrJcpJI2ZFvBme3pGKaGNiLi709c37svivv0o/edit#slide=id.g42acc26207_0_156 Presentation Document
