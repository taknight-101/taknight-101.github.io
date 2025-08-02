---
layout: post
title: "Devlog #2: Refactoring Critical Business Logic in a Live Spring Boot System — Design, Pain, and Patterns"
permalink: "core-refactor"
categories: [DevLog, Backend]
tags: [Devlogs, Spring-boot, Refactor, Design Patterns] # TAG names should always be lowercase
image: /images/refactor/betweenNoise.png
---

## Intro — Setting the Stage

Working within a Spring Boot microservices architecture, My mission with refactoring a large, critical business flow deeply embedded in a legacy monorepo structure with dozens of Maven modules. As is often the case with legacy systems, the existing codebase lacked reusability, maintainability, and extensibility. It was difficult to test, brittle under change, and hard to extend without side effects.

This devlog shares how I tackled the following three challenges:

1. Understanding a massive, unfamiliar codebase quickly.

2. Refactoring complex logic into a reusable, generic component architecture.

3. Maintaining development progress while safely testing the refactor live.

## Challenge 1: Understanding a Huge Codebase in Little Time

My first move was to isolate the critical path of the business flow. Rather than trying to understand everything at once, I focused on the parts of the system that were essential for the business to function.

Before diving into the code, it's crucial to develop a solid understanding of the business process behind the system you're working with. This includes becoming familiar with domain-specific terminology, identifying edge cases, and grasping the expectations and pain points from the business perspective. This foundational knowledge helps avoid misinterpretations and ensures that technical solutions align with real-world business needs.

Using BPMN diagrams and call traces, I identified:

1. The exact sequence of business steps.

2. Which components were critical (i.e. blocking or transactional).

3. Which components could be treated as auxiliary or delayed.

This allowed me to create a mental map of the flow, and to begin reasoning about refactoring without needing to detangle the entire system upfront.

## 📘 A Quick Note on BPMN

![BPMN Flow Diagram]({{ site.baseurl }}/images/refactor/BPMN-1.png)

For those unfamiliar, BPMN (Business Process Model and Notation) is a standardized graphical notation for modeling business processes.

Events (circles) represent something that happens (start, end, or intermediate).

Tasks (rectangles) represent units of work.

Gateways (diamonds) handle decisions or forks in the flow.

This is a great <a href="https://app.crismo.io/" target="_blank">tool</a> to use if you want to create your own BPMN diagrams.

---

Using BPMN helped clearly distinguish:

- Critical path steps: Essential to business value, must succeed.

- Optional/supporting steps: Can be retried, postponed, or omitted safely.

- It’s a powerful way to communicate complex flows between developers and domain experts, and it’s a valuable reference when restructuring business logic.

## Challenge 2: Refactoring into a Reusable Pattern

❌ The Problem with the Old Code

The legacy flow was driven by deserializing a linked-list metadata structure describing each business step. It also managed a global, in-memory state context throughout the execution.

This introduced serious design flaws:

1. No SRP (Single Responsibility Principle): State management logic was tangled with business logic, violating the idea that each module or class should have one and only one reason to change.

2. Side effects: Unexpected failures would leave context in invalid states.

3. Poor testability: Hard to isolate components.

4. Difficult to short-circuit the flow safely.

✅ My Approach: Clean Composition + DDD Principles

I introduced a reusable component architecture:

> Modular design using Domain-Driven Design (DDD) concepts.

In DDD (Domain Driven Design), an Aggregate Root is the entry point to a cluster of domain objects that are treated as a single unit. It encapsulates all business rules and ensures the consistency of changes.

For example, in an `e-commerce` domain, `Order` might be an aggregate root that coordinates `OrderItems`, `ShippingDetails`, and `PaymentStatus`. External systems interact with the `Order` aggregate rather than its internals.

In my case, each refactored business step was modeled as a well-bounded aggregate that exposes only a stable and minimal API, enforcing both consistency and separation of concerns.

The project structure was also restructured to reflect this separation — organizing aggregates and their internal logic into dedicated modules, making the codebase easier to navigate, maintain, and extend.

Core Interface:

```java
interface Component<I, O, E> {
OperationResult<O, E> execute(I input);
}
```

Each business step became a clean, focused component implementing that contract.

> <a href="https://www.geeksforgeeks.org/system-design/template-method-design-pattern/" target="_blank">Template Method pattern</a> applied via composition, not inheritance.

Traditionally, the Template Method pattern is implemented using an abstract class with concrete subclasses overriding specific steps of an algorithm. However, The choice of choosing composition, allowed more flexibility and modularity for the use case at hand.

Here's how It might be structured:

