# Form handling

Forms in Vue can be as simple as plain HTML forms to complicated nested trees of custom Vue component form elements. 
We will gradually go through the ways of interacting with form elements, setting values and triggering events. 

The methods we will be using the most are `setValue()` and `trigger()`. 

## Interacting with form elements

Let's take a look at a very basic form:

```vue
<template>
  <div>
    <input type="email" v-model="email">

    <button @click="submit">Submit</button>
  </div>
</template>

<script>
export default {
  data() {
    return { 
      email: ''
    }
  },
  methods: {
    submit() {
      this.$emit('submit', this.email)  
    }
  }
}
</script>
```

### Setting element values

The most common way to bind an input to data in Vue is by using `v-model`. As you probably know by now, it takes care of what events each form element emits,
and the props it accepts, making it easy for us to work with form elements.

To change the value of an input in VTU, you can use the `setValue()` method. It accepts a parameter, most often a `String` or a `Boolean`, and returns a `Promise`, which resolves after Vue has updated the DOM.

```js
test('sets the value', async () => {
  const wrapper = mount(Component)
  const input = wrapper.find('input')

  await input.setValue('my@mail.com')

  expect(input.element.value).toBe('my@mail.com')
})
```

As you can see, `setValue` sets the `value` property on the input element to what we pass to it.

We are using `await` to make sure that Vue has completed updating and the change has been reflected in the DOM, before we make any assertions.

### Triggering events

Triggering events is the second most important action when working with forms and action elements. Let's take a look at our `button`, from the previous example.

```html
<button @click="submit">Submit</button>
``` 

To trigger a click event, we can use the `trigger` event.

```js
test('trigger', async () => {
  const wrapper = mount(Component)  
  // trigger the element
  await wrapper.find('button').trigger('click')
  // assert some action has been performed, like an emitted event.
  expect(wrapper.emitted('submit')).toBeTruthy()
})
```

We trigger the `click` event listener, triggering the `submit` method. Similar to `setValue` we are using `await` to make sure the action is being reflected by Vue.
 
We can then assert that some action has been performed, like if the emitted event has been called.

Let's combine these two to test whether our simple form is emitting the email the user inputs.

```js
test('emits the input to its parent', async () => {
  const wrapper = mount(Component)  
  // set the value
  await wrapper.find('input').setValue('my@mail.com')
  // trigger the element
  await wrapper.find('button').trigger('click')
  // assert the `submit` event is emitted,
  expect(wrapper.emitted('submit')[0][0]).toBe('my@mail.com')
})
```

## Advanced workflows

Now that we know the basics, let's dive into more complex examples.

### Working with various form elements
 
We saw `setValue` works with input elements, but is much more versatile, as it can set the value on various types of input elements.

Let's take a look at a more complicated form, which has more types of inputs.

```vue
<template>
  <form @submit.prevent="submit">
    <input type="email" v-model="form.email">

    <textarea v-model="form.description"/>

    <select v-model="form.city">
      <option value="new-york">New York</option>
      <option value="moscow">Moscow</option>
    </select>

    <input type="checkbox" v-model="form.subscribe"/>

    <input type="radio" value="weekly" v-model="form.interval"/>
    <input type="radio" value="monthly" v-model="form.interval"/>
    
    <button type="submit">Submit</button>
  </form>
</template>

<script>
export default {
  data() {
    return { 
      form: { 
        email: '',
        description: '',
        city: '',
        subscribe: false,
        interval: ''
      }
    }
  },
  methods: {
    async submit() {
      this.$emit('submitted')
    }
  }
}
</script>
```

Our extended Vue component is a bit longer, has a few more input types and now has the `submit` handler moved to a `<form/>` element.

The same way we set the value on the `input`, we can set it on all the other inputs in the form.

```js
import { mount } from '@vue/test-utils'
import FormComponent from './FormComponent.vue'

test('submits a form', async () => {
  const wrapper = mount(FormComponent)
  await wrapper.find('input[type=email]').setValue('name@mail.com')
  await wrapper.find('textarea').setValue('Lorem ipsum dolor sit amet')
  await wrapper.find('select').setValue('moscow')
  await wrapper.find('input[type=checkbox]').setValue()
  await wrapper.find('input[type=radio][value=monthly]').setValue()
})
```

As you can see, `setValue` is a very versatile method, it can work with all types of form elements.

We are using `await` everywhere, to make sure that each change has been applied before we trigger the next. This is a good practice to follow,
that can save you time later on, if you do assertions but changes are not applied to the DOM yet. 


::: tip
If you dont pass a parameter to `setValue` for `OPTION`, `CHECKBOX` or `RADIO` inputs, they will set as `checked`.
:::

We have set values in our form, now it's time to submit the form and do some assertions.

### Triggering complex event listeners

Event listeners are not always simple `click` events. Vue allows you to listen to all kinds of DOM events, add special modifiers like `.prevent` and more. Let's take a look how we can test those.

