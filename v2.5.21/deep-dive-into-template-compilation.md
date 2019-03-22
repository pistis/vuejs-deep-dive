> _이 문서는 [vue.js v2.5.21][git-1] 코드를 기반으로 작성되었습니다._  
> _이 문서는 [영문 공식가이드][guide-1]와 [한국어 공식가이드][guide-2]를 참고하였습니다._

---

# 프롤로그
> 템플릿을 컴파일 하는 세부 구현에 대해서는 서비스를 개발하는데에 필수적으로 알아야 하는 사항은 아니다.  
> 다만, 알고 있다면 Vue를 사용하다가 맞닥뜨리는 문제를 풀어나가는 디버깅 능력을 조금은 쌓을 수 있지 않을까?  
> (혹시 아는가? 작은 컨트리뷰션이라도 할 수 있을지...)  

## Template Compilation
Vue의 소스코드를 보면 `src/compiler` 라는 컴파일을 담당하는 모듈이 있다.  
`템플릿 컴파일이란 어떤 일을 하는 걸까?`  

살펴보기 전에 먼저, Vue는 다양한 방법으로 사용할 수 있는것을 알고 있을 것이다.  

[영문 공식가이드][guide-1]에 보면 다음과 같은 예시를 볼 수 있다.  
![explanation-of-different-builds][img-explanation-of-different-builds]

