- Start Date: 2019-01-14
- Target Major Version: 2.x & 3.x
- Reference Issues: https://github.com/vuejs/vue/issues/7740, https://github.com/vuejs/vue/issues/9180, https://github.com/vuejs/vue/issues/9306
- Implementation PR: (leave this empty)

# Summary 摘要

Introducing a new syntax for scoped slots usage:

为作用域插槽引入新的适用语法:

- New directive `v-slot` that unifies `slot` and `slot-scope` in a single directive syntax.

- 以新的 `v-slot` 指令将 `slot` 和 `slot-scope` 结合在一起.

- Shorthand for `v-slot` that can potentially unify the usage of both scoped and normal slots.

- `v-slot` 的简写，可以潜在地将作用域和普通插槽的用法统一.

# Basic example 基础示例

Using `v-slot` to declare the props passed to the scoped slots of `<foo>`:

使用 `v-slot` 来声明传递给 `<foo>` 的作用域插槽的props:

``` html
<!-- default slot 默认插槽 -->
<foo v-slot="{ msg }">
  {{ msg }}
</foo>

<!-- named slot 具名插槽-->
<foo>
  <template v-slot:one="{ msg }">
    {{ msg }}
  </template>
</foo>
```

# Motivation 动机

When we first introduced scoped slots, it was verbose because it required always using `<template slot-scope>`:

当我们第一次引入作用域的插槽时，它很冗长，因为它总是需要使用 `<template slot-scope>`:

``` html
<foo>
  <template slot-scope="{ msg }">
    <div>{{ msg }}</div>
  </template>
</foo>
```

To make it less verbose, in 2.5 we introduced the ability to use `slot-scope` directly on the slot element:

为了使 `slot` 语法更简洁，在Vue 2.5 版本中，我们引入了直接在slot元素上使用 `slot-scope` 的功能:

``` html
<foo>
  <div slot-scope="{ msg }">
    {{ msg }}
  </div>
</foo>
```

This means it works on component as slot as well:

这意味着 `slot` 也可以作为插槽使用在组件上：

``` html
<foo>
  <bar slot-scope="{ msg }">
    {{ msg }}
  </bar>
</foo>
```

However, the above usage leads to a problem: the placement of `slot-scope` doesn't always clearly reflect which component is actually providing the scope variable. Here `slot-scope` is placed on the `<bar>` component, but it's actually defining a scope variable provided by the default slot of `<foo>`.

但是, 上面的用法导致了一个问题: `slot-scope` 的位置并不总是明确地反映出哪个组件实际上在提供作用域变量( scope variable ). 这里的 `slot-scope` 放置在 `<bar>` 组件上, 但实际上是 `<foo>` 的 默认插槽(`default scope`) 在提供作用域变量.

This gets worse as the nesting deepens:

随着嵌套的加深, 情况变得更糟:

``` html
<foo>
  <bar slot-scope="foo">
    <baz slot-scope="bar">
      <div slot-scope="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </div>
    </baz>
  </bar>
</foo>
```

It's not immediately clear which component is providing which variable in this template.

这样并不能明确的表达哪个组件在此模板中提供哪个变量.

Someone suggested that we should allow using `slot-scope` on a component itself to denote its default slot's scope:

有人建议我们应该允许在组件本身上使用 `slot-scope` 来表示其默认插槽的范围:

``` html
<foo slot-scope="foo">
  {{ foo }}
</foo>
```

Unfortunately, this cannot work as it would lead to ambiguity with component nesting:

不幸的是, 这将导致组件嵌套的歧义, 从而无法正常工作:

``` html
<parent>
  <foo slot-scope="foo"> <!-- provided by parent or by foo? -->
    {{ foo }}
  </foo>
</parent>
```

This is why I now believe allowing using `slot-scope` without a template was a mistake.

这就是为什么我现在认为, 允许在非 `template` 标签上使用 `slot-scope` 是一个错误.