In our form above, we moved the event from the `button` to the `form` element. This is a good practice to follow, as this allows you to submit a form by hitting the `enter` key, which is a more native approach.

To trigger the `submit` handler, we will use the `trigger` method again.

```js
test('submits the form', async () => {
  // ... previous test code
  await wrapper.find('form').trigger('submit.prevent')
  expect(wrapper.emitted('submit')[0][0]).toBe("")
})
```

You probably noticed we have a `.prevent` in there, why? This is so that the page does not refresh when the form is submitted.
 
To test it, we directly copy-pasted our event string `submit.prevent` into `trigger`, that is because VTU can read the passed event and all the applied modifiers on it, and selectively apply what is necessary. Native event modifiers like `.prevent`, `.stop` etc, are Vue-specific and as such we dont need to test them, Vue internals do that already.
 
We then make a simple assertion, whether the form emitted the correct event and payload.

#### Multiple modifiers on the same event

Let's assume you have a very detailed and complex form, with special interaction handling. How can we go about testing that?

```html
<input @keydown.meta.c.exact.prevent="captureCopy" v-model="input" />
```

Assume we have an input that handles when the user clicks `cmd` + `c`, and we want to intercept and stop him from copying. Testing this is as easy as copy pasting the event.

```js
test('handles complex events', async () => {
  const wrapper = mount(Component)
  await wrapper.find(input).trigger('keydown.meta.c.exact.prevent')
  // assert something
})
```

VTU will read the even and apply the appropriate properties to the event object. In this case it will match something like this:

```js
{
  // ... other properties
  "key": "c",
  "metaKey": true
}
```

#### Adding extra data to an event

Let's say your code needs something from inside the `event` object. You can test such scenarios by passing extra data as a second parameter.
```vue
<template>
  <form>
    <input type="text" v-model="value" @blur="handleBlur">
    <button>Submit</button>
  </form>
</template>
<script>
export default {
  data() {
    return {
      value: ''
    }
  },
  methods: {
    submit() {
      // some logic  
    },
    handleBlur(event) {
      if(event.relatedTarget.tagName === 'BUTTON'){ 
        this.$emit('focus-lost')
      }
    }
  }
}
</script>
```
```js
import Form from './Form.vue'
test('emits an event only if you lose focus to a button', () => {
  const wrapper = mount(Form)
  const componentToGetFocus = wrapper.find('button')
  wrapper.find('input').trigger('blur', {
    relatedTarget: componentToGetFocus
  })
  expect(wrapper.emitted('focus-lost')).toBeTruthy()
})
```

Here we assume our code checks inside the `event` object, whether the `relatedTarget` is a button or not. We can simply pass a reference to such an element,mimicking what would happen if the user clicks on a `button` after typing something in the `input`.

## Interacting with Vue Component inputs

Inputs are not only plain elements. We often use Vue components that function like inputs. They can add markup, styling and alot of functionality, in an easy to use format.

Testing forms that use such inputs can be daunting at first, but with a few simple rules, it quickly becomes a walk in the park.

Take this simple input for example.

```vue
<template>
  <label>
    {{ label }}
    <input type="text" :value="modelValue" @input="$emit('update:modelValue', $event.target.value)">
  </label>
</template>
<script>
export default {
  props: ['modelValue', 'label']
}
</script>
```

This Vue component adds a label and emits back whatever you type. To use it you just do:

```html
<custom-input v-model="input" label="Text Input" class="text-input"/>
```

### Finding the real input element 

Most of these Vue components have a real `button` or `input` in them. You can just as easily find that element and act on it:

```js
test('fills in the form', async () => {
  // ... some extra test code 
  await wrapper.find('.text-input input').setValue('text')
  // continue with assertions or actions like submit the form, assert DOM.
})
```

### Using setValue on the Vue Component

But what if your input is not that simple? What if you are using a UI library, like Vuetify? You cannot just dig inside it, hoping to find the correct input/button and hope those don't change in the next version.

In such cases you can use set the value directly, using the component instance and, you guessed it, `setValue`.

Assume we have a form that uses the Vuetify textarea.

```html
<form>
  <v-textarea v-model="model" ref="description" />
</form>
```

We can update set it's value by simply finding it using `findComponent` and setting the value.

```js
test('sets data in the form', async () => {
  // init wrapper, do other tasks
  await wrapper.findComponent({ ref: 'description' }).setValue('Some very long text...')
  wrapper.find('.submit').trigger('click')
  expect(wrapper.emitted('submit')[0][0]).toEqual({
    model: 'Some very long text...'
  })
})
```

## Conclusion

What did we learn?

* You can use `setValue` to set the value on both DOM inputs and Vue component inputs.
* You can use `trigger` to trigger DOM events, both with and without modifiers.
* You can add extra event data to `trigger`, using the second parameter.
* Dont assert data on the `vm` if possible. Use the DOM, or emitted events instead.
* Assert the outcome (async submitted data, emitted event, conditional elements in DOM etc).