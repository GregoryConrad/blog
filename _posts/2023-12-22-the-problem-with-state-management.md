---
layout: post
title:  "The Problem With State Management, and Why It Shouldn’t Be a Problem"
date:   2023-12-22
---

> This few-minute and bite-sized read is condensed from my recently-completed
[Master's Thesis](/thesis.pdf).
Go take a look at it after you're done here if you want to learn more!

In case you haven’t noticed, there are more state management frameworks than atoms in the universe,
and there appears to be a new addition every day.
But state management should be easy, right?
Just update some state when a user interacts with a UI, and then update the UI accordingly.
Here’s the catch: the second you add new feature requirements into an application,
it isn't such a clear cut problem and things can start to get tricky.


## Background
To handle these trickier scenarios,
many solutions available today do just one or two things *really well*.
For example, BLoC is great at separating complex state logic from UI code via the reducer pattern.
Riverpod, Signals, and Recoil are fantastic in regards to data modeling and data flow,
at least once you wrap your head around the whole reactive paradigm.
Although not ideal for state management (which would be a whole blog post on its own),
streams are still great at representing transformations on individualized and discrete data.
If you are familiar with it,
`ValueNotifier` is perfect when you have a single mutable variable
whose mutations need to be reflected in your UI.
GetX is, well, an exception to this pattern, as it does so many things and none of them well.


## The Problem
So why do we have all of these libraries?
Short answer: it’s because they all fill a different need,
and only specialize in that one area or two.
While this makes them great and highly specialized
for the specific problem they were designed to solve,
this comes with the drawback that they are pretty unergonomic for everything else.

As an example, BLoC introduced cubits because normal blocs
are almost always too verbose for simple state/UI relationships.
While Riverpod readily supports data compostion and caching,
[data persistence](https://github.com/rrousselGit/riverpod/issues/1032)
and [data mutations](https://github.com/rrousselGit/riverpod/issues/1660)
have been open issues for several years now,
and both are common requirements in offline-first and networked applications.


## The Solution
> This section will briefly go into a bit of some theory here, so bear with me.

When you are building an application, you should normally start from a singular
target, goal, or idea in mind.
Then, you break down that one overarching concept into a set of feature requirements.
Breaking those feature requirements down further,
you will arrive at smaller and more individualized feature requirements.
This process repeats until you eventually get down to the code itself,
at which point you start to implement individual features.

Here's the key observation: this entire process actually forms a tree!
As such, we can say that _features assemble into trees_
(and this concept is really important for the rest of this post).
Perhaps even more important is that many of these features have commonalities
that are shared across subtrees,
which opens the door for high code reuse.

> As an aside, I often find that when I program,
the closer I keep some code to its underlying theory, the cleaner the resulting code is;
inheritance (when used over composition) and unwarranted abstractions often result in a mess.

### Features as Trees
Getting back to reality here, what do "features as trees" look like,
or even mean for that matter?

Let's take an application that needs to perform a simple network request (say an HTTP GET).
This alone sounds like a pretty simple requirement!
Now how about we need to add in a cache for this network request.
We may need to put some thought here into orchestrating the requests and caching,
but this should be doable.
But wait! Our customer now wants to be able to perform POST requests too,
so we'll need to also add invalidation.
How do we manage all of these competing requirements?
And as a thought experiment--think of all the moving pieces you may need
in your own state management solution of choice to implement this.

In our particular example, the overarching feature is having some best-effort data
always available to an application user,
which itself is a _composition_ over the underlying (and independent)
network request, caching, and invalidation features.

### Composition
For those unfamiliar with Component-Based Software Engineering,
the general idea is that a developer can produce independent _components_
that can later be _assembled_ into a full application.
Composition in this manner results in loosely coupled applications
and [significantly increases code quality](https://dl.acm.org/doi/10.5555/379381).

While global/application-level state and logic is assembled into a dependency graph[^1],
individual application features form a tree.
As such, in our networking example above,
we can assemble the individual features (network request, cache, and invalidation)
together into a tree.

[^1]: Even if you are not using something like Riverpod/Signals/Recoil, dependency inversion techniques and libraries like Guice actually do result in dependency graphs.

### In Code
Let's again consider our network + cache + invalidation example.
To support the overarching feature (displaying highly-available best-effort data to our users),
we ideally want to be able to assemble the individual features together
with something like the following pseudo-Dart:
```dart
({MyData Function() getState, void Function() invalidateState}) myFancyFeature() {
  final networkClient = networkFeature(yourApiUrl);
  final (getNetworkState, invalidateNetworkState) = invalidationFeature(networkClient);
  final (persistedState, setPersistedState) = cacheFeature(
    read: () => readFromDatabaseOfYourChoice(),
    write: (data) => writeToDatabaseOfYourChoice(data),
  );

  return ({
    getState: () => persistedState ?? getNetworkState().then(setPersistedState),
    invalidateState: () {
      invalidateNetworkState();
      setPersistedState(null);
    },
  });
}
```

See how easy that problem gets once we introduce composition?
_Magic_. 

#### “That Pseudocode Is Great, but How Do We Actually Do That?”
I am only aware of two approaches that will let you write code with feature-focused composition[^2].

[^2]: Some readers may suggest the use of mixins; I'd advise against this. In addition to relying on inheritance over true composition, mixins only support a _single_ usage per `with MyMixin`, which breaks the whole "feature tree" idea (say your component now needs to persist two unrelated values--you're out of luck since you can't do `with MyMixin, MyMixin`).

The first one you may have heard of: Hooks!
Aside from the commonly-discussed benefits like automatically managed
text editing and animation controllers,
hooks shine due to their easy composability.

There's just one big issue with hooks:
they only work within widgets for [ephemeral state](https://docs.flutter.dev/data-and-backend/state-mgmt/ephemeral-vs-app).

And that is where we come to our next approach: [ReArch](https://github.com/gregoryconrad/rearch-dart/).

> Disclaimer: I am the author of ReArch.
While I would love ReArch to get some exposure as a result of this post,
a strong motivation behind this written piece is my interest in the "meta" of software engineering.

ReArch enables feature-focused composition for both ephemeral and app state via _side effects_,
which are named as such because they provide applications with mutability
and a mechanism to interact with the outside world.
With ReArch's side effects model, applications are also given high code reuse
and knowledge transference between all application layers _for free_.

Finally, here is one way we can implement our above example via feature composition
for some application-level state in ReArch:

```dart
// This function is a "capsule", which represents a piece of app state.
// See more here: https://rearch.gsconrad.com/core/capsules
({
  AsyncValue<int> Function() getData,
  void Function() invalidateData,
}) myCachedNetworkedDataCapsule(CapsuleHandle use) {
  final (persistedState, persistData) = use.persist(
    read: readFromDatabase,
    write: writeToDatabase,
  );
  final (getNetworkState, invalidateNetworkRequest) = use.invalidatableFuture(
    () async {
      final data = await fetchFromNetwork();
      persistData(data);
      return data;
    },
  );
  final runTransaction = use.transactionRunner(); // more on this in a second

  return (
    getData: () {
      return switch (persistedState) {
        // If we have some form of useful data, let's use it
        // instead of going to the network.
        AsyncData<int?>(:final data) when data != null => AsyncData(data),

        // If we still don't know what our new persisted data status is,
        // let's wait a bit before falling back to the network.
        AsyncLoading<int?>(previousData: Some(:final value))
            when value != null =>
          AsyncLoading(Some(value)),
        AsyncLoading<int?>() => const AsyncLoading(None()),

        // If persisting error-ed and/or we don't have any useful data,
        // we need to resort to the network.
        _ => getNetworkState(),
      };
    },

    // "runTransaction" allows you to update multiple side effects at once,
    // resulting in only a singular build.
    invalidateData: () => runTransaction(() {
          invalidateNetworkRequest();
          persistData(null);
        }),
  );
}
```

ReArch is also the subject of my Master's Thesis,
and I have spent the better part of a year honing and improving it[^3].
If you are familiar with Hooks/Riverpod/Signals/Recoil,
ReArch should feel exceedingly natural to you.
ReArch also supports the reducer pattern that is employed by BLoC
for particularly complex state relations.
Even then, if you would prefer a more traditional/OOP-oriented approach to application development,
you have my full endorsement to use your own approach to feature composition
coupled with some form of dependency inversion!
You may just eventually find that easy composition and reactivity is a nice touch,
and ReArch will be waiting for you then. 😉

[^3]: You may have seen the [original 1.0.0 announcement](https://www.reddit.com/r/FlutterDev/comments/16n446w/announcing_rearch_a_reimagined_way_to/) a few months ago.

## Conclusion
The problem with state management, as I have outlined it above,
is the lack of feature-focused composition being readily available,
which causes every library to reinvent the wheel every time users request new and common features.
Thus, the solution is simple: develop a minimal framework for composition
that users can extend themselves to develop their own features with ease.
I present ReArch for this purpose, but hooks are also sufficient
(at least just for ephemeral state).

<!-- Footnotes -->
<br>

---
<br>
<!-- footnotes will be rendered here -->
