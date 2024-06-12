# Crayon (TODO)

A tool for adding lightweight data-bound templating syntax to web projects at runtime or compile time.

## Basic JavaScript (No Build System)

```html
<html>
  <head>
    <title>My App</title>
  </head>
  <body>
    <div id="root">
      <hello-world />
    </div>

    <script type="module">
      import { html, mount, Component } from 'https://cdn.alshdavid.com/crayon/index.js'

      Component.register(HelloWorldComponent, {
        tag: 'hello-world',
        template: html`
          <div>{{ helloWorld }}</div>
          <button (onclick)="onClick">Click Me!</button>
        `
      })
      class HelloWorldComponent {
        helloWorld

        constructor() {
          this.helloWorld = 'initial value'
        }

        onClick() {
          this.helloWorld = 'updated value'
        }
      }

      const instance = mount({
        root: document.querySelector('#root')
        target: HelloWorldComponent,
      })
    </script>
  </body>
</html>
```

# Using External Components

```typescript
import { Component, html, mount } from 'crayon'
import { FooComponent } from './foo.js'

Component.register(HelloWorldComponent, {
  tag: 'hello-world',
  template: html`
    <div>Hello World</div>
    <foo /> <!-- This is FooComponent -->
  `,
  components: [FooComponent]
})
export class HelloWorldComponent {}
```

# Using External Components (Lazy)

```typescript
import { Component, html, mount } from 'crayon'

Component.register(HelloWorldComponent, {
  tag: 'hello-world',
  template: html`
    <div>Hello World</div>
    <foo /> <!-- This is FooComponent -->
  `,
  components: [
    () => (await import('./foo.js')).FooComponent,
  ]
})
export class HelloWorldComponent {}
```

## Routing

Using the generalized `crayon-router`

```javascript
import { Router } from 'crayon-router'
import crayonRouter from 'crayon-router/crayon'
import { HomePageComponent } from './pages/home.js'
import { UserPageComponent } from './pages/user.js'
import { NotFoundPageComponent } from './pages/not-found.js'

const router = new Router()

app.use(crayonRouter({
  root: document.querySelector('#root')
}))

app.path('/', async ctx => {
  await ctx.mount(HomePageComponent)
})

app.path('/users/:id', async ctx => {
  await ctx.mount(UserPageComponent)
})

app.path('/**', async ctx => {
  await ctx.mount(NotFoundPageComponent)
})

await app.load()
```

## Services / Dependencies

```typescript
import { Component, html, bootstrap, Provider } from 'crayon'

class UserService {
  async getUserName(): string {
    return fetch(`/api/user`).then(response => response.text())
  }
}

Component.register(HelloWorldComponent, {
  tag: 'hello-world',
  template: html`
    <div>Hello {{ userName }}</div>
  `,
  inject: ['UserService']
})
export class HelloWorldComponent {
  userName: string
  #userService: UserService

  constructor(
    userService: UserService
  ) {
    this.userService = userService
    this.userName = ''
  }

  async onInit() {
    this.userName = await this.#userService.getUserName()
  }
}

const services = new Map()
services.add('UserService', () => new UserService())

const instance = mount({
  root: document.querySelector('#root')
  target: HelloWorldComponent,
  provide: services,
})
```

## Props

```typescript
import { Component, html, InputBinding, OutputBinding, OutputEmitter } from 'crayon'

Component.register(HelloWorldComponent, {
  tag: 'hello-world',
  template: html`
    <div>{{ foo }}</div>
    <button (onclick)="bar.emit('clicked')">Click me</button>
  `,
  inputBindings: ['foo'],
  outputBindings: ['bar'],
  provide: [
    InputBinding,
    OutputBinding,
  ]
})
export class HelloWorldComponent {
  foo: string
  bar: OutputEmitter<string>

  constructor(
    inputBinding: InputBinding,
    outputBinding: OutputBinding,
  ) {
    this.foo = inputBinding.init(this, 'foo')
    this.bar = outputBinding.init(this, 'bar')
  }
}
```

## Reactivity

Reactivity only occurs on top level component properties and is not deeply evaluated

## Internal details, what do templates compile to?

Crayon compiles its template syntax into JSX using the `html` tagged template and uses Preact under the hood as a runtime

### Input

```javascript
Component.register(HelloWorldComponent, {
  tag: 'hello-world',
  template: html`
    <div>{{ helloWorld }}</div>
    <button (onclick)="onClick">Click Me!</button>
  `,
  provide: ['InjectThis']
})
class HelloWorldComponent {
  helloWorld

  constructor(
    injectThis: string
  ) {
    this.helloWorld = injectThis // 'initial value'
  }

  onClick() {
    this.helloWorld = 'updated value'
  }
}
```

### Output

```jsx
import { h, Component } from 'preact'
import { ProviderContext } from 'crayon'

class HelloWorldComponent extends Component {
  static contextType = ProviderContext
  context

  constructor(props) {
    super()
    this.setState({
      helloWorld: this.context.get('InjectThis')
    })
  }

  onClick() {
    this.setState({
      helloWorld: 'updated value'
    })
  }

  render() {
    return <Fragment>
      <div>{this.state.helloWorld}</div>
      <button onClick={() => this.onClick()}>Click Me!</button>
    </Fragment>
  }
}
```