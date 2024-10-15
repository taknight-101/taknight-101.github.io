---
layout: post
title: "An Orchestration Pattern in Frontend Web Component Based Frameworks"
permalink: "orchestrator_pattern"
categories: [Platform Engineering, ServiceNow]
tags: [
    Front-end,
    Web Component Standard,
    REST APIs,
    GIS,
    leaflet,
    reflective programming,
  ] # TAG names should always be lowercase
# image: /images/utilities/1.jpg
---

## Preface

In this blog, i want to explain how i built a solution architecture that overcame lots of business challenges through software design principles & programming paradigms

## Intro

To start straight, this is the visual diagram of the solution architecture proposed

![]({{ site.baseurl }}/images/orchestrator_pattern/SA.png)

As can be interpreted from the glossary of the visual above, the solution environment is the servicenow platform, and to elaborate the environment context to the subject, the platform offers a component based frontend framework called `Now Experience Framework`, its docs can be found [here](https://developer.servicenow.com/dev.do#!/reference/next-experience/xanadu/ui-framework/getting-started/introduction), And it's a react like library with standard web component principles like component state, component life cycle hooks, and, props

The original problem was to embed GIS algorithms and map visualization that are not native to the platform or to the now experience framework

But as The framework offers means to implement custom components that are then used in a component tree to be displayed within a page in the platform, and there are action based events & triggers that whichever component can make use of to communicate with the component in question, the main idea being that, you implement an atomic component and deploy it making sure it has hooks for communication with other already deployed components

And based on that, The approach i've taken to design the solution architecture above was the following:

- Development flexibility → having clear separation between component code & map code to minimize coupling and speed up the deployment process through calls to hosted REST APIs

- Having an agnostic solution with respect to business use cases that not only works with mapping related features, but for any other alike use case via iframe embedding and native web technology features enablement like html/css/js

---

The main player in the architecture is the custom component implemented in the visual called `Map Component`, as it's the orchestrator or the combiner of 2 critical pieces to achieve the main functionally of the solution, and for all intents and purposes any application logic for that matter, recall that the architecture not only aims at solving the mentioned business problem, but rather any such problem that stems in such sandboxed environment.

As any program we write is composed of 2 pieces `code + data`, where code is the instructions the machine performs on the data to be processed to extract meaning or value from,
the same principle is employed here by the map component being the orchestrator, as it first:

1. Fetches the data from REST APIs
2. Fetches the Code also from a REST API
3. Glues the 2 pieces together, and that's the tricky part

the 3rd point actually resembles a programming paradigm - one of my favorite in all times - and that's `Reflection` and in short terms, is a way to treat code in itself as data, this allows you for introspection of your code structure, layout, data types, and many more, and you won't believe how much this is useful and frankly used a lot in libraries internal code ,and many frameworks you might use on a daily basis is built on top of that 1st principle, also this mechanic has own interface & implementation in interpreted & compiled languages, examples are python's [introspect module](https://book.pythontips.com/en/latest/object_introspection.html), and [Java Reflection API](https://www.javatpoint.com/java-reflection)

Following is a code snippet of the solution implementation for the idea above, and how it made it possible to combine and inject dynamic data with code treated as static data

```javascript

[MAP_CODE_FETCHED]: ({
			state: { agents, work_order_tasks },
			updateState,
			action: { payload },
		}) => {
			let passive_htmlContent = payload.result.result;


			passive_htmlContent = passive_htmlContent.replace(
				"passed_value",
				JSON.stringify(agents)
			);

			passive_htmlContent = passive_htmlContent.replace(
				"tasks_passed_value",
				JSON.stringify(work_order_tasks)
			);

			// updateState({ htmlContent: payload.result.result });
			updateState({ htmlContent: passive_htmlContent });
		},

```

In the code snippet above, as the event `MAP_CODE_FETCHED` is dispatched, the following arrow function is invoked which accepts some dynamic data fetched from rest apis in previous steps and stored in component state, and then lastly the event's payload having the `passive_htmlContent` which is the static code to be then merged with that dynamic data.

The following lines do exactly that, as explained earlier, as the code here is treated as a string, the code fetched is filled with placeholders for dynamic data to be injected at, and that's simply achieved by a simple string.replace() call in js, to merge the 2 pieces and then update the state to render the UI with the full logic intended.

---

This approach satisfies the open closed principle in that any modification to the code is done elsewhere and further updates are fetched from the api as any else, but the component code is written & deployed once.

Also the overall solution is extendible because as mentioned in the approach taken in mind during design time, this code is then to be embedded within an iframe `srcdoc` attribute, that accepts a vanilla html/css/js page with all it involves regarding algorithms and logic.

During my Career, i approached the reflection programming paradigm in many use cases ,ranging from application development use cases to even cybersecurity use cases, and from my point of view, this paradigm's usefulness is endless

That's all for now folks, and see you in another blog :)
