# Render 함수

Vue는 대부분의 경우 템플릿을 사용하여 애플리케이션을 구축할 것을 권장합니다. 그러나 JavaScript가 완전히 필요한 상황이 있습니다. 그럴 땐 **render 함수**를 사용할 수 있습니다.

Let's dive into an example where a `render()` function would be practical. Say we want to generate anchored headings:
{code0}render(){/code0} 함수가 실용적일 수 있는지 예제를 봅시다. 링크가 달린 헤드라인(Anchored Heading) 을 만들고 싶다면: 

```html
<h1>
  <a name="hello-world" href="#hello-world">
    Hello world!
  </a>
</h1>
```

링크가 달린 헤드라인은 매우 자주 사용되기 때문에 컴포넌트로 만들어 둘수 있습니다:

```vue-html
<anchored-heading :level="1">Hello world!</anchored-heading>
```
헤더라인은 `level` prop을 가지고 만들어야 하기 때문에 금방 이런 형태로 가게 됩니다: 

```js
const app = Vue.createApp({})

app.component('anchored-heading', {
  template: `
    <h1 v-if="level === 1">
      <slot></slot>
    </h1>
    <h2 v-else-if="level === 2">
      <slot></slot>
    </h2>
    <h3 v-else-if="level === 3">
      <slot></slot>
    </h3>
    <h4 v-else-if="level === 4">
      <slot></slot>
    </h4>
    <h5 v-else-if="level === 5">
      <slot></slot>
    </h5>
    <h6 v-else-if="level === 6">
      <slot></slot>
    </h6>
  `,
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

This template doesn't feel great. It's not only verbose, but we're duplicating `<slot></slot>` for every heading level. And when we add the anchor element, we have to again duplicate it in every `v-if/v-else-if` branch.
이 템플릿은 그다지 맘에 들지 않습니다. 장황할 뿐만 아니라, 머리말 레벨마다 `<slot></slot>`을 중복해서 사용하고 있습니다. 여기에 링크(Anchor)를 추가해야 하는데, `v-if/v-else-if` 분기 마다 또 중복된 코드를 추가해야 하죠. 

While templates work great for most components, it's clear that this isn't one of them. So let's try rewriting it with a `render()` function:
템플릿은 대부분의 컴포넌트를 만들때 잘 적용되지만, 여기에서는 그렇지 못하네요.  `render()` 함수를 이용해서 다시 작성해 봅시다:

```js
const app = Vue.createApp({})