```java
// Generic result wrapper
public record OperationResult<O, E>(O output, E error, boolean success) {
    public static <O, E> OperationResult<O, E> ok(O output) {
        return new OperationResult<>(output, null, true);
    }
    public static <O, E> OperationResult<O, E> fail(E error) {
        return new OperationResult<>(null, error, false);
    }
}

// Generic component interface
public interface Component<I, O, E> {
    OperationResult<O, E> execute(I input);
}

// A reusable wrapper to orchestrate components with optional hooks
public class ComponentRunner<I, O, E> {
    private final Component<I, O, E> coreComponent;
    private final Runnable before;
    private final Runnable after;

    public ComponentRunner(Component<I, O, E> coreComponent, Runnable before, Runnable after) {
        this.coreComponent = coreComponent;
        this.before = before;
        this.after = after;
    }

    public OperationResult<O, E> run(I input) {
        if (before != null) before.run();
        OperationResult<O, E> result = coreComponent.execute(input);
        if (after != null) after.run();
        return result;
    }
}

// Example usage
Component<String, Integer, String> parser = input -> {
    try {
        return OperationResult.ok(Integer.parseInt(input));
    } catch (NumberFormatException e) {
        return OperationResult.fail("Invalid number");
    }
};

ComponentRunner<String, Integer, String> runner = new ComponentRunner<>(
    parser,
    () -> System.out.println("[Before Execution]"),
    () -> System.out.println("[After Execution]")
);

// Execute the component and handle short-circuit logic
OperationResult<Integer, String> result = runner.run("123");

if (!result.success()) {
    System.out.println("Flow stopped due to error: " + result.error());
    return;
}

System.out.println("Result: " + result.output());

```

Note that the following lines:

```java
Component<String, Integer, String> parser = input -> {
    try {
        return OperationResult.ok(Integer.parseInt(input));
    } catch (NumberFormatException e) {
        return OperationResult.fail("Invalid number");
    }
};

```

uses functional style in Java via a lambda expression.

and that's because

```java
Component<String, Integer, String>
```

is a functional interface (i.e. it has a single abstract method: execute(I input)).

`input -> { ... }` is a lambda implementing that method.

This is equivalent to writing a verbose anonymous class:

```java
Component<String, Integer, String> parser = new Component<>() {
    @Override
    public OperationResult<Integer, String> execute(String input) {
        try {
            return OperationResult.ok(Integer.parseInt(input));
        } catch (NumberFormatException e) {
            return OperationResult.fail("Invalid number");
        }
    }
};
```

But thanks to Java’s support for lambdas (Java 8+), the same logic is expressed more concisely.

⚖️ Benefits

1. Uniform contract made it easy for teams to understand and use.

2. Generics enabled strict typing for each component's input/output/error.

3. Reusable PipelineExecutor managed orchestration and allowed for short-circuiting.

The result: a closed for modification, open for extension architecture that can be safely extended for future flows or variations.

For example, when a new business requirement arises that introduces a new validation rule or processing step, a developer can simply implement a new component that conforms to the existing `Component<I, O, E>` interface. This component can then be plugged into the orchestration pipeline without modifying any of the existing components. Similarly, if a new integration system needs to be added, it can be registered as an isolated module and handled consistently through the same architecture, reducing the risk of regressions or side effects in already stable logic. Even if the new system doesn't conform to the `Component<I, O, E>` interface, a design pattern such as <a href="https://www.geeksforgeeks.org/system-design/adapter-pattern/" target="_blank">Adapter</a> can be employed within that module to bridge the external system's contract to the internal architecture — maintaining both cohesion and alignment with Domain-Driven Design principles.

## Challenge 3: Parallel Refactor Without Breaking Existing Development

With other developers actively building features on the old flow, I couldn’t afford to disrupt ongoing work.

The approach i took was inspired by a concept from **cloud-native deployments** — namely, <a href="https://martinfowler.com/bliki/CanaryRelease.html" target="_blank">canary releases</a> — where new versions of a service are deployed to a small subset of users or traffic before full rollout. I brought this principle into my code refactor: instead of replacing the old business flow outright, I ran the new implementation in parallel, allowing it to coexist while testing its correctness and adoption feasibility.

![Canary Diagram]({{ site.baseurl }}/images/refactor/canary.png)

So by using canary testing:

- My new implementation ran in parallel, exposed via new controller endpoints.

- I validated the output against the old system.

- Failed refactor attempts (like an initial gateway-interface procedural approach i took initially but didn't prove its adoption feasibility) were safely discarded without impact.

This strategy let me iterate rapidly while giving the team confidence that the new system wouldn't break theirs.

Though my plan didn't go 100% smooth initially and some team developers had to rework some of their integration with my core, it didn't happen again — thanks to continuous feedback and quick iterations.

As Martin Fowler once said: _"If it hurts, do it more often."_ This experience proved that embracing early integration and collaborative refinement leads to stronger long-term stability.

## Lessons Learned

- Great software isn't just "working" code — it's code that works for developers.

- Refactoring is as much about developer experience as it is about correctness.

- Using DDD principles, generics, and pattern composition enabled a flexible, robust architecture.

- Communication is everything: I kept the team in the loop and earned trust by proving value incrementally.

## Conclusion

If you find yourself in a similar situation, remember:

"When developers are your clients, improving their lives improves your system."

- Plan for dual-track development.
- Validate as you go.
- Communicate transparently.
- And above all, don’t just refactor code — refactor the developer experience itself.
