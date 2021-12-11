---
title: "Make libraries, not configs"
date: 2021-12-12T01:10:00+03:00
draft: true
description: A case study of how tools can be made more flexible by being extensible with code instead of configuration
---

While researching load testing tools for an API we build at work, we discovered [Artillery](https://www.artillery.io/). It's a tool that allows you to load test APIs by specifying your requests in a YAML file. This YAML file is then interpreted and used by the tool to hammer a particular API.
It is powerful, as in provides a lot of functionality and configurability, and seems to strike the right balance between usability and flexibility. This essay argues that the latter isn't completely true and that there is a better way to achieve the same functionality while simultaneously simplifying the implementation.


## The example-driven development model

It is a regular occasion to encounter a library that has a great website with a beautiful UI and concise examples in which everything seems to happen almost magically. But then you start using it and as soon as you need to stray off the beaten path you begin pulling your hair out. I call this "example-driven development" and it particularly plagues new libraries and frameworks in JavaScript-land. It's common for developers to exaggerate their own use-cases while downplaying functionality and flexibility they don't themselves need. It is only natural and is observed in not only developers but all people and all areas of life.

This is especially visible in comments of the type "You shouldn't need to ever do that". Yes, please educate me on alternatives and why it is not generally a good idea to do it. However, the real world is not perfect, and I may have to do it regardless, please give me a hook or an integration-point where I can do it myself. If a tool is using Webpack I should be able to access that Webpack configuration without having to "eject" from the tool completely. I'm looking at you, Angular CLI. Just give me an `augmentWebpackConfig(config: WebpackConfig): WebpackConfig` function and annotate it with "DO THIS AT YOUR OWN RISK" if you'd like.


## All I want for Christmas is functions

Back to Artillery. Artillery is great when you read the documentation. And also when you experiment with it on a PoC. You would even be very happy using it for small projects or microservices. The deficiency comes when you have many requests and you want your load tests to model real-life traffic and common user flows. An Artillery scenario (list of requests) looks like this:

```yaml
config:
  target: "https://example.com/api"
  phases:
    - duration: 60
      arrivalRate: 5
      name: Warm up
    - duration: 120
      arrivalRate: 5
      rampTo: 50
      name: Ramp up load
    - duration: 600
      arrivalRate: 50
      name: Sustained load
  payload:
    path: "keywords.csv"
    fields:
      - "keyword"

scenarios:
  - name: "Search and buy"
    flow:
      - post:
          url: "/search"
          json:
            kw: "{{ keyword }}"
          capture:
            - json: "$.results[0].id"
              as: "productId"
      - get:
          url: "/product/{{ productId }}/details"
      - think: 5
      - post:
          url: "/cart"
          json:
            productId: "{{ productId }}"
```

The example above has just 3 very simple requests. Now imagine having 10 flows each including 20 requests. At that point you would most definitely want to split those requests into pieces and name them so that they are reusable. If you try to imagine it, it would probably look like this:

```yaml
config:
  ...

scenarios:
  - name: Search and buy
    flow: @include 'flows/search-and-buy.yaml'
  - name:
    flow: @include 'flows/just-browsing.yaml'
  - name:
    flow: @include 'flows/adding-to-cart.yaml'
```

Here you start thinking about scripts that pre-process your Artillery files. You encounter some Python lib that does something similar, so you integrate it.

At some point you realize that you can share specific steps in a flow. In all flows you have to log in. In most you want to search for some item using your search endpoint. In more than one you want to add an item to your cart. You start wanting to split those flows into reusable sets of requests:

```yaml
flow:
  - post:
      url: "/search"
      json:
        kw: "{{keyword}}"
      capture:
        - json: "$.results[0].id"
          as: "productId"
  - get:
      url: "/product/{{productId}}/inventory"
```

So you expand your YAML-preprocessing scripts to support such fragments. But for these to be truly reusable you'd want them to be configurable. You'd want the ability to give them parameters (e.g. `keyword`, `productIndex`) and you'd want to be able to receive a value back, like the `productId` that was found. And it seems like it can work if you put the above into a separate file and inject it in multiple places. But those variables are global, so you start having to trace and coordinate variables through multiple files. **What you want is a function.** A function can take parameters, return values and has a local scope that won't override something else in a completely-separate file.

Function support would be the ideal solution. It completely solves these problems. And if Artillery workflows were represented with a programming language and not a config file, it would have been something that was available at no implementation cost.


## Configs are easier to understand... to a point

But wait, I hear you say, YAML is just structured data and so it's easier to understand. I would counter that with an example:

```ts
export default new LoadTest({
  config: {
    target: 'https://example.com/api',
    phases: [
      { name: 'Warm up', duration: 60, arrivalRate: 5 },
      { name: 'Ramp up load', duration: 120, arrivalRate: 5, rampTo: 50 },
      { name: 'Sustained load', duration: 600, arrivalRate: 50 }
    ],
    payload: { path: 'keywords.csv', fields: ['keyword'] }
  },

  scenarios: [
    new Scenario('Search and buy', async ({ payload, request, sleep }) => {
      const { results } = await request.get('/search', {
        body: { kw: payload.keyword },
      })

      const productId = results[0].id

      await request.get(`/product/${productId}/details`)

      await sleep(5)

      await request.post('/cart', {
        body: { productId }
      })
    })
  ]
})
```

Compare this to the original YAML version:

```yaml
config:
  target: "https://example.com/api"
  phases:
    - duration: 60
      arrivalRate: 5
      name: Warm up
    - duration: 120
      arrivalRate: 5
      rampTo: 50
      name: Ramp up load
    - duration: 600
      arrivalRate: 50
      name: Sustained load
  payload:
    path: "keywords.csv"
    fields:
      - "keyword"

scenarios:
  - name: "Search and buy"
    flow:
      - post:
          url: "/search"
          json:
            kw: "{{ keyword }}"
          capture:
            - json: "$.results[0].id"
              as: "productId"
      - get:
          url: "/product/{{ productId }}/details"
      - think: 5
      - post:
          url: "/cart"
          json:
            productId: "{{ productId }}"
```

The code version is shorter and infinitely more configurable. I can easily reuse parts of the logic by extracting them into functions - no YAML-scripting schenanigans.

> But YAML will get you started more quickly

If you have never programmed in JavaScript - sure, that's true. However if you have, you nevertheless have to learn Artillery's specific YAML; you have to read through the documentation and keep consulting with it in at least the first week of working with the tool.

If you don't know JavaScript and the tool makes you use it - learning just enough JS to get to the same level of functionality would arguably take a similar amount of time. But learning some small amount of JS would be useful in many other places, not just this one. The YAML you learn here is unique to Artillery and will not translate to anything else.

I'd also argue that someone who can't be bothered to learn how to use variables and call simple functions in JS shouldn't be writing your performance tests in the first place.


## Programming languages are more verbose


Personal opinion here, but this:

```ts
const response = await request.get('/search', {
  body: { kw: payload.keyword },
})
const productId = response.results[0].id
```

is much more natural, concise and straightforward than this:

```yaml
- post:
    url: "/search"
    json:
      kw: "{{ keyword }}"
    capture:
      - json: "$.results[0].id"
        as: "productId"
```

You've done something like the first example a million times. You've done something like the second only if you've used Artillery before. If using a statically typed language (e.g. TypeScript), you even get to use type suggestions and validation instead of having to always keep the Artillery manual open on your second monitor.

And the YAML option is so much less flexible - the `capture` used here must be explicitly implemented by the authors. If you want to, for example, get a random product from the result, the first code will change just a tiny bit:

```ts
import { sample } from 'lodash'

const productId = sample(response.results).id
```

while doing this in a pure-YAML version is impossible unless specifically supported with special-case functionality. Artillery supports [resorting to JS functions](https://www.artillery.io/docs/guides/guides/http-reference#loading-custom-js-code) in such cases, however this is extremely ugly and forces you to read through yet-another guide on how to use something Artillery-specific. Also good luck making those JS functions support actual input parameters and not just a dictionary with a reference to the whole world.


## Library APIs are hard to get right... and so are configs

Putting your tool developer hat on, whichever approach you choose, you'd have some things to figure out:

When building a tool based on APIs, you have to figure out what level of abstraction you want to achieve. Too low and using your library would be a chore and an extremely verbose mess. Too high and you fall into the config territory where you have to implement every single customization into your tool. You can't just take a YAML approach and put it in a single "config" object in JS. Well, you absolutely can, but you've missed the whole point.

When building using configs, you have to again figure out what you support. However, the format limits you to just configuration. Any logic you have to represent in your format you have to implement yourself. You'd have to implement [variables](https://www.artillery.io/docs/guides/guides/http-reference#extracting-and-re-using-parts-of-a-response-request-chaining), [conditions](https://www.artillery.io/docs/guides/guides/http-reference#conditional-requests), [loops](https://www.artillery.io/docs/guides/guides/http-reference#loops) and [functions](https://www.artillery.io/docs/guides/guides/http-reference#writing-custom-logic-in-javascript), for God's sake. Oh, and don't forget to provide your users a way to [debug](https://www.artillery.io/docs/guides/guides/http-reference#debugging) their YAML-programs, because they can't just put a `console.log(response)` on the request they want. At this point you've implemented your own crippled programming language where every variable is global and you can't write functions. Good job and welcome to the 1950s, have a pleasant stay!

You would have to put more effort to build sufficiently flexible and generic config-based tool, than to build a DSL / library with an almost-infinite flexibility. Your users will also have to put more effort into understanding and becoming proficient in your way of doing an "if", "while" and variable assignment in YAML.


## YAML is for data, code is for logic

The fundamental idea is that **code is the best and most appropriate representation of logic**. If you need to model logic - you can stand on the shoulders of giants and use an actual programming language. If you need to represent data - then by all means use YAML or JSON or XML. [ColdFusion](https://www.quackit.com/coldfusion/tutorial/coldfusion_loops.cfm) has proven that markup languages are not the best at describing logic.


## The line is, unfortunately, blurred

Of course, distinguishing between data/configration and logic is sometimes difficult. It's also possible that what starts as a simple config-only tool with a couple of properties grows into a DIY-programmming-language monster (as I suspect happened with Artillery). However, if you're building a tool it's better to start with something that will provide room for growth. Because if you hit a wall you'd have to break the whole house down and build it again, which will inevitably alienate a big percentage of your users.

I hope the next time you want to build a tool, or need to choose between tools, you think back to this dilemma and make the best decision.