app.component('anchored-heading', {
  render() {
    const { h } = Vue

    return h(
      'h' + this.level, // 태그명
      {}, // props와 속성props/attributes
      this.$slots.default() // 하위 컴포넌트 배열
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

`render()` 함수로 구현할때 훨신 단순해집니다. 하지만 컴포넌트 인스턴트 프로퍼티에 대해 훨씬 더 잘 알아야만 하죠. 여기에서 `Hello world!` 를 `anchored-heading` 안에 전달하는 것처럼, `v-slot` 지시문(directive)이 없는 자식 컴포넌트를 컴포넌트에 전달하면, 그 자식 컴포넌트는 `$slots.default()`를 이용해 접근할수 있습니다. **잘 모르겠다면, `instance properties API` 를 render 함수를 살펴보기 전에 읽어 보는 것이 좋습니다.**

## DOM 트리

렌더 기능에 대해 알아보기 전에, 조금은 브라우저가 어떻게 작동하는지 알아 보는 것이 중요합니다. 아래 HTML 예제를 봅시다.

```html
<div>
  <h1>제목</h1>
  문장이 여기에 들어갑니다.
  <!-- TODO: Add tagline -->
</div>
```

브라우저가 이 코드를 읽게 되면, 모든 내용을 추적하기 위해 ["DOM 노드" 트리](https://javascript.info/dom-nodes) 를 만듭니다.

위 HTML의 DOM 노드 트리는 아래와 같습니다.

![DOM Tree Visualization](/images/dom-tree.png)

모든 엘리먼트는 노드(Node)입니다. 모든 텍스트도 하나의 노드입니다. 심지어 주석도 노드입니다! 각 노드는 자식을 가질 수 있습니다. (즉, 각 노드는 다른 노드를 포함할 수 있습니다).

Updating all these nodes efficiently can be difficult, but thankfully, we never have to do it manually. Instead, we tell Vue what HTML we want on the page, in a template:
모든 노드를 효율적으로 갱신하는 것은 어려습니다만, 다행히 우리는 수동으로 업데이트할 필요가 없습니다. 페이지에서 수정하고 싶은 HTML을 템플릿에 작성하면 Vue가 대신 해줍니다.

```html
<h1>{{ blogTitle }}</h1>
```

또는 렌더 함수에서 아래와 같이 작성합니다:

```js
render() {
  return Vue.h('h1', {}, this.blogTitle)
}
```

두가지 경우 모두, Vue는 페이지를 최신버전으로 갱신하며,   `blogTitle`이 바뀌어도 마찬가지입니다. 

## 가상 DOM 트리

Vue는 실제 DOM에서의 변경사항을 추적하기 위해 **가상(Virtual) DOM**을 만들어 페이지를 갱신합니다. 이를 자세히 살펴보면 아래와 같습니다"

```js
return Vue.h('h1', {}, this.blogTitle)
```

위의 코드에서 `h()` 함수가 반환하는 것은 무엇일까요? DOM 같은 것을 반환하는 것으로 보이지만 _정확히_  실제의 DOM은 아닙니다. 여기에서 반환되는 객체는  모든 하위 노드의 설명을 포함하여, 페이지에 렌더링해야하는 노드의 정보를 Vue에 설명하는 정보를 가집니다. 우리는 이 노드 기술(Description)을 "가상노드(Virtual node)"라고 부르며, 약어로써 **VNode** 를 사용합니다. 가상 DOM"은 Vue 컴포넌트의 트리로 구축된 VNodes 전체트리를 말합니다.

## `h()` 전달인자(Argument)

`h()`는  VNode를 생성하는 유틸리티 함수입니다. 아마 더 정확한 이름은 `createVNode()` 이 맞겠지만, 매우 자주 사용되기 때문에 간결하게 하기 위해 `h()`  라고 이름 지었습니다.  이 함수는 3개의 인자를 받습니다.

```js
// @returns {VNode}
h(
  // {String | Object | Function } 태그
  // HTML 태그명 또는 컴포넌트, 비동기 컴포넌트
  // null을 반환하면 주석이 됩니다. 
  //
  // 필수.
  'div',

  // {Object} props
  // 속성, 프로퍼티,이벤트에 해당하는 객체.
  // 템플릿에서 사용합니다. 
  //
  // 옵션.
  {},

  // {String | Array | Object} 하위 vnode
  // `h ()`를 사용하여 만들어졌거나, 
  // 'text VNodes'를 만들어내는 문자열,  
  // 슬롯이있는 객체를 가져 오는 하위 VNode.
  //
  // 옵션
  [
    'Some text comes first.',
    h('h1', 'A headline'),
    h(MyComponent, {
      someProp: 'foobar'
    })
  ]
)
```

## 완전한 예제

이상의 지식을 바탕으로 우리 컴포넌트를 완전히 만들어 봅시다:

```js
const app = Vue.createApp({})

/** 하위 노드에서 재귀적으로 문자열을 가져옵니다.  */
function getChildrenTextContent(children) {
  return children
    .map(node => {
      return typeof node.children === 'string'
        ? node.children
        : Array.isArray(node.children)
        ? getChildrenTextContent(node.children)
        : ''
    })
    .join('')
}

app.component('anchored-heading', {
  render() {
    // 하위 노드의 문자열 컨텐츠를 기반으로 kebab-case 형태의 id를 만듭니다. 
    const headingId = getChildrenTextContent(this.$slots.default())
      .toLowerCase()
      .replace(/\W+/g, '-') // replace non-word characters with dash
      .replace(/(^-|-$)/g, '') // remove leading and trailing dashes

    return Vue.h('h' + this.level, [
      Vue.h(
        'a',
        {
          name: headingId,
          href: '#' + headingId
        },
        this.$slots.default()
      )
    ])
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

## 제약 사항

### VNodes는 고유해야 합니다.

컴포넌트 트리의 모든 VNode는 고유해야 합니다. 아래 예제는 다음 렌더 함수가 잘못된 경우입니다:

```js
render() {
  const myParagraphVNode = Vue.h('p', 'hi')
  return Vue.h('div', [
    // 악! 중복된 VNode입니다!
    myParagraphVNode, myParagraphVNode
  ])
}
```


같은 엘리먼트/컴포넌트를 여러 개 만들려면 팩토리 함수를 사용해서 복제할 수 있습니다. 예를 들어, 다음의 렌더 함수는 20개의 동일한 p태그를 렌더링하는 완벽한 방법입니다:

```js
render() {
  return Vue.h('div',
    Array.apply(null, { length: 20 }).map(() => {
      return Vue.h('p', 'hi')
    })
  )
}
```

## Template기능을 일반 JavaScript로 변경하기

### `v-if` 와 `v-for`

if든, for든 자바스크립트에서 쉽게 해낼수 있는 것이기 때문에 vue 렌더 함수에서 별달리 제공 하는것은 없습니다. `v-if`와  `v-for` 를 사용하는 템플릿을 예로 들어보겠습니다:

```html
<ul v-if="items.length">
  <li v-for="item in items">{{ item.name }}</li>
</ul>
<p v-else>No items found.</p>
```

위 내용을 JS의 `if`/`else`과 `map` 를 사용하는 렌더 함수로 만들어 보겠습니다:

```js
props: ['items'],
render() {
  if (this.items.length) {
    return Vue.h('ul', this.items.map((item) => {
      return Vue.h('li', item.name)
    }))
  } else {
    return Vue.h('p', 'No items found.')
  }
}
```

### `v-model`

`v-model` 디렉티브는 `modelValue`와  `onUpdate:modelValue` prop들을 이용하여 확장되는데, 템플릿 컴파일 과정에서  우리가 직접 제공해주어야 합니다.

```js
props: ['modelValue'],
render() {
  return Vue.h(SomeComponent, {
    modelValue: this.modelValue,
    'onUpdate:modelValue': value => this.$emit('update:modelValue', value)
  })
}
```

### `v-on`

이벤트 핸들러를 위해 적절한 prop 이름을  주어야 합니다. 예. `click` 이벤트를 처리하기 위해서는 `onClick`을 prop 명으로 사용해야 합니다.

```js
render() {
  return Vue.h('div', {
    onClick: $event => console.log('clicked', $event.target)
  })
}
```

#### 이벤트 수식어

`.passive`, `.capture`와 `.once`  이벤트 수식어의 경우, 카멜케이스(camelCase)를 사용하여 이벤트명 뒤에 연결할 수 있습니다.

예 :

```javascript
render() {
  return Vue.h('input', {
    onClickCapture: this.doThisInCapturingMode,
    onKeyupOnce: this.doThisOnce,
    onMouseoverOnceCapture: this.doThisOnceInCapturingMode,
  })
}
```

For all other event and key modifiers, no special API is necessary, because we can use event methods in the handler:
다른 이벤트와 키 수식어를 처리하기 위해 별도의 API가 필요하지는 않습니다. 그저 이벤트 처리를 위한 메소드를 사용하시면 됩니다.

| 수식어                                           | 동일기능을 하는 핸들러 r                                                                                                |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `.stop`                                               | `event.stopPropagation()`                                                                                            |
| `.prevent`                                            | `event.preventDefault()`                                                                                             |
| `.self`                                               | `if (event.target !== event.currentTarget) return`                                                                   |
| Keys:<br>`.enter`, `.13`                              | `if (event.keyCode !== 13) return` (change `13` to [another key code](http://keycode.info/) for other key modifiers) |
| Modifiers Keys:<br>`.ctrl`, `.alt`, `.shift`, `.meta` | `if (!event.ctrlKey) return` (change `ctrlKey` to `altKey`, `shiftKey`, or `metaKey`, respectively)                  |

다음은 이러한 모든 수식어를 함께 사용하는 예입니다:


```js
render() {
  return Vue.h('input', {
    onKeyUp: event => {
      // Abort if the element emitting the event is not
      // the element the event is bound to
      if (event.target !== event.currentTarget) return
      // Abort if the key that went up is not the enter
      // key (13) and the shift key was not held down
      // at the same time
      if (!event.shiftKey || event.keyCode !== 13) return
      // Stop event propagation
      event.stopPropagation()
      // Prevent the default keyup handler for this element
      event.preventDefault()
      // ...
    }
  })
}
```

### Slots

You can access slot contents as Arrays of VNodes from [`this.$slots`](../api/instance-properties.html#slots):

```js
render() {
  // `<div><slot></slot></div>`
  return Vue.h('div', {}, this.$slots.default())
}
```

```js
props: ['message'],
render() {
  // `<div><slot :text="message"></slot></div>`
  return Vue.h('div', {}, this.$slots.default({
    text: this.message
  }))
}
```

렌더 함수를 사용해서 슬롯을 하위 컴포넌트로 전달하려면:

```js
render() {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return Vue.h('div', [
    Vue.h('child', {}, {
      // pass `slots` as the children object
      // in the form of { name: props => VNode | Array<VNode> }
      default: (props) => Vue.h('span', props.text)
    })
  ])
}
```

## JSX

`render` 함수 안에서 많은 코드를 작성해야 한다면 좀 고통스러울수 있습니다.

```js
Vue.h(
  Vue.resolveComponent('anchored-heading'),
  {
    level: 1
  },
  [Vue.h('span', 'Hello'), ' world!']
)
```

특히, 템플릿으로 이 기능을 만들었을때 너무 간결해서 비교가 심하게 되면 더더욱 그럴겁니다.

```vue-html
<anchored-heading :level="1"> <span>Hello</span> world! </anchored-heading>
```

바로 이런 경우를 위해 JSX를 뷰와 함게 사용하기 위해 [Babel plugin](https://github.com/vuejs/jsx-next) 를 적용해서, 좀더 템플릿 코드와 비슷하게 만들수 있게됩니다:

```jsx
import AnchoredHeading from './AnchoredHeading.vue'

const app = createApp({
  render() {
    return (
      <AnchoredHeading level={1}>
        <span>Hello</span> world!
      </AnchoredHeading>
    )
  }
})

app.mount('#demo')
```

JSX가 JavaScript에 매핑하는 방법에 대한 자세한 내용은 [사용 문서](https://github.com/vuejs/jsx-next#installation) 를 참조하십시오.

## 템플릿 컴파일

Vue의 템플릿이 실제로 render 함수로 컴파일 되는지 알고 싶을 것입니다. 이는 일반적으로 알 필요가 없는 내부 구현 사항이지만, 특정 템플릿 기능이 어떻게 컴파일 되는지 알고 싶다면, 흥미로울 수 있습니다. 다음은 `Vue.compile` 을 사용하여 템플릿 문자열을 라이브 컴파일하는 데모입니다.

<iframe src="https://vue-next-template-explorer.netlify.app/" width="100%" height="420"></iframe>