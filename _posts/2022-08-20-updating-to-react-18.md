---
layout: post
title: React moved into adulthood. Reflections on updating React to v18
description: Thoughts on updating to v18 in a big Typescript project
tags: [JS, React, Typescript, React 18, ceateRoot API]
published: true
comments: true
---

React 18 is out. I mean... it's been available for over four months and I've just finished a kinda challenging process of migration from 17 to 18 in a huge JS app which is a mature SaaS product. I guess I completed it since you never know when the monster named tech debt could bite you. Anyways, I'd want to share some views on updating to 18v here, in the hopes that someone would find them useful.

If you're unfamiliar with the topic of version 18, [read the official docs of the latest React](https://reactjs.org/blog/2022/03/29/react-v18.html).

## The new API

Before considering bumping, please note - React's most significant API has been modified. This means the commonly known and essential root mounting with `ReactDOM.render` is deprecated. Of course, it's still supported (due to backward compatibility), but it produces irritating warnings in production env and floods specs run. Hint: a crazy spy may save us from the warnings flood in a spec run, but nothing will help in production except replacing the old API with the desired new `ReactDOMClient.createRoot`.

> Only the use of the createRoot API enables all React 18 features; otherwise, it behaves like v17. It's not worth updating if you don't have time to refactor.

I once went to a tech conference where one of the speakers was convincing the audience that it's not crazy to consider releasing tech products with a failure. Failures bring crises, crises - the best solutions. I don't take a stupid warning a failure, but what is most important the API replacement may fail in big JS projects of course. I assume the cost of the bump, disappointment, wasted time, tech debt to pay off etc. Maybe it's not worth making it at this time, following the YAGNI principle. So, before updating the API in crucial parts, consider what would your customers do if an app didn't work for one hour? Especially when test coverages are not credible and only a happy path is checked on your CI. That was not my case, but it's worth asking questions: is it a good time to do such updating? Do I really need the React 18 features now? What profits does it bring to my project? Please take such updating as a huge task to accomplish.

## Code updates

Apart from replacing `ReactDOM.render` with `ReactDOMClient.createRoot`, the whole bump entails code updatings, like:

- Each browser's event mock must be wrapped into `act` (as well as the mounting itself) if a component is run with `createRoot` in specs. Consider ~2k places you have to update to make specs green again. You may think a script which updates everything in less than a second with fancy regex may save us time. Believe me, ~10-15% of occurrences need special care. That means hours of debugging and figuring out what's wrong with the spec setup or assertion.

- Because `unmountComponentAtNode` is deprecated, a `Root` instance must be accessible on unmounting.
  Before, the obsolete API took a node and unmounted components at any time. Currently, the `createRoot` returns a `Root` object which has two methods: `unmount` and `render`. That's the next reason for refactoring such places since direct access to a Root instance is needed.

