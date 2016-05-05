# Syntactic Tail Calls (STC)
Discussion and specification for an explicit syntactic opt-in for Tail Calls, dubbed Syntactic Tail Calls or STC.

Example program:

```js
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }

  return continue factorial(n - 1, acc * n)
}
```

Or, using an arrow function:

```js
let factorial = (n, acc = 1) =>
  n == 1 ? acc
         : continue factorial(n - 1, acc * n);
```

## History &amp; Background
ES6 specified Proper Tail Calls (PTC). With PTC, calls that are [in tail position](http://www.ecma-international.org/ecma-262/6.0/index.html#sec-isintailposition) [must not create additional stack frames](http://www.ecma-international.org/ecma-262/6.0/index.html#sec-preparefortailcall).

During deliberation, there was apparently much discussion about the relative merits of PTC weighed against the potential downsides including difficulties in debugging scenarios and possible implementation difficulties. Unfortunately, the TC39 process at this time did not require heavy implementation involvement and so while many implementers were skeptical, the feature was included and standardized as part of ES6.

Recently implementations have begun taking a serious look at implementing tail calls. Some implementations have not had significant concerns, while others have. Many, if not all, of these concerns could be addressed by changing the design of PTC to require some sort of syntactic opt-in. This repository is an attempt to create a proposal for that syntactic opt-in as a replacement to the PTC specification in ES6.

## Merits of PTC

PTC has the nice property that it "just works" - programs written in such a way that they can take advantage of proper tail calls take advantage of them with no additional work for the developer. Indeed a developer may not even be aware that PTC exists and yet the runtime can optimize stack usage of their code regardless.

Additionally, programs that may work today for small inputs may work with much larger inputs in engines that support PTC.

(Proponents should probably flesh this out into sub-sections. Volunteers?)

## Issues with PTC

A number of issues have been raised with the PTC design. While many of these issues were considered as part of the pre-ES6 specification process, not all of them were, and not all were backed by actual implementation experience and data. Further, while the champion group recognizes that PTC and related concepts have been studied and debated extensively for the last few decades, we believe it can't hurt to reexamine these issues in the context of the constraints developers and implementers are operating under today.

### Performance

There is a belief among some implementers and developers at large that PTC is an optimization strategy and that by enabling PTC, an engine will be making code run faster. This belief is validated by the JSC team's implementation which shows some performance wins in some cases. However, this belief is invalidated by the v8 team which sees it as mostly performance neutral, and the Chakra team which, due to other constraints, may not be able to implement the feature without regressing performance of existing code that happens to include tail calls or sacrificing spec conformance.

Because PTC automatically opts-in a bunch of web code into using tail calls, a performance degradation is a very serious concern. (See #7). STC side-steps this problem by making it intentional when to use the feature which can involve an evaluation of implementations and whether using the feature will result in desirable performance characteristics for the unique scenario.

### Developer Tools

Developers rely extensively on call stacks to debug their code. Implementing PTC will result in many of those frames being missing. This can be mitigated by implementing some form of shadow stack (Eg. JSC's ShadowChicken) which maintains a side-stack of frames that can be presented to users during debugging. This comes with a cost and so would likely only be enabled with dev tools open (and thus doesn't solve the Error.stack issue below). It is also not a silver bullet with potential problems such as introducing GC indeterminism to debugging workflows. (See #6).

As an example, consider this simple program:
```js
function foo(n) {
  return bar(n*2);
}

function bar() {
  throw new Error();
}

foo(1);
```

Here the call to bar will re-use the frame from foo since the call to bar is in tail position. Therefore, in Error.stack and in the dev tools (without the mitigations considered above), the foo frame will be missing.

STC side-steps this problem by making it an intentional choice when to elide frames. Developer tools would not need to display every frame to the developer because the developer has opted in to eliding those frames. Or, developer tools could use the same best-effort approach required to PTC to give a different and potentially better experience.

### Error.stack

JavaScript exceptions will have different error.stack information with PTC enabled because of the elided stack frames. Consider again the example from above:


```js
function foo(n) {
  return bar(n*2);
}

function bar() {
  throw new Error();
}

try {
  foo(1);
} catch(e) {
  print(e.stack);
}

/*
output without PTC
Error
    at bar
    at foo
    at Global Code

output with PTC (note how it appears that bar is called from Global Code)
Error
    at bar
    at Global Code
*/
```

This may be a problem for developers who use this information to debug their programs. However, assuming we cannot recover all of the elided frames when not doing actual debugging, there is concern that web analytics and telemetry applications will be impacted by some browsers having different callstacks for the same errors. This can also be mitigated to some degree by updating all of the services that collect these stacks to expect elided frames. However this is not super comforting in the short term.

STC avoids this problem by making tail calls an explicit opt-in. Code that works today continues to work including any error.stack information collected and reported to servers.

There is the additional complexity with how to standardize Error.stack in light of elided frames. STC gives us an obvious path forward, but the path forward is less obvious with PTC. (See #6)

### Cross-Realm Tail Calls

PTC exposes previously hidden implementation details about function objects. In particular, Mozilla has voiced that it will be impossible for them to implement PTC across realm boundaries because of their membrane-based security model. While there has already been some discussion (see #2) of the merits of these claims, and [an open PR](https://github.com/tc39/ecma262/pull/508) to address these issues in the current spec, STC provides a convenient opportunity to revisit these concerns directly, and to allow for implementations to be compliant by throwing warnings in this case, if it is impossible to complete the tail call while consuming linear resources.


### Developer Intent

Even with workarounds for the above issues, there is a question whether developers would benefit from PTC in ways they wouldn't with STC. STC has the benefit of encoding user intent and thus making clear when an algorithm is explicitly depending on tail call semantics to function properly. This avoids refactoring hazards among other things.

Another potential benefit of STC is that we know what the developer is intending and can give warnings or errors for, for example, claiming they're about to do a tail call and not putting the call in tail position.

## Proposal

This proposal wishes to advance a syntactic opt-in for tail calls. Presented above is one likely alternative: `return continue`. The `return continue` statement indicates that what follows is a tail call and thus should not grow the stack. Using `return continue` with a non-tail call can be syntax error in most cases (eg. `return continue 1 + foo()` would throw early). `return continue` would also enable errors or warning for syntactically proper tail calls that cannot be guaranteed to not grow the stack, for example cross-realm calls in FireFox. Additionally, developers opting to use `return continue` can manage the complexity of elided stack frames and thus less invasive solutions (or, perhaps, no solution) is required for debugging scenarios, and the semantics of existing code is maintained.


### Syntax Alternatives

The syntax for this proposal is being discussed in #1. Options include discussed include opting in at the function level, statement level (ala return), or at the expression/callsite level. Strawmen on the table presently include:

#### Return Continue

```js
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }

  return continue factorial(n - 1, acc * n)
}

let factorial = (n, acc = 1) => continue
  n == 1 ? acc
         : factorial(n - 1, acc * n);

// or, if continue is an expression form:
let factorial = (n, acc = 1) =>
  n == 1 ? acc
         : continue factorial(n - 1, acc * n);
```

#### Function sigil

```js
// # sigil, though it's already 'claimed' by private state.
#function() { /* all calls in tail position are tail calls */ }

// Note that it's hard to decide how to readably sigil arrow functions.

// This is probably most readable.
() #=> expr
// This is probably most in line with the non-arrow sigil.
#() => expr

// rec sigil similar to async functions
rec function() { /* likewise */ }
rec () => expr
```

#### !-return

```js
function () { !return expr }

// It's a little tricky to do arrow functions in this method.
// Obviously, we cannot push the ! into the expression, and even
// function level sigils are pretty ugly.

// Since ! already has a strong meaning, it's hard to read this as
// a tail recursive function, rather than an expression.
!() => expr

// We could do like we did for # above, but it also reads strangely:
() !=> expr
```
