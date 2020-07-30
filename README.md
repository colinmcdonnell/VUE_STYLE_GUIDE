


# American First Finance Vue.js Style Guide and Frontend Standards
### *** Including GIT and use of Sass/SCSS ***

## Purpose

This guide provides a uniform way to structure your [Vue.js](http://vuejs.org/) code. Making it:

* easier for developers/team members to understand and find things.
* easier for IDEs to interpret the code and provide assistance.
* easier to (re)use build tools you already use.

## Table of Contents

1. [Module based development](#markdown-header-1-module-based-development)
2. [Variable names](#markdown-header-2-variable-names)
3. [Reusable axios calls](#markdown-header-3-reusable-axios-calls)
4. [Vue component names](#markdown-header-4-vue-component-names)
5. [Keep component expressions simple](#markdown-header-5-keep-component-expressions-simple)
6. [Keep component props primitive](#markdown-header-6-keep-component-props-primitive)
7. [Harness your component props](#markdown-header-7-harness-your-component-props)
8. [Assign `this` to `component`](#markdown-header-8-assign-this-to-component)
9. [Component structure](#markdown-header-9-component-structure)
10. [Component imports](#markdown-header-10-component-imports)
11. [Component event names](#markdown-header-11-component-event-names)
12. [Use component name as style scope](#markdown-header-12-use-component-name-as-style-scope)
13. [Lint your component files](#markdown-header-13-lint-your-vue-projects-files-and-use-prettier)
14. [Node packages and dependencies](#markdown-header-14-node-packages-and-dependencies)
15. [Use mixins wherever possible](#markdown-header-15-use-mixins-wherever-possible)
16. [Create components when needed](#markdown-header-16-create-components-when-needed)
17. [GIT](#markdown-header-17-git)
18. [SCSS styles](#markdown-header-18-scss-styles)


## 1. Module based development

Always construct your app out of small modules which do one thing and do it well.

A module is a small self-contained part of an application. The Vue.js library is specifically designed to help you create *view-logic modules*.

### Why?

Small modules are easier to learn, understand, maintain, reuse and debug. Both by you and other developers.

### How?

Each Vue component (like any module) must be [FIRST](https://addyosmani.com/first/):

(F)ocused.

(I)ndependent.

(R)eusable.

(S)mall.

(T)estable.


If your component does too much or gets too big, split it up into smaller components which each do just one thing.

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 2. Variable Names

Each variable name must be:

* **Meaningful**: specific enough, but not overly abstract.
* **Short**: 2 or 3 words.
* **Pronounceable**: we want to be able to talk about them.

Variable names must also be:

* **Casing**: [Hungarian notation](https://en.wikipedia.org/wiki/Hungarian_notation)
* **Declarations**: Usage of ```let``` or ```const``` when applicable. See this [article](https://tylermcginnis.com/var-let-const/) by Tyler McGinnis.

### Why?

* Allows developers to quickly recognize what a particular variable type is and what it is used for.
* Improves code readability and maintainability.
* [let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let) and [const](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const) declarations will maintain our scope.

### How?

```html
<!-- recommended -->
// use const if variable will never be changed during execution - variable will not be reassigned
const sDataString = 'string'
const aDataArray = []
const oDataObj = {}

// use let if variable will only exist in this scope and is able to change - variable may be reassigned
let iDataNumber = 123 (int is used for all number types)
let bDataBool = true

<!-- avoid -->
var isLoading = false
var firstName = 'John'
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 3. Reusable Axios Calls

***Use async/await.***
Async/await allows us to write asynchronous code in a synchronous manner. 

Always set up your axios instance(s) in a way to prevent duplicate efforts when writing calls. We can do this by using mixin files (see [#13](#markdown-header-15-use-mixins-wherever-possible)) or plugins.

If we need to make multiple calls to the same end point, there should only be one axios call written and then reused when necessary.

### Why?

* Reduces duplicate efforts and lines of code needed in the application. 
* Consolidates logic to a central location.
* Promotes readability and maintainability. 
* Async/await provides a cleaner syntax when writing asynchronous Javascript code.

### How?

Using axios plugin:

```js
export default function({ $axios, redirect }, inject) {
  // Create a custom axios instance
  const api = $axios.create({
    headers: {
      common: {
        Accept: 'application/json',
        'Content-Type': 'application/json'
      }
    }
  });

  // Inject to context as $api
  inject('api', api);
}
```

If using the nuxt module, you will need to register your plugin in the nuxt.config.js file:
```js
modules: ['@nuxtjs/axios'],
plugins: ['~/plugins/axios']
```


Set auth token (if needed) before making a call:

```js
// set token before $post
this.$api.setToken(sAuthToken);
const response = await this.$api.$post(sUrl, oOutbound);

if (response.success) {
  // handle response
} 
```

Using a mixin:

```js
import axios from 'axios';

axios.interceptors.response.use(
  function(response) {
    return response;
  },
  function(error) {
    const originalRequest = error.config;
    let oData = JSON.parse(originalRequest.data);
    if (
      error.response.status >= 401 &&
      !originalRequest._retry
    ) {
      originalRequest._retry = true;
      originalRequest.data = JSON.stringify(oData);
      return axios(originalRequest);
    }
    return Promise.reject(error);
  }
);

export const callAxiosMixin = {
  methods: {
    // create method to make axios calls
    async $callAxios(oOutBound, sToken = '') {
      if (!oOutBound.data) {
        oOutBound.data = {};
      }
      axios.defaults.headers.common = {
        Accept: 'application/json',
        'Content-Type': 'application/json',
        Authorization: sToken
      };
      const response = await axios(oOutBound)
        .then(response => {
          return response;
        })
        .catch(error => {
          // handle catch
        });
      return response.data;
    },

    //Retrieve Customer Status - We can now call this function from the mixin anywhere in our app to check the customer status
    async $getCustomerStatus(iCust, iAcct) {
      const oOutbound = {
        url: 'get-cust-status',
        method: 'post',
        data: {
          iCust: iCust,
          iAcct: iAcct
        }
      };
      const response = await this.$callAxios(
        'customer-status',
        oOutbound,
        this.sJWT
      );
      if (!response.success) {
        // handle response
      }
      return response;
    },
  }
};
```

Import and register axios mixin in root (app.js) to be globally accessible in all components:

```js
import { callAxiosMixin } from '@mixins/callAxiosMixin';
Vue.mixin(callAxiosMixin);
```

Making call to mixin from a component:

```js
async getCustomerStatus() {
  await this.$getCustomerStatus().then(response => {
    if (response.success) {
      // handle response
    } 
  });
}
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 4. Vue Component Names

Each component name must be:

* **Meaningful**: not over specific, not overly abstract.
* **Short**: 2 or 3 words.
* **Pronounceable**: we want to be able to talk about them.

Vue component names must also be:

* **Custom element spec compliant**: do NOT use reserved names.
* **Namespaced**: very generic and otherwise 1 word, so that it can easily be reused in other projects.
* **Casing**: component file names should be camelCase while templates will include hyphens (kebab-case) and name attributes will be PascalCase.

### Why?

* The name is used to communicate about the component. So it must be short, meaningful and pronounceable.
* camelCasing file names while using hyphens (kebab-case) for the template and PascalCase for the name attribute provides separation of file name, template and component attributes.

### How?

```html
<!-- recommended file names-->
appHeader.vue
userList.vue
rangeSlider.vue

<!-- recommended component names-->
name: "AppHeader"
name: "UserList"
name: "RangeSlider"

<!-- recommended template usage-->
<app-header></app-header>
<user-list></user-list>
<range-slider></range-slider>

<!-- avoid -->
btn-group.vue <!-- hyphen not needed and unpronounceable. use `buttonGroup` instead -->

name: "btnGroup" <!-- unpronounceable. use `button-group` instead -->

<btn-group></btn-group> <!-- short, but unpronounceable. use `button-group` instead -->
<ui-slider></ui-slider> <!-- all components are ui elements, so is meaningless -->
<slider></slider> <!-- not custom element spec compliant -->
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 5. Keep component expressions simple

Vue.js's inline expressions are 100% Javascript. This makes them extremely powerful, but potentially also very complex. Therefore you should **keep expressions simple**.

### Why?

* Complex inline expressions are hard to read.
* Inline expressions can't be reused elsewhere. This can lead to code duplication and code rot.
* IDEs typically don't have support for expression syntax, so your IDE can't autocomplete or validate.

### How?

If it gets too complex or hard to read **move it to methods or computed properties**!

```html
<!-- recommended -->
<template>
  <h1>
    {{ `${year}-${month}` }}
  </h1>
</template>

<script>
  export default {
    computed: {
      month() {
        return this.twoDigits((new Date()).getUTCMonth() + 1);
      },
      year() {
        return (new Date()).getUTCFullYear();
      }
    },
    methods: {
      twoDigits(num) {
        return ('0' + num).slice(-2);
      }
    },
  };
</script>

<!-- avoid -->
<template>
  <h1>
    {{ `${(new Date()).getUTCFullYear()}-${('0' + ((new Date()).getUTCMonth()+1)).slice(-2)}` }}
  </h1>
</template>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 6. Keep component props primitive

While Vue.js supports passing complex JavaScript objects via these attributes, you should try to **keep the component props as primitive as possible**. Try to only use [JavaScript primitives](https://developer.mozilla.org/en-US/docs/Glossary/Primitive) (strings, numbers, booleans) and functions. Avoid complex objects.

### Why?

* By using an attribute for each prop separately the component has a clear and expressive API;
* By using only primitives and functions as props values our component APIs are similar to the APIs of native HTML(5) elements;
* By using an attribute for each prop, other developers can easily understand what is passed to the component instance;
* When passing complex objects it's not apparent which properties and methods of the objects are actually being used by the custom components. This makes it hard to refactor code and can lead to code rot.

### How?

Use a component attribute per props, with a primitive or function as value:

```html
<!-- recommended -->
<range-slider
  :values="[10, 20]"
  :min="0"
  :max="100"
  :step="5"
  @on-slide="updateInputs"
  @on-end="updateResults">
</range-slider>

<!-- avoid -->
<range-slider :config="complexConfigObject"></range-slider>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 7. Harness your component props

In Vue.js your component props are your API. A robust and predictable API makes your components easy to use by other developers.

Component props are passed via custom HTML attributes. The values of these attributes can be Vue.js plain strings (`:attr="value"` or `v-bind:attr="value"`) or missing entirely. You should **harness your component props** to allow for these different cases.

### Why?

Harnessing your component props ensures your component will always function (defensive programming). Even when other developers later use your components in ways you haven't thought of yet.

### How?

* Use defaults for props values.
* Use `type` option to [validate](http://vuejs.org/v2/guide/components.html#Prop-Validation) values to an expected type.**[1\*]**
* Check if props exists before using it.

```html
<template>
  <input type="range" v-model="value" :max="max" :min="min">
</template>

<script>
  export default {
    props: {
      max: {
        type: Number, // [1*] This will validate the 'max' prop to be a Number.
        default() { return 10; },
      },
      min: {
        type: Number,
        default() { return 0; },
      },
      value: {
        type: Number,
        default() { return 4; },
      },
    },
  };
</script>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 8. Assign `this` to `component`

Within the context of a Vue.js component element, `this` is bound to the component instance.
Therefore when you need to reference it in a different context, ensure `this` is available as `component`.

In other words: Do **NOT** code things like `var self = this;` anymore if you're using **ES6**. You're safe using Vue components.

### Why?

* Using ES6, there's no need to save `this` to a variable;
* In general, when using arrow functions the lexical scope is kept
* If you're **NOT** using ES6 and, therefore, not using `Arrow Functions`, you'd have to add `this` to a variable. That's the only exception.

### How?

```html
<script>
export default {
  methods: {
    hello() {
      return 'hello';
    },
    printHello() {
      console.log(this.hello());
    },
  },
};
</script>

<!-- avoid -->
<script>
export default {
  methods: {
    hello() {
      return 'hello';
    },
    printHello() {
      const self = this; // unnecessary
      console.log(self.hello());
    },
  },
};
</script>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 9. Component structure

Make it easy to reason and follow a sequence of thoughts. See the How.

### Why?

* Having the component export a clear and grouped object, makes the code easy to read and easier for developers to have a code standard.
* Grouping makes the component easier to read (name; extends; props, data and computed; components; watch and methods; lifecycle methods, etc.);
* Use the `name` attribute. Using [vue devtools](https://chrome.google.com/webstore/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd?hl=en) and that attribute will make your development/testing easier;

### How?

Component structure:

```html
<template>
  <div class="RangeSlider__Wrapper">
    <!-- ... -->
  </div>
</template>

<script>
  export default {
    // Do not forget this little guy
    name: 'RangeSlider',
    // LAZY LOAD components (see Component Imports section)
    components: {
        componentExample: () => import('./componentExample')
    },
    directives: {},
    // share common functionality with component mixins
    mixins: [],
    // compose new components
    extends: {},
    // component properties/variables
    props: {
      bar: {}, // Alphabetized
      foo: {},
      fooBar: {},
    },
    // variables
    data() {},
    computed: {},
    // methods
    watch: {},
    methods: {},
    // component Lifecycle hooks
    beforeCreate() {},
    mounted() {},
};
</script>

<style scoped>
  .RangeSlider__Wrapper { /* ... */ }
</style>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 10. Component Imports

Each component should be imported with lazy loading.
* Lazy loading is a process of loading parts (chunks) of your application lazily - call it when you need it

### Why?

* So we can cut off bundle size when we still need to add new features and improve our application.
* Loading components only when we really need them.
* It is wasteful at best, to download, parse and execute the entire bundle everything on every page load when only a few parts are needed

### How?

Component import (with lazy load):

```html
components: {
    componentExample: () => import('./componentExample')
}
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 11. Component event names

Vue.js provides all Vue handler functions and expressions that are strictly bound to the ViewModel. Each components events should follow a good naming style to avoid issues during development. See the **Why** below.

### Why?

* Developers are free to use native like event names that can cause confusion down the line;
* The freedom of naming events can lead to a [DOM template incompatibility](https://vuejs.org/v2/guide/components.html#DOM-Template-Parsing-Caveats);

### How?

Event names should be camelCased
A unique event name should be fired for unique actions in your component that will be of interest to the outside world, like: uploadSuccess, uploadError or even dropzoneUploadSuccess, dropzoneUploadError (if you see the need for having a scoped prefix);

Example of committing to the vuex store:
```js
<!-- recommended -->
this.$store.commit('oPayment/saveCard', this.bSaveCard); // the saveCard event is a mutator in oPayment state

<!-- avoid -->
this.$store.commit('oPayment/save-card', this.bSaveCard)
```

Example of emitting an eventBus event:
```js
<!-- recommended -->
eventBus.$emit('brandingContent', brandingContent);

<!-- avoid -->
eventBus.$emit('branding-content', brandingContent);
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 12. Use component name as style scope

Vue.js component elements are custom elements which can very well be used as style scope root.
Alternatively the component name can be used as CSS class namespace.

### Why?

* Scoping styles to a component element improves predictability as it prevents styles leaking outside the component element.
* Using the same name for the module directory, the Vue.js component and the style root makes it easy for developers to understand they belong together.

### How?

Use the component name as a namespace prefix based on BEM and OOCSS **and** use the `scoped` attribute on your style class.
The use of `scoped` will tell your Vue compiler to add a signature on every class that your `<style>` have. That signature will force your browser (if it supports) to apply your components
CSS on all tags that compose your component, leading to a no leaking css styling.

```html
<template>
    <div class="MyExample">
      <li></li>
    </div>
</template>

<style scoped>
  /* recommended */
  .MyExample { }
  .MyExample li { }
  .MyExample__item { }

  /* avoid */
  .My-Example { } /* not scoped to component or module name, not BEM compliant */
</style>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 13. Lint your vue project's files (and use prettier)

Linters improve code consistency and help trace syntax errors.
Using pre-hooks we can make sure all files are linted and prettified before making it to the repository.

### Why?

* Ensures all developers use the same code style.
* Helps you trace syntax errors before it's too late.
* Prevents "ugly" code from muddying up the project.

### How?

#### ESLint, Prettier and Husky

Configure ESLint in a `.eslintrc.js` file (so IDEs can interpret it as well):

This example also shows the nuxt extension.

```js
module.exports = {
  root: true,
  env: {
    browser: true,
    node: true
  },
  parserOptions: {
    parser: 'babel-eslint'
  },
  extends: [
    '@nuxtjs',
    'prettier',
    'prettier/standard',
    'prettier/unicorn',
    'prettier/vue',
    'plugin:prettier/recommended',
    'plugin:nuxt/recommended'
  ]
};
```

Run ESLint manually

This command will lint all files with .js and .vue extensions.

```bash
eslint --ext .js,.vue .
```

Pre-commit hook for prettier via husky package:

```bash
// install husky package
npm install husky lint-staged prettier --save
```

Then add settings to your package.json:

```json
"lint-staged": {
    "*.{js,vue}": [
        "eslint",
        "prettier --write"
    ]
},
"husky": {
    "hooks": {
        "pre-commit": "lint-staged"
    }
}
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 14. Node Packages and Dependencies

Check to see if you can elegantly solve an issue before adding a package to your project.

***"Can I figure out how to format dates instead of adding the entire moment.js library to my app?"***

All packages (non-initialization) must be approved by peer front end developers before pulling anything into a project.

Always update the team regarding what packages will need to be installed when pulling new updates from Develop.

***Don't reinvent the wheel, but also don't be lazy***

### Why?

* So we can avoid using large packages that will hinder our application's performance.
* Reduce the amount of libraries required for an application to function.

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 15. Use mixins wherever possible

### Why?

Mixins encapsulate reusable code and avoid duplication. If two components share the same functionality, a mixin can be used. With mixins, you can focus on the individual component task and abstract common code. This helps to better maintain your application.

### How?

Let's say you have a mobile and a desktop menu components who share some functionality. We can abstract the core 
functionalities of both into a mixin like this.

```js
const MenuMixin = {
  data () {
    return {
      language: 'EN'
    }
  },

  methods: {
    changeLanguage () {
      if (this.language === 'DE') this.$set(this, 'language', 'EN')
      if (this.language === 'EN') this.$set(this, 'language', 'DE')
    }
  }
}

export default MenuMixin
```

To use the mixin, simply import it into both components (I only show the mobile component).

```html
<template>
  <ul class="mobile">
    <li @click="changeLanguage">Change language</li>
  </ul>  
</template>

<script>
  import MenuMixin from './MenuMixin'

  export default {
    mixins: [MenuMixin]
  }
</script>
```

---

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 16. Create components when needed

### Why?

Vue.js is a component framework based. Not knowing when to create components can lead to issues like:

* If the component is too big, it probably will be hard to (re)use and maintain;
* If the component is too small, your project gets flooded, harder to make components communicate;

### How?

* Always remember to build your components for your project needs, but you should also try to think of them being able to work out of it. If they can work out of your project, such as a library, it makes them a lot more robust and consistent;
* It's always better to build your components as early as possible since it allows you to build your communications (props & events) on existing and stable components.

### Rules

* First, try to build obvious components as early as possible such as: modal, popover, toolbar, menu, header, etc. Overall, components you know for sure you will need later on. On your current page or globally;
* Secondly, on each new development, for a whole page or a portion of it, try to think before rushing in. If you know some parts of it should be a component, build it;
* Lastly, if you're not sure, then don't! Avoid polluting your project with "possibly useful later" components, they might just stand there forever, empty of smartness. Note it's better to break it down as soon as you realize it should have been, to avoid the complexity of compatibility with the rest of the project;

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 17. GIT
***Never commit without linting and running prettier*** (see [#13](#markdown-header-13-lint-your-vue-projects-files-and-use-prettier))

It is important that we do not make contributing code harder than it should be... 
ALWAYS pull develop into your working branch once releases are made to production and the production environment branch is merged into develop.

***Weekly syncs after deployments are required and an every day develop pull doesn't hurt***

If there have been packages added to your project, always make sure to run ```npm install --save``` before attempting to compile or run the project locally.

Make sure you are ignoring all necessary files from git by including the files in your .gitignore

### Why?

* Reduce conflicts when merging.
* Keeping all code current with the production branch.
* Prevents developers from committing unreadable, unwanted or "ugly" code.

### .gitignore example

Your git ignore file should limit your commits from including at minimum these files for a nuxt/laravel project:

*addresses nuxt project ignore rules as well

```.dotenv
/public/hot
/public/storage
/storage/*.key
/storage
/public/js/app.js
/public/_nuxt
/public/app
/public
/public/mix-manifest.json
/public/spa.html
/vendor
.env
.env.backup
.phpunit.result.cache
Homestead.json
Homestead.yaml
```

Additional suggested files to ignore (if applies to nuxt project)

*Created by .ignore support plugin (hsz.mobi)

```.dotenv
### Node template
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Directory for instrumented libs generated by jscoverage/JSCover
lib-cov

# Coverage directory used by tools like istanbul
coverage

# nyc test coverage
.nyc_output

# Grunt intermediate storage (http://gruntjs.com/creating-plugins#storing-task-files)
.grunt

# Bower dependency directory (https://bower.io/)
bower_components

# node-waf configuration
.lock-wscript

# Compiled binary addons (https://nodejs.org/api/addons.html)
build/Release

# Dependency directories
node_modules/
jspm_packages/

# TypeScript v1 declaration files
typings/

# Optional npm cache directory
.npm

# Optional eslint cache
.eslintcache

# Optional REPL history
.node_repl_history

# Output of 'npm pack'
*.tgz

# Yarn Integrity file
.yarn-integrity

# dotenv environment variables file
.env

# parcel-bundler cache (https://parceljs.org/)
.cache

# next.js build output
.next

# nuxt.js build output
.nuxt

# Nuxt generate
dist

# vuepress build output
.vuepress/dist

# Serverless directories
.serverless

# IDE / Editor
.idea
.editorconfig

# Service worker
sw.*

# Mac OSX
.DS_Store

# Vim swap files
*.swp

```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


## 18. SCSS styles
***Please refer to the SCSS style guide [here](https://docs.gitlab.com/ee/development/fe_guide/style/scss.html) to understand AFF's expectations.***

***Always avoid the following:***

* **Styles should not be copied directly from the design comps**: These styles do not properly reflect what is necessary to achieve the desired UI. Font sizes and hex values are usually the only usable styles provided from design comps.
* **Use of `!important`**: This should only be used when overriding Bootstrap, Vuetify or any other UI framework. Even then... use with caution and avoid at all cost.
* **Use of `float` property**: We use flexbox where floats are not necessary and can cause issues with flex display.
* **Custom Media Queries**: We should always rely on the breakpoints that our framework provides us. Here are [Vuetify's breakpoints](https://vuetifyjs.com/en/customization/breakpoints/) and [Bootstrap's breakpoints](https://getbootstrap.com/docs/4.0/layout/overview/). It's important to know the UI framework you are working with and only target their breakpoints.

***Reasons we use SCSS:***

* **Expressive**: We can compress several lines of code and encourage better nesting of styles.
* **Nested Syntax**: Nesting allows a cleaner way of targeting elements and provides better code readability (which improves maintainability of code).
* **Variables**: We can define variables so that updating a theme color only has to happen in a single location.
* **Well Documented**: Information regarding syntax, compiling, advanced mathematical functions or anything else you could want to know are easily found online.


***Examples:***

**Using Variables**

If we need to change the shade of red used for errors, we only have to replace the hex values for these variables.

This prevents us from searching for and changing each instance. In the below example, both page.vue and component.vue will be updated by simply changing the hex values for $bg-error and $aff-error-text in the variables file.

**assets/variables.scss file**
```scss
$bg-error: #feecef;
$aff-error-text: #f8485e;
```

**in ```<styles>``` (always define lang type)**
```scss
<!-- page.vue -->
<style lang="scss">
.error-box {
  background: $bg-error;
  .error-message {
    color: $aff-error-text;
  }
}
</style>

<!-- component.vue -->
<style lang="scss">
.error-box {
  background: $bg-error;
  .error-message {
    color: $aff-error-text;
  }
}
</style>
```

**Nested Syntax**

It is important that we do not mix CSS with SCSS as well as properly formatting our code in order to improve maintainability. This will cause issues if a developer has to find a style that is not properly nested. 

You could be targeting a class that is already being targetted in the nested syntax by writing CSS outside of scope. This could have a negative effect when compiling SCSS.

```scss
<!-- recommended -->
.nav-container {
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
  position: fixed;
  width: 100%;
  z-index: 99;
  background: #fff;
  .navigation {
    justify-content: space-between;
    align-items: center;
    .nav-logo {
      min-width: 200px;
      padding: 0.5rem;
    }
  }
}


<!-- avoid writing CSS outside of SCSS nesting -->
.nav-container {
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
  position: fixed;
  width: 100%;
  z-index: 99;
  background: #fff;
  .navigation {
    justify-content: space-between;
    align-items: center;
  }
}
.nav-container .navigation .nav-logo {
  min-width: 200px;
  padding: 0.5rem;
}


<!-- avoid writing CSS and 4 spaced indents -->
.nav-container {
    box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
    position: fixed;
    width: 100%;
    z-index: 99;
    background: #fff;
}

.navigation {
    justify-content: space-between;
    align-items: center;
}

.nav-logo {
    min-width: 200px;
    padding: 0.5rem;
}
```



**Media Queries**

Custom breakpoints will cause the developer to fight against their framework. Always avoid using breakpoints that are not already defined by the UI framework as well as targeting `min-width` AND `max-width`. We can handle an entire project by only using `min-width` and it is highly advised we do not mix selectors if possible.

```scss
<!-- recommended -->
<style lang="scss">
@media (min-width: 960px) {
  .nav-container {
    .navigation {
      .nav-links {
        justify-content: space-between;
        box-shadow: none;
        overflow: unset;
      }
    }
  }
}
</style>

<!-- avoid -->
<style lang="scss">
@media (max-width: 1000px) and (min-width: 380px) {
  .nav-container {
    .navigation {
      .nav-links {
        justify-content: space-between;
        box-shadow: none;
        overflow: unset;
      }
    }
  }
}
</style>
```

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)

## PLEASE CONTRIBUTE!

Create a feature branch named (yourname_revisions) and submit a Pull Request containing revisions or additions to this document.
Add all vue developers for review.

[return to table of contents](#markdown-header-american-first-finance-vuejs-style-guide-and-frontend-standards)