### Why a new directive instead of fixing `slot-scope`? 为什么用一个新的指令而不是继续使用 `slot-scope`?

If we can go back in time, I would probably change the semantics of `slot-scope` - but:

如果我们可以回到过去, 我可能会改变 `slot-scope` 的语义, 但:

1. That would be a breaking change now, and that means we will never be able to ship it in 2.x.

1. 它将会是一个非常大的变更. 并且意味着我们永远不可能在 `2.x` 版本中发布这个功能.

2. Even if we change it in 3.x, changing the semantics of existing syntax can cause a LOT of confusion for future learners that Google into outdated learning materials. We definitely want to avoid that. So, we have to introduce something new to differentiate from `slot-scope`.

2. 即使我们在 `3.x` 版本中修改, 更改现有语法的语义可能会使将来的学习者感到困惑, 因为他们将通过Google获得过时的学习材料. 我们希望尽可能的避免这种状况, 所以, 我们需要引入一个新的语法, 来区分 `slot-scope`.

3. In 3.x, we plan to unify slot types so it's no longer necessary to differentiate between scoped vs. non-scoped slots (conceptually). A slot may or may not receive props, but they are all just slots. With this conceptual unification, having `slot` and `slot-scope` being two special attributes seems unnecessary, and it would be nice to unify the syntax under a single construct as well.

3. 在 `3.x` 版本中, 我们计划统一插槽类型, 从此不再需要（从概念上）区分作用域插槽和非作用域插槽. 一个 `slot` 可能会也可能不会接收  `props`, 但他们都是 `slot`. 通过这种概念上的统一，将 `slot` 和 `slot-scope` 作为两个特殊属性似乎是不必要的，并且最好在单一结构下统一语法。

# Detailed design 详细设计

### Introducing a new directive: `v-slot`. 引入一个新的指令 `v-slot`.

- It can be used on `<template>` slot containers to denote slots passed to a component, where the slot name is expressed via the **directive argument**:

-

  ``` html
  <foo>
    <template v-slot:header>
      <div class="header"></div>
    </template>

    <template v-slot:body>
      <div class="body"></div>
    </template>

    <template v-slot:footer>
      <div class="footer"></div>
    </template>
  </foo>
  ```

  If any slot is a scoped slot which receives props, the received slot props can be declared using the directive's attribute value. The value of `v-slot` works the same way as `slot-scope`, so JavaScript argument destructuring is supported:

  ``` html
  <foo>
    <template v-slot:header="{ msg }">
      <div class="header">
        Message from header slot: {{ msg }}
      </div>
    </template>
  </foo>
  ```

- `v-slot` can be used directly on a component, without an argument, to indicate that the component's default slot is a scoped slot, and that props passed to the default slot should be available as the variable declared in its attribute value:

  ``` html
  <foo v-slot="{ msg }">
    {{ msg }}
  </foo>
  ```

### Comparison: New vs. Old

Let's review whether this proposal achieves our goals outlined above:

- Still provide succinct syntax for most common use cases of scoped slots (single default slot):

  ``` html
  <foo v-slot="{ msg }">{{ msg }}</foo>
  ```

- Clearer connection between scoped variable and the component that is providing it:

  Let's take another look at the deep-nesting example using current syntax (`slot-scope`) - notice how slot scope variables provided by `<foo>` is declared on `<bar>`, and the variable provided by `<bar>` is declared on `<baz>`...

  ``` html
  <foo>
    <bar slot-scope="foo">
      <baz slot-scope="bar">
        <div slot-scope="baz">
          {{ foo }} {{ bar }} {{ baz }}
        </div>
      </baz>
    </bar>
  </foo>
  ```

  This is the equivalent using the new syntax:

  ``` html
  <foo v-slot="foo">
    <bar v-slot="bar">
      <baz v-slot="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </baz>
    </bar>
  </foo>
  ```

  Notice that **the scope variable provided by a component is also declared on that component itself**. The new syntax shows a clearer connection between slot variable declaration and the component providing the variable.

