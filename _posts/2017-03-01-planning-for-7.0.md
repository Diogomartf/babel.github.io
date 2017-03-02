---
layout: post
title:  "Planning for 7.0"
author: Henry Zhu
date:   2017-03-01 12:00:00
categories: announcements
share_text: "Planning for 7.0"
---

If you didn't know already, we're planning on releasing a 7.0 version soon! I'd like to go over some notable changes and a few things we are still working out!

## 😎 Awesome Changes

### [#4315](https://github.com/babel/babel/issues/4315) Drop support for unmaintained Node versions: 0.10, 0.12, 5

Progress in OSS projects often comes at the cost of upgrading for its users. Because of this, we've always been hesitant in making the choice to introduce a major version bump/breaking changes. But by dropping unsupported versions of Node, we can not only make a number of improvements to the codebase, but also upgrade dependencies and tools (ESLint, Yarn, Jest, Lerna, etc).

## Meta

### [#5218](https://github.com/babel/babel/pull/5218) Remove `babel-runtime` from Babel dependencies

Babel itself doesn't have that many external dependencies, but in 6.x [each package has a dependency on `babel-runtime`](https://github.com/babel/babel/blob/958f72ddc28e2f5d02adf44eadd2b1265dd0fa4d/packages/babel-plugin-transform-es2015-arrow-functions/package.json#L12) so that built-ins like Symbol, Map, Set, and others are available without needing a polyfill. By changing the minimum supported version of Node to v4 (where those built-ins are supported natively), we can drop the dependency entirely.