- I also discover issues with updates to a state with v18. The cause: one of the new features is in action: [_automatic batching_](https://reactjs.org/blog/2022/03/08/react-18-upgrade-guide.html#automatic-batching). As alwyas, in such cases, you can disable it or refactor a component in question taking the new behavior into account. Disabling is done by `flushSync` helper.

- Installed third-party libraries might not support React 18 at this time or never will. Some of them might have code built using outdated ideas and APIs. Without a general rewrite, they will no longer be able to support version 18 at all. Checking the cost of swapping out one package for another or making a contribution is highly advised. Note: NPM provides the —legacy-peer-deps switch; when used, it suppresses warnings for peerDependencies that are not supported.

- If your project is a framework-like app, such as an SDK, and you need to support both v17 and v18 at the same time, this is today not possible in a clean way, in my opinion. The 'createRoot' function is imported from 'react-dom/client' while the 'render' function is imported from 'react-dom'. You can try to devise various workarounds, such as exporting it from the same (sub)module or proxying it. Dynamic imports can also enter the game, although all solutions are basically hacks hard to maintain.

- If you use a Node server as an app that generates a static markup or so, you will also need to update the API, eg. `hydrate` is now `hydrateRoot`. Some methods are marked deprecated compared to the v17 like `renderToNodeStream`, some are new like `renderToPipeableStream`. You have to delve into documentation and consider the usage which may cause side-effects you cannot expect.

## Types updates

If you use Typescript with React as I do, you must fix the issues that the most recent types package will point up, like:
<br>

- There is no longer support for implicit children given to a component; this was convenient for devs using v17 or lower because it involved less code to write. Children must currently be explicitly declared in prop definitions or via one of the built-in types, such as `PropsWithChildren`. What was the motivation for the change? According to the author of the patch, implicit children are an excess prop (one that is sent to a component but is not actually handled by a component) that violates the [POLA (principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment#:~:text=The%20principle%20of%20least%20astonishment,not%20astonish%20or%20surprise%20users.) rule of software design. The goal, as always, was to increase consistency and eliminate excesses.

- No more `React.SFC` or `React.StatelessComponent` and similar types that start with `SFC-`. They've been replaced by `React.FC` / `React.FunctionComponent`. I delighted with the naming change, which was never correct in my opinion because 'Stateless' was a synonym for a function component, but nobody takes into account that class components might also be stateless. That was puzzling because the naming indicated the implementation detail.
- An empty object `{}` is no longer supported as a legit `ReactFragment`. No explanation should be offered there, in my opinion, because it was never accurate because erroneous types might be rendered and TS did not complain but failed at runtime. AFAIK, it was the hack for implicit children.

- The default type of 'this.context' is 'unknown,' not 'any,' which did not raise TS issues and was silent. The update was made in response to a request from the React community. In fact, this means that a suitable type must be introduced to a context wherever it is used.
- There are more, non-critical changes to the type definitions as described above. For example, type deprecations. You may readily identify them by using [codemod](https://github.com/eps1lon/types-react-codemod), a tool built by the author of these changes to types.

Aside from code updates, the React types must be modified to reflect at least the primary changes. Regrettably, for large JS projects, TS is more rigorous now, and does not allow things that were permitted in v17 or below. This should be considered when comparing the TS rules followed in an app. What if the majority of the components need to be refactored or updated?
<br>

## Strict mode behavior

The next, interesting thing is the strict mode that v18 offers. React, as docs states, in the further releases wants to ensure [the reusable state](https://reactjs.org/docs/strict-mode.html#ensuring-reusable-state). For that purpose, the newest StrictMode adds ‘strict mode effects’ that intentionally call side effects double times (mounting, unmounting, mounting) in dev mode. Some effects may not work as expected eg. subscriptions might be not properly destroyed in clean-up functions. This may make the strict mode off until callbacks are adjusted to the double invocation. If you enabled the mode in v18, the not properly cleaned subscriptions may entail additional refactorings.

## More pieces of advice

Here are some pointers for anyone thinking about bumping:

- Check with your team if eg. upcoming sprint is the right moment for the fully bumping. Considering the task as large and potentially failing without code refactorings, type updates, or paying off tech debt. Is it more important to use the features that v18 delivers or is it a nice-to-have? Which issues will React 18 address in your project?

- Evaluate the third-party libraries your app requires. Do they include v18 support in their bundle? Does the package.json - peerDependecies specify the latest React? How much work will it take to get them ready for version 18? Try updating them to the most recent versions and see whether everything works as it should. Even if the maintainers make them v18 compliant, you should upgrade them to the most recent version, before ensure that none of your features are broken. For a variety of reasons, legacy apps cannot bump certain of the packages, check if your software can manage that. Consider making contributions to other libraries that haven't yet supported v18 but are required by your software. Keep in mind you can always use `npm i --legacy-peer-deps` to skip `peerDependencies` check.

- Always stick to the official React docs and manuals regarding update commands you need to type in the terminal, skip tutorials that suggest crazy things.

- Analyze the specs you have. Verify that `act` is used around each of the mocked events. All functions responsible for (re)rendering components must be compared with act and v18 is more strict in that matter, see [act docs](https://reactjs.org/docs/test-utils.html#act) for further info. If your assertion fails without any reasons try to use `act` in your setup. Best if the [React test recipes](https://reactjs.org/docs/testing-recipes.html) are followed.
- Count the number of times your app uses deprecated API methods such as `ReactDOM.render` and `ReactDOM.unmountComponentAtNode`. Change them using the `createRoot` API. As the `Root` object must be accessible for the `unmount` method, refactoring is most likely required. I'd create a PR for each instance of the deprecation to guarantee that nothing breaks.

- Apply the `StrictMode` immediately after the bump. It may benefit in finding unsupported methods and other issues; see the [StrictMode documentation](https://reactjs.org/docs/strict-mode.html).

- Determine how many Class components you still keep. If not too much, I'd focus on converting the most critical ones into functions: `React.FC`. I've noticed that hook-based components aren't very verbose when it comes to warnings or errors.

- If you're using Typescript, you can utilize the [codemod](https://github.com/eps1lon/types-react-codemod) to highlight areas of your app that need to be tweaked. The most significant change is the removal of support for the implicit children prop. It must be stated explicitly.

- You might not want to use the automatic batching feature. It could potentially be a reason for the spec failing. You can disable it by using the 'flushSync' helper, but keep in mind that it degrades performance, see the [docs of flushSync](https://reactjs.org/docs/react-dom.html#flushsync).

- I would take care of any potential refactorings stated above before the React update, as they all make React 17 happy as well. You can interpret it as a bump preparations.

## Summary

What I like about software engineering is that it can be done incrementally. I recommend having a reliable plan in place to change the React version of an app smoothly and easily. The way that nobody noticed but your CI tools on performance score. In my case that was the most challenging React update I've ever made. If you are on the same task and need advice, you can try to reach out to me.

React 18 is great ;-)