2 ~ 4줄 : template속성에 템플릿을 문자열로 지정하고 런타임시 컴파일하여 render 함수를 만들고 렌더링이 되도록 하는 방법  
7 ~ 11줄 : render 함수를 미리 만들어(build 단계) 런타임시 컴파일 과정없이 바로 렌더링 되도록 하는 방법 ([render function](https://vuejs.org/v2/guide/render-function.html)을 직접 만들어서 사용할 수도 있다.)

template 속성에 템플릿을 지정하는 방법과 비슷한 방법은 .vue확장자로된 싱글파일 컴포넌트로 개발하는 방법이다. (대부분 이 방식을 많이 사용할거라 생각한다.)  

Vue는 두가지의 번들 방식을 제공하고 있는데 Runtime + Compiler 방식과 Runtime-only 방식이다.  

vue-loader를 사용한다면 아마도 싱글 파일 컴포넌트를 사용하면서 빌드 단계에서 컴파일을 하도록 설정하여 사용하게 될 것이다.  
결국 프로덕션 코드에 로딩되는 vue 번들은 Runtime-only가 될 것이다.(Full에 비해 30% 파일 사이즈 축소)

만약 cdn을 통해 vue.js full version 번들을 바로 import하여 사용한다면 Full 버전인 Runtime + Compiler방식의 번들을 사용하게 될 것이다.  

`템플릿 컴파일이란 어떤 일을 하는 걸까?` 라는 질문에 대한 답을 알았다.  
즉, **template을 파싱하여 render function을 만들어 내는 과정**을 말한다.  

디렉토리 내의 모든 소스코드를 다 살펴보진 못하겠지만 Vue에서 템플릿을 컴파일 하는 과정을 살펴 본다.  

# Deep dive into 'Template Compilation'
## Flow
template -> render function -> virtual dom -> real dom의 전체 과정을 그림으로 먼저 보자.  
![flow-activity-diagram][flow-activity-diagram]

## 예제로 시작하자
![example-input-message][example-input-message]

아주 간단한 예제이다.  
static 한 h1태그가 있고(data의 변경에 의해 변경되지 않는 템플릿) v-model을 통해 msg내부 data의 반응형 속성을 변경하면 입력한 값과 그 값을 뒤집은 값을 p태그로 렌더링한다.  
여기에 정의된 template이 어떤 과정을 통해 render 함수로 컴파일 되는지 알아보자.  

## template이 compile되는 시점
첫번째로 Runtime + Compiler방식을 사용한다면 template은 컴포넌트의 mount 시점에 compile 된다.  
![code-vue-prototype-$mount][code-vue-prototype-$mount]

12 ~ 18줄 : `compileToFunctions` 함수에 template 문자열을 넘기면 compile 후 render함수와 staticRenderFns함수를 반환한다.  

두번째로 Runtime Only방식을 사용한다면 template은 빌드단계에서 vue-loader에 의해 compile된다.(위 entry-runtime-with-compiler.js의 $mount함수는 Runtime Only 번들에는 포함되지 않는다.)  

vue-loader는 vue-template-compiler npm package 모듈을 사용하여 template을 compile하는데 vue-template-compiler의 정체는 이것이다.  

```javascript
// vue.js의 build/config.js
// 생략...
// Web compiler (CommonJS).
  'web-compiler': {
    entry: resolve('web/entry-compiler.js'),
    dest: resolve('packages/vue-template-compiler/build.js'),
    format: 'cjs',
    external: Object.keys(require('../packages/vue-template-compiler/package.json').dependencies)
  },
  // Web compiler (UMD for in-browser use).
  'web-compiler-browser': {
    entry: resolve('web/entry-compiler.js'),
    dest: resolve('packages/vue-template-compiler/browser.js'),
    format: 'umd',
    env: 'development',
    moduleName: 'VueTemplateCompiler',
    plugins: [node(), cjs()]
  },
// 생략...
```
`build/config.js`는 vue가 빌드될 때 만들어지는 모든 모듈들의 entry 설정정보가 명시되어 있는 파일이다.  
이 중에 vue-template-compiler가 있다.  
즉, vue의 특정 버전이 빌드되서 릴리즈될때 마다 vue-template-compiler 모듈도 동일한 버전으로 빌드되어 npm에 배포된다.  

위에 정의된 `web/entry-compiler.js`는 `src/platforms/web/entry-compiler.js` 파일이며 이 파일이 빌드하는 compile 함수는 결국 `Vue.prototype.$mount` 함수에서 사용하는 `compileToFunctions` 함수와 동일한 함수이다.  

```javascript
// src/platforms/web/compiler/index.js

import { baseOptions } from './options'
import { createCompiler } from 'compiler/index'

const { compile, compileToFunctions } = createCompiler(baseOptions)

export { compile, compileToFunctions }
```
이 `src/platforms/web/compiler/index.js`에서 `compileToFunctions`함수를 export 한다.  

> 추가로 vue-loader는 싱글 파일 컴포넌트인 .vue 파일을 1차적으로 `src/sfc/parser.js`의 `parseComponent`함수로 파싱하여 template을 추출한다.  

## compileToFunctions 함수
위에서 살펴보았듯이 'Runtime + Compiler' 방식과 'Runtime Only' 방식 모두에서 사용하는 `compileToFunctions` 함수는 template을 compile 하여 render함수를 만들어 반환하는 역할을 한다.  

각 단계의 코드를 보자.(일부 코드는 지면상 생략하였다.)  
![to-function.js][to-function.js]
18 ~ 20줄 : 만약 특정 컴포넌트를 여러번 인스턴스화 한다면 결국 같은 내용의 template을 render로 만들게 된다. 그를 위해 cache를 해놓고 한번 만들어진 render 함수는 재사용한다.  
23줄 : 여기서 호출하는 compile함수의 핵심 로직은  `src/compiler/create-compiler.js` -> `src/compiler/index.js` 까지 거슬러 올라가면 볼 수 있는 `baseCompile` 함수이다.  

baseCompile 함수로 가보자.  

![compiler-index.js][compiler-index.js]
함수의 flow를 간략하게 나타내보면 다음과 같다.  

> 이제부터 나오는 AST는 Abstract syntax tree를 말한다.


6줄 : parse (src/compiler/parser/index.js) : template을 parsing하여 AST를 만든다.  
7줄 : optimize (src/compiler/optimizer.js) : 파싱된 AST를 최적화한다.(static 처리)  
8줄 : generate (src/compiler/codegen/index.js) : 최적화된 AST를 render와 staticRenderFns 함수로 만든다.  

예제와 같이 단계별로 살펴보자.  
먼저 예제의 template은 다음과 같다.  
![example-input-message][example-input-message]

### parse
이 template 문자열이 parse 함수를 통해 AST로 변환된다.  
parse가 호출되는 시점을 디버깅 해보면 다음과 같다.  
![parse-debug-step-1][parse-debug-step-1]
실제 컴포넌트의 template 속성의 문자열값과 동일한 값을 전달한다.  

parse함수로 들어가보자.
![compiler-parser-index.js][compiler-parser-index.js]
8줄 : 이 stack은 template을 parsing하는 동안 AST element를 임시로 담아두는 stack이다.  
9줄 : AST element의 root element이다.  
12줄 : parseHTML 함수(코드가 250 line가까이 되기 때문에 지면에 첨부하지 않는다.)는 template 문자열을 순차적으로 탐색하며 정규식으로 매칭한다. 그를 통해 tag(html tag와 component)와 각종 attribute정보(html attr과 vue 지원 속성들), comment, text node등의 정보를 추출한다.  
19줄/23줄/27줄/30줄 : parseHTML함수내에서는 정보를 추출하고 start/end/chars/comment 함수들을 호출하면서 전달한다. 이후 정보는 ast element로 변환된다.
1. start : AST element를 생성, vue 속성처리, vue component 생성 등
2. end : start에서 생성된 AST element의 후처리
3. chars : text node를 parsing하여 expression인 경우 함수형태의 eval이 가능한 문자열로 변환 후 AST element로 생성
4. comment : comment AST element로 생성

34줄 : parseHTML 과정이 모두 완료되면 root변수는 tree 구조의 AST element가 된다.  

ASTElement의 상세한 structure는 `flow/compiler.js`에 정의되어 있으며 type으로 구분되어 진다.  
- type 1 : DOM Node 또는 Vue Component, Slot, Template Tag를 나타낸다.
- type 2 : Expression을 나타낸다.
- type 3 : Text Node를 나타낸다.

#### ASTElement
예제의 최종 결과 ASTElement를 보자.  
(이해를 위해 예제를 간단하게 하였기에 flow/compiler.js에 정의된 속성들 대부분이 존재하지 않는다. 다만 가장 기본적인 값들은 볼 수 있다.)  

**먼저 root ASTElement이다.**  
![root-ast][root-ast]

예제 템플릿의 경우 root에 특별한 속성이 없기 때문에 심플하다.  
div DOM Node이기 때문에 type은 1이며 children에 h1, p, input등의 ASTElement등이 존재한다.  

템플릿과 비교하며 children ASTElement들을 하나씩 살펴보자.  

**h1 tag이다.**  
```html
<h1>Template Compilation</h1>
```
![h1-ast][h1-ast]
ASTText를 children으로 하나 가지고 있다.  
특별한 속성은 없다.  


**첫번째 p tag이며 동적으로 변경되는 요소이다.**  
```html
<p>Message : {{msg}}</p>
```
![first-p-ast][first-p-ast]
ASTExpression을 children으로 가지고 있다.  
특별한 속성은 없지만 expression과 text정보를 구분해서 가지고 있다.  

**두번째 p tag이며 동적으로 변경되는 요소이다.**  
```html
<p>Reversed Message : {{reversedMsg}}</p>
```
![second-p-ast][second-p-ast]
ASTExpression을 children으로 가지고 있다.  
특별한 속성은 없지만 expression과 text정보를 구분해서 가지고 있다.    

**마지막으로 input tag이다.**  
```html
<input v-model="msg" placeholder="Enter Message...">
```
![input-ast][input-ast]
attrs는 HTML의 속성을 나타내며 attrsList와 attrsMap은 모든(vue and html) 속성정보를 나타낸다.  
또한, v-model의 경우 vue에서 제공하는 기본 directive중 하나인데, directives속성을 통해 별도로 나타낸다.  

### optimize
`parse`함수를 통해 이렇게 만들어진 ASTElement는 `optimize`함수로 전달되어 최적화 과정을 거치게 된다.  
![optimize][optimize]

optimize 코드를 보자.  지면에 다 싣기에는 코드가 길어 일부만 포함하였다.
![optimize.js][optimize.js]
optimize가 하는 일은 동적으로 변하는 요소인지 아니면 static한 요소인지를 판별하여 ASTNode(ASTElement, ASTExpression, ASTText)의 다음 속성들의 값을 결정하는 일을 한다.  
```javascript
static?: boolean;
staticRoot?: boolean;
staticInFor?: boolean;
staticProcessed?: boolean;
```

### generate
마지막으로 `generate` 함수로 최적화된 ASTElement를 전달하여 render와 staticRenderFns 함수를 생성한다.  

![generate][generate]
![generate.js][generate.js]
7줄 : genElement를 통해 최종 render함수의 로직을 문자열로 만든다.  
9줄 : 차후 이 문자열을 eval로 실행하게 된다.  

#### render function
최종적으로 만들어진 render 함수를 보자.  
![render][render]

28 ~ 47줄 : _c 헬퍼함수는 virtual dom을 생성하는 함수이다.  
49 ~ 67줄 : _s, _v 헬퍼함수와 그 외 다수의 render 헬퍼함수가 정의되어 있다.  
2 ~ 26줄 : eval로 실행될 render함수의 문자열 형태이다. _c, _s, _v등 render helper 함수들로 이루어져 있다. eval로 실행되면 render함수는 virtual dom을 생성한다.  

## 마치며
template -> render -> virtual dom -> Real DOM의 flow를 모두 다루고 싶었지만 내용이 길어져 template -> render만 먼저 살펴보았다.  

P.S
문서의 내용중 잘못된 부분이나 개선이 필요한 부분이 있다면 피드백 해주시면 좋을 거 같습니다. vamalboro@gmail.com

Written by 피스티스.

---

[git-1]: https://github.com/vuejs/vue/tree/v2.5.21

[guide-1]: https://vuejs.org/v2/guide/installation.html#Runtime-Compiler-vs-Runtime-only

[guide-2]: https://kr.vuejs.org/v2/guide/installation.html#%EA%B0%81-%EB%8B%A4%EB%A5%B8-%EB%B9%8C%EB%93%9C%EA%B0%84-%EC%B0%A8%EC%9D%B4%EC%A0%90

[cdn]: https://cdnjs.com/libraries/vue

[img-explanation-of-different-builds]: http://drive.google.com/uc?export=view&id=1PiQHwMJRpAsg8EQGdAYmNf-6tMcD_bjF

[example-input-message]: http://drive.google.com/uc?export=view&id=1yYCGf_MT8w4OVx7maMim3i2nThJ6-dYz

[code-vue-prototype-$mount]: http://drive.google.com/uc?export=view&id=1uVIsXnFqaVLhwbEx5gIFCd7xMpGMg0JG

[to-function.js]: http://drive.google.com/uc?export=view&id=1hsqt5fgxUj9LvWMj9XXxYtkxWFVXvXNh

[compiler-index.js]: http://drive.google.com/uc?export=view&id=1f5jlW0EubMxmSg-V8kDzkUGVOlkCsGNN

[parse-debug-step-1]: http://drive.google.com/uc?export=view&id=197MiFAGnwf3SVyuvscIRwYNHkI96UR1o

[compiler-parser-index.js]: http://drive.google.com/uc?export=view&id=1Oa613N7oIE01sJs_QJZ1kzGs9dU_E41y

[root-ast]: http://drive.google.com/uc?export=view&id=1gEMXR0o_gDYmBov5TS4ynGtODJd2wLNF

[h1-ast]: http://drive.google.com/uc?export=view&id=1CHcIucCJpN_vuEdOgXVGnIpqEwIPfVCN

[first-p-ast]: http://drive.google.com/uc?export=view&id=1RiL1mVSoSkqfM3deOL3hrdzsGOVvPEEE

[second-p-ast]: http://drive.google.com/uc?export=view&id=1s9haoqSqzJ1jzjBzhszs6fIEWBeiuYQt

[input-ast]: http://drive.google.com/uc?export=view&id=1qp6sZ5NVgt-sIEiiyUE7WA3S4DYWol16

[optimize]: http://drive.google.com/uc?export=view&id=1UGaB58UcxmPVlvqtZLAh8XSikYBZHobj

[optimize.js]: http://drive.google.com/uc?export=view&id=16G56u5u-ox68Qzw_R6pZpNXsB7lLfYYV

[generate]: http://drive.google.com/uc?export=view&id=1_y7F5McdMckUc08S3JKqRhQVN2m1DMBF

[generate.js]: http://drive.google.com/uc?export=view&id=1Rdc5T2WMZ_s9DLXSyy60ggfBesE68jlP

[render]: http://drive.google.com/uc?export=view&id=1-2TpfEhECl5Y_g4yjvRejEo1kPTQj-NN

[flow-activity-diagram]: http://drive.google.com/uc?export=view&id=1HefuVpCm4R9_KNnOpA2dAI2G_rTO8rFB