> This is an issue on npm 2 (we didn't recommended its use with Babel 6) and older yarn, but not npm 3 due to its deduping behavior.

With [Create React App](https://github.com/facebookincubator/create-react-app) the size of the node_modules folder changed drastically when babel-runtime was hoisted.

- `node_modules` for npm 3: ~120MB
- `node_modules` for Yarn (<`0.21.0`): ~518MB
- `node_modules` for Yarn (<`0.21.0`) with hoisted `babel-runtime`: ~157MB
- `node_modules` for Yarn + [PR #2676](https://github.com/yarnpkg/yarn/pull/2676): ~149MB ([tweet](https://twitter.com/bestander_nz/status/833696202436784128))

So although this issue will be fixed "upstream" by using npm 3/latest Yarn, we can do our part by simply dropping our own dependency on `babel-runtime`.

### [#5224](https://github.com/babel/babel/pull/5224) Independent Publishing of Packages

> I mention this in [The State of Babel](http://babeljs.io/blog/2016/12/07/the-state-of-babel#the-future-versioning).
> https://github.com/babel/babylon/issues/275

You might remember that after Babel 6, Babel became a set of npm packages with its own ecosystem of custom presets and plugins.

However since then, we've always used a "fixed/synchronized" versioning system (so that no package is on v7.0 or above). When we do a new release such as `v6.23.0` only packages that have updated code in the source are published with the new version while the rest of the packages are left as is. This mostly works in practice because we use `^` in our packages.

Unfortunately this kind of system requires that a major version be released for all packages once a single package needs it. This either means we make a lot small breaking changes (unnecessary) or we batch lots of breaking changes into a single release. Instead, we want to differentiate between the experimental packages (Stage 0, etc) and everything else (es2015).

This means that we intend to make major version bumps to any experimental proposal plugins when the spec changes rather than waiting to update all of Babel. So anything that is < Stage 4 would be open to breaking changes in the form of major version bumps and same with the Stage presets themselves if we don't drop them entirely.

For example:

Say you are using preset-env (which keeps up to date and currently includes everything in es2015, es2016, es2017) + an experimental plugin. You also decide to use object-rest-spread because it's cool.

```json
{
  "presets": ["env"],
  "plugins": ["transform-object-rest-spread"]
}
```

If the spec to an experimental proposal changes, we should be free to make a breaking change and make a major version bump for that plugin only. Because it only affects that plugin, it doesn't affect anything else and people are free to update when possible. We just want to make sure that users update to the latest version of any experimental proposal when possible and provide the tools to do so automatically if that is reasonable as well.

## 🚀 New Feature

### [#3683](https://github.com/babel/babel/pull/3683) Add `transform-unicode-property-regex` to the Stage 2 preset

[transform-unicode-property-regex](https://github.com/mathiasbynens/babel-plugin-transform-unicode-property-regex) is maintained by [@mathiasbynens](https://github.com/mathiasbynens)

> Compile Unicode property escapes (\p{…} and \P{…}) in Unicode regular expressions to ES5 or ES6 that works in today’s environments.

Input

```js
var regex = /\p{ASCII_Hex_Digit}/u;
```

Output

```js
var regex = /[0-9A-Fa-f]/;
```

## 🤔 Questions

### Using npm Scoped Packages

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Thoughts on <a href="https://twitter.com/babeljs">@babeljs</a> using npm scoped packages for 7.0?</p>&mdash; Henry Zhu (@left_pad) <a href="https://twitter.com/left_pad/status/821551189166878722">January 18, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Seems like most who understood what scoped packages are were in favor?

Pros

- Don't have to worry about getting a certain package name (the reason why this was brought up in the first place).

> Many package names have been taken (preset-es2016, preset-es2017, 2020, 2040, etc). Can always ask to transfer but not always easy to do and can lead to users believing certain packages are official due to the naming.

Cons

- We need to migrate to new syntax
- Still unsupported on certain non-npm tools (lock-in)
- No download counts unless we alias back to old names

Sounds like we may want to defer, and in the very least it's not a breaking change given it's a name change.

### Dropping the "env" option in `.babelrc`

> [babel/babel#5276](https://github.com/babel/babel/issues/5276)

The "env" configuration option (not to be confused with babel-preset-env) has been a source of confusion for our users as seen by the numerous issues reported.

The [current behavior](http://babeljs.io/docs/usage/babelrc/#env-option) is to merge the config values into the top level values which isn't always intuitive such that developers end up putting nothing in the top level and just duplicating all the presets/plugins under separate envs.

To eliminate the confusion (and help our power users), we're considering dropping the env config option all together and recommending users use the proposed JS config format (see below).

### `.babelrc.js`

> [babel/babel#4630](https://github.com/babel/babel/issues/4630)

ESlint allows for an `.eslintrc.js` file and webpack has `webpack.config.js`.

Developers can easily implement environment config changes in JavaScript itself

```js
var env = process.env.BABEL_ENV || process.env.NODE_ENV;
module.exports = {
  plugins: [
    env === 'production' && "transform-react-constant-elements"
  ].filter(Boolean)
};
```

Or how [create-react-app](https://github.com/facebookincubator/create-react-app/blob/65e63403952f4f3c7e872f707fb3736e339254d9/packages/babel-preset-react-app/index.js#L42-L51) easily worked around a bug with using "env" in a external preset: 

```js
var plugins = [];
if (env === 'production') {
  plugins.push.apply(plugins, ["transform-react-constant-elements"]);
}
module.exports = { plugins };
```

Either way, it seems both simple and straightforward to write this kind of logic (and we could always provide some helper functions for this).

### `external-helpers`, `transform-runtime`, `babel-polyfill`

> "regeneratorRuntime is not defined" - reported all the time.

Basically we need a better solution around how to deal with built-ins/polyfills.

- Developers don't know what regenerator-runtime is, they just want to use generators/async functions.
- Many developers are confused as to why a runtime is needed at all or why Babel doesn't compile `Promise`, `Object.assign`, or some other built-in.
- Developers are confused with the difference between `transform-runtime` the Babel plugin and the runtime itself, `babel-runtime`.
- Complaints about generated code size since `babel-polyfill` includes all polyfills (although now we have [`useBuiltIns`](https://github.com/babel/babel-preset-env#usebuiltins)) and no one knowing about `external-helpers`

Can we combine/replace these packages and have an easier, default experience?

### Removal of Stage X presets

> [babel/babel#4914](https://github.com/babel/babel/issues/4914)
> [babel/babel#5128](https://github.com/babel/babel/issues/5128)

Many in the community (and TC39) have expressed concerns over the Stage X presets. I believe I just added them to have an easy migration path from Babel 5 to Babel 6 (used to be a "stage" option).

While we want to have an easy to use tool, it turns out that many companies/developers use these "not yet JavaScript" presets all the time, and in production. "Stage 0" doesn't really set the same tone as `babel-preset-dont-use-this-stage-0`.

> Ariya just made an [awesome poll](https://twitter.com/AriyaHidayat/status/833797322786025472) that explains what I'm talking about

Developers don't actually know what features are in what version of JavaScript (and they shouldn't have to know). However it is a problem when we all start thinking that "features" that are actually still proposals are in the spec already.

Many open source projects (including Babel still 😝), tutorials, conference talks, etc all use `stage-0`. React promotes the use of JSX, class properties (currently Stage 2), object rest/spread (now Stage 3) and we all believe that it's just JavaScript because Babel compiled it for them. So maybe removing this abstraction would help people understand more about what is going on and the tradeoffs one is making when choosing to use Stage X plugins.

It also seems much easier to maintain your own preset than to have to update the Stage preset.

> I often see people go "I want object rest, and that's stage 2, so I enabled stage 2". They now have a load of other experimental features enabled they might not know about and probably don't need.
> Also, as stages change over time then people who aren't using shrinkwrap or yarn will get new features appearing, possibly without their knowledge. If a feature is canned they might even get one vanishing. @glenjamin

### Removal of ES20xx presets

If we're going to think about removing the Stage X presets, why not the ES20XX presets (currently ES2015, ES2016, ES2017)?

> It's annoying making a yearly preset (extra package/dependency, issues with npm package squatting unless we do scoped packages)

Developers shouldn't even need to make the decision of what yearly preset to use? If we drop/deprecate these presets then everyone can use [babel-preset-env](https://github.com/babel/babel-preset-env) instead which will already update as the spec changes.