### More Usage Comparisons

#### Default slot with text

``` html
<!-- old -->
<foo>
  <template slot-scope="{ msg }">
    {{ msg }}
  </template>
</foo>

<!-- new -->
<foo v-slot="{ msg }">
  {{ msg }}
</foo>
```

#### Default slot with element

``` html
<!-- old -->
<foo>
  <div slot-scope="{ msg }">
    {{ msg }}
  </div>
</foo>

<!-- new -->
<foo v-slot="{ msg }">
  <div>
    {{ msg }}
  </div>
</foo>
```

#### Nested default slots

``` html
<!-- old -->
<foo>
  <bar slot-scope="foo">
    <baz slot-scope="bar">
      <template slot-scope="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </template>
    </baz>
  </bar>
</foo>

<!-- new -->
<foo v-slot="foo">
  <bar v-slot="bar">
    <baz v-slot="baz">
      {{ foo }} {{ bar }} {{ baz }}
    </baz>
  </bar>
</foo>
```

#### Named slots

``` html
<!-- old -->
<foo>
  <template slot="one" slot-scope="{ msg }">
    text slot: {{ msg }}
  </template>

  <div slot="two" slot-scope="{ msg }">
    element slot: {{ msg }}
  </div>
</foo>

<!-- new -->
<foo>
  <template v-slot:one="{ msg }">
    text slot: {{ msg }}
  </template>

  <template v-slot:two="{ msg }">
    <div>
      element slot: {{ msg }}
    </div>
  </template>
</foo>
```

#### Nested & mixed usage of named / default slots

``` html
<!-- old -->
<foo>
  <bar slot="one" slot-scope="one">
    <div slot-scope="bar">
      {{ one }} {{ bar }}
    </div>
  </bar>

  <bar slot="two" slot-scope="two">
    <div slot-scope="bar">
      {{ two }} {{ bar }}
    </div>
  </bar>
</foo>

<!-- new -->
<foo>
  <template v-slot:one="one">
    <bar v-slot="bar">
      <div>{{ one }} {{ bar }}</div>
    </bar>
  </template>

  <template v-slot:two="two">
    <bar v-slot="bar">
      <div>{{ two }} {{ bar }}</div>
    </bar>
  </template>
</foo>
```

# Drawbacks

- Introducing a new syntax introduces churn and makes a lot of learning materials covering this topic in the ecosystem outdated. New users may get confused by discovering a new syntax after going through existing tutorials.

  - We need good documentation updates on scoped slots to help with this.

- The default slot usage `v-slot="{ msg }"` doesn't precisely convey the concept that `msg` is being passed to the slot as a prop.

# Alternatives

- New special attribute `slot-props` ([as a previous version of this draft](https://github.com/vuejs/rfcs/blob/8575e72d5a401db5d7206e127e3a6012491d68ed/active-rfcs/0000-new-scoped-slots-syntax.md))
- Directive (`v-scope`) instead of a special attribute as originally proposed in [#9180](https://github.com/vuejs/vue/issues/9180);
- Alternative shorthand symbol (`&`) as suggested by @rellect [here](https://github.com/vuejs/vue/issues/9306#issuecomment-453946190)

# Adoption strategy

The change is fully backwards compatible, so we can roll it out in a minor release (planned for 2.6).

`slot-scope` is going to be soft-deprecated: it will be marked deprecated in the docs, and we would encourage everyone to use / switch to the new syntax, but we won't bug you with deprecation messages just yet because we know it's not a top priority for everyone to always migrate to the newest stuff.

In 3.0 we do plan to eventually remove `slot-scope`, and only support the new syntax. We will start emitting deprecation messages for `slot-scope` usage in the next 2.x minor release to ease the migration to 3.0.

Since this is a pretty well defined syntax change, we can potentially provide a migration tool that can automatically convert your templates to the new syntax.
