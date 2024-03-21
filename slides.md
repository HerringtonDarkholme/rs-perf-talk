---
theme: seriph
background: /background.avif
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## Rust Performance in JavaScript
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
drawings:
  persist: false
transition: fade
title: Benchmark Rusty Parsers
mdc: true
colorSchema: 'light'

---

# Benchmark Rusty Parsers

Demystify Native Tooling Performance in JavaScript

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" alt="GitHub" title="Open in GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: fade
layout: image
image: /splash-white.png
---

---
transition: fade
layout: image
image: /splash.png
---

<style>
.slidev-page-3.fade-enter-active {
    transition-duration: 3s !important;
}
</style>

---
transition: fade-out
layout: two-cols
---

# About Me

A brief intro, won't be long

<br/>

* 🌐 Frontend Vimmer
* 💻 A web dev
* ⚒️ And a web-dev-tool dev
* 💻 Enjoys TypeScript and Rust
* 🚀 The author of [ast-grep](https://ast-grep.github.io/)
* 🐙 [GitHub](https://github.com/HerringtonDarkholme)
* 📰 [Medium](https://medium.com/@hchan_nvim)

::right::

<img style="padding: 30px 120px;" src="https://avatars.githubusercontent.com/u/2883231?v=4"/>
<img src="https://ast-grep.github.io/logo.svg"/>

---
transition: fade-out
layout: image-right
image: /ast.jpg
---

# Rust in JavaScript

- **Rust** is becoming the native choice in JS world
- Why (Not)?
  - Pros: Predictably Fast. Great Ecosystem.
  - Cons: Learning cost. Fewer contributors. Hard to design **plugins**.
- We need JS plugin!
  - More customizable and easier to use
  - It should understand and change code
- One critical task is parsing JS/TS code
- Our topic today is to benchmark parsers!

---
transition: slide-up
layout: two-cols
---

# But, JS <-> RS has cost!

Calling Rust functions from JavaScript is expensive!

<v-clicks>

* Using Rust function in JS involves some extra conversions and steps.

* **Foreign function call is like travel abroad.**
  * function is destination
  * arguments are luggage
  * return value are souvenir
  * calling convention is visa/passport

* The more luggage/souvenir you have, the more money and time you need to pack and check them.

</v-clicks>


::right::

![img](/crab.jpeg)

---

# Benchmark Design: Choosing Parsers

- **Tree-sitter Based**
  - **[ast-grep](https://ast-grep.github.io/)**: A tree-sitter tool[^1] for structural search, lint, and rewriting based on AST
  - **[Tree-sitter](https://tree-sitter.github.io)**: An incremental parsing library that can build and update concrete syntax trees.

- **Native Rust**
  - **[swc](https://swc.rs/)**: A super-fast TS/JS compiler written in Rust, performant and usable in both RS and JS.
  - **[oxc](https://oxc-project.github.io/)**: A suite of high-performance tools for JS/TS, maybe the fastest parser.

- **JavaScript Based**
  - **[Babel](https://babeljs.io/)**: The Babel parser (previously Babylon) is a JavaScript parser used in Babel compiler.
  - **[TypeScript](https://www.typescriptlang.org/)**: The official parser implementation from the TypeScript team.

<br/>

[^1]: Disclaimer: Presenter is ast-grep's [author](https://github.com/HerringtonDarkholme)

---
layout: two-cols
---

# File Size Categories

To assess parser across a variety of codebases

- **Single Line:**
  - A one-line snippet as baseline
- **Small File:**
  - A 24-line common utility file
- **Medium File:**
  - A typical 400-line file of average file
- **Large File:**
  - The glorious `checker.ts`

<br/>
<p class="opacity-50">Factors not considered: JIT/GC/NodeJS flags</p>

::right::

# Concurrency Level

With more cores comes more performance

* Simulate workload by _parsing five files concurrently_
* In **sync** and **async** fashion

```ts
import {ts as sg} from '@ast-grep/napi'

// sync version
const parseSync = () => sg.parse(source)
// a for loop to parse 5 files, sequentailly
for (let i = 0; i < CONCURRENCY; i++) { parseSync() }

// async version
const parse = () => sg.parseAsync(source)
// parse 5 files asynchronously and concurrently
const tasks = Array(CONCURRENCY).fill(...).map(parse)
await Promise.all(tasks)
```

---
layout: center
---

# Sync Parse

* performance is calcuated as _operations per second_
* _Fastest_ parser has _100%_ score, others have percentages of the fastest parser's score

<div grid="~ cols-5 gap-4">
<div col="span-3">

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qfwhfivek2p0rq3m3paf.png)

</div>
<div col="span-2">
<br/>
<br/>

* TypeScript consistently outperforms the competitors
* Native parsers show improved performance for larger files
* Babel demonstrates unstable performance

</div>
</div>

---
layout: center
---

# Async Parse

* performance is calcuated as _operations per second_
* _Fastest_ parser has _100%_ score, others have percentages of the fastest parser's score

<div grid="~ cols-5 gap-4">
<div col="span-3">

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vnyust8234v4y1vtsm6q.png)


</div>
<div col="span-2">
<br/>
<br/>

* ast-grep won when parsing medium to large files concurrently
* TypeScript/Tree-sitter experience a decline in performance with larger files
* SWC and Oxc maintain consistent performance

</div>
</div>

---

# Parse Time Breakdown

NAPI time can be dissected into three main components

<h1>

$$
\textit{time} = \textit{ffi\_time} + \textit{parse\_time} + \textit{serde\_time}
$$

</h1>


- **Foreign Function Interface**: A fixed cost to invoke functions across different languages

- **Parse**: a variable cost that scales with the size of the input

- **Serialization/Deserialization**: convert Rust data into a JS-compatible format. It may be a fixed or variable cost, depending on the implementation

<!--
In essence, benchmarking a parser involves measuring the time for the actual parsing (`parse_time`) and accounting for the extra overhead from cross-language function calls (`ffi_time`) and data format conversion (`serde_time`).

Understanding these elements helps us evaluate the efficiency and scalability of the parser in question.
-->

---
layout: two-cols
clicks: 3
---

# FFI Overhead

<br/>

<v-clicks>

* "one line" stands for the baseline FFI overhead
  * minimal parse/serde time
* FFI is less significant as file size grows
  * it's largely size-independent
  * ast-grep's perf increased from 72% to 78%
  * suggesting a rough 6% FFI overhead
* FFI is more pronounced in async parsing
  * ast-grep’s one-line perf: 72% sync vs 60% async
  * swc may have its [unique implementation](https://github.com/oxc-project/oxc/blob/2d5e0d5d0775300463f36b925e2f1ce71f119b90/napi/parser/src/lib.rs#L96).

</v-clicks>

::right::

<arrow v-click="[1, 2]" x1="170" y1="10" x2="170" y2="60" color="#564" width="2" arrowSize="1" />
<arrow v-click="[1, 2]" x1="170" y1="420" x2="170" y2="370" color="#564" width="2" arrowSize="1" />

<arrow v-click="[2, 3]" x1="270" y1="10" x2="170" y2="60" color="#564" width="2" arrowSize="1" />
<arrow v-click="[2, 3]" x1="300" y1="10" x2="400" y2="60" color="#564" width="2" arrowSize="1" />

<arrow v-click="[3, 4]" x1="230" y1="180" x2="180" y2="90" color="#564" width="2" arrowSize="1" />
<arrow v-click="[3, 4]" x1="230" y1="370" x2="180" y2="280" color="#564" width="2" arrowSize="1" />

<br/>

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rghu4pqjmutcueudt81s.png)

<br/>

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xs8ve7oelccjcrh1lbdj.png)

---
layout: two-cols
clicks: 2
---

# Serde Overhead

Unfortunately, we failed to replicate swc/oxc's blazing performance we witnessed in other applications.

<v-clicks>

* swc/oxc is slower than TSC
  * Caused by calling [`JSON.parse` on strings](https://github.com/swc-project/swc/blob/5d944185187402691292fdb73ea767bd580e2a52/node-swc/src/index.ts#L108)
  * Sending RS data is even slower than JSON
* Tree-sitter/ast-grep avoid serde overhead
  * By [returning a tree object](https://github.com/ast-grep/ast-grep/blob/1c3accfd7dccef293c480951759b86c418cde977/crates/napi/src/sg_node.rs#L297)
  * Tree nodes access requires [invoking Rust methods](https://github.com/ast-grep/ast-grep/blob/1c3accfd7dccef293c480951759b86c418cde977/crates/napi/src/sg_node.rs#L78) from JS
  * _Distribute the cost over reading_

</v-clicks>

::right::

<div v-click=[1,3]>

SWC's JSON parse
```ts {3-5}
parseSync(src, options, filename) {
  ...
  if (bindings) {
    return JSON.parse(
      bindings.parseSync(src, toBuffer(options), filename)
    );
  } else { ... }
}
```

</div>

<div v-click=[2,3]>

ast-grep's tree
```rs
#[napi]
impl SgRoot { ... } // in Rust
```
```ts
export class SgNode { // in JS
  text(): string
  parent(): SgNode | null
  children(): SgNode[]
}
// serde cost is amortized across reads
```

</div>

---
layout: two-cols
---

# Parallel Parsing

<br/>

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vnyust8234v4y1vtsm6q.png)

::right::

<br/>
<br/>

* JS parsers are slower when parsing concurrently

* They are CPU bounded because files must be parsed one by one on main the thread

* Almost all native TS parsers have parallel support, except tree-sitter

* Native will not parse more four files at the same time

---

#  Perf Summary

The perf summary table outlines the time complexity for different operations.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0rm26rv2o0jndzeubxib.png)

* `constant` denotes a constant time cost that does not change with input size

* `proportional` indicates a variable cost that grows proportionally with the input size

* `N/A` signifies that the cost is not applicable

---
transition: slide-up
---

#  Perf Summary

* JS-based parsers run entirely in JS Engine
  * No FFI or serde overhead
  * Only parsing time, which grows with the input file size

* Rust-based parsers is influenced by a fixed FFI cost, a variable parsing and serde cost
  * serde overhead varies depending on the implementation
  * ast-grep/tree-sitter have a fixed serialization cost of one tree object
  * swc/oxc have the serde costs proportional to the input size

* Native parsers can be parallel, while JS parsers cannot


---
layout: two-cols
clicks: 2
---

# Performance Tricks

How can we make Rust binding faster?

<v-clicks>

- **Avoid Serde Cost at beginning**

  - Return a Rust object wrapper to Node.js
  - Can lead to slower AST access in JS
  - The cost is amortized over the reading phase

- **Use multiple CPU cores**

  - NAPI can use `AsyncTask` for multi-cores
  - `AsyncTask` is scheduled on [libuv threads](http://docs.libuv.org/en/v1.x/threadpool.html)
  - libuv thread pool size is set to four by default
  - [Expand thread pool size](https://dev.to/bleedingcode/increase-node-js-performance-with-libuv-thread-pool-5h10) can improve perf

</v-clicks>

::right::

<div v-click=[2,3]>

```cpp {1-3}
// async work is scheduled on libuv threads!
napi_status NAPI_CDECL napi_create_async_work(...) {
  uvimpl::Work* work = uvimpl::Work::New(
    evn, resource, name, execute, complete, data);
  ... // more omitted
}
```

![libuv](/libuv.png)

</div>

---
layout: image-right
image: https://source.unsplash.com/collection/94734566/1920x1080
---

# Thanks

<br/>

### Thank [Seattle.JS](https://seattlejs.com/)


* If you find this talk helpful, please [give it a star](https://github.com/ast-grep/ast-grep) and share it with your friends!

* [https://github.com/ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)

