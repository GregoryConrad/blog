---
layout: post
title:  "What I Learned From Writing My First Dart Macros"
date:   2025-01-11
---

I recently decided to write some Widget-related macros for [ReArch],
and I wanted to share some learnings I had throughout that process here.
If you are looking to make a macro for whatever reason,
I'd strongly suggest reading through this quick blog post
for some ideas on how to structure your code and also some best-practices.
Even if you don't plan on building a macro, you might be able to learn something new!

Without further ado, here are the key takeaways.

## 1. Modeling Data Flow
When you think about it,
all a macro does is consume some input code and produce some output code
via a series of transformations.
So why not model and write our code exactly like that:
a pipeline of data transformations, or in more proper terminology, _data flow_.

For relatively simple macros, you can get away with abusing `late final` for this purpose.
How is `late final` at all related to data flow, you ask?
Here's how:

```dart
final class _MyMacroData {
  // We define our data inputs via the constructor
  // (in this case, just a FunctionDeclaration):
  _MyMacroData(this.function);
  final FunctionDeclaration function;

  // We can define our intermediaries and data outputs:
  late final dataType = function.returnType;
  late final functionName = function.identifier.name;
  late final positParams = function.positionalParameters;
  late final namedParams = function.namedParameters;

  // That's been all basic stuff so far, so let's ramp things up a bit:
  late final typeParams = function.typeParameters.map((t) => t.code);
  late final firstOptPositParam = // for diagnostic error reporting purposes
      positParams.where((p) => !p.isRequired).firstOrNull;

  // Here is where things get interesting; you can also execute any arbitrary code:
  late final widgetName = () {
    final upperCaseCutOff = functionName.lastIndexOf('_') + 2;
    return functionName.substring(0, upperCaseCutOff).toUpperCase() +
        functionName.substring(upperCaseCutOff);
  }();

  // Many other fields/functions omitted for brevity.
}
```

Do you see how expressive this is?
Each `late final` in our class is one stage of our "pipeline" that can
directly access the data of previous stages to compute its own data.
Modeling your macro code like this enables you to expose just a few final fields
that you may later reference when generating the output code.
The `late final` fields are only ever executed when used, and only once,
which perfectly fills our need of lazy/demand-driven evaluation here.

Even better is that this approach easily supports composition.
I.e., you can easily reuse the same backing `_MyMacroData` to house the same intermediary
data stages for entirely different macros.
Here's an example where we have two classes that produce code outputs for some macros:
```dart
final class _MyFunctionalWidgetOutput {
  _MyFunctionalWidgetOutput(this.data);
  final _MyMacroData data; // from previous snippet

  late final functionalWidgetCode = () {
    return DeclarationCode.fromParts([
      throw UnimplementedError('Your code gen using this.data goes here.'),
    ]);
  }();
}

final class _SomeOtherOutput {
  _SomeOtherOutput(this.data, this.otherData);
  final _MyMacroData data;         // from previous snippet
  final _OtherMacroData otherData; // some arbitrary macro data like the one above

  late final someOtherCode = () {
    return DeclarationCode.fromParts([
      throw UnimplementedError('Code using this.data and this.otherData...'),
    ]);
  }();
}

// Create an instance of _MyFunctionalWidgetOutput or _SomeOtherOutput
// to use in your corresponding macro classes.
```

The process detailed above is actually just a simplified subset of
[incremental computation (IC)](https://en.wikipedia.org/wiki/Incremental_computing);
however, since Dart macros don't have any form of persistence/shared state,
there is little need to recompute some values if the inputs change
(because the inputs won't change in the lifecycle of your macro).

Regardless, it is still worth mentioning [ReArch] (which is a fully-featured IC engine)
in case you are building a significantly more complex macro.
[ReArch]'s "capsules" effortlessly handle async macro APIs (like type resolution)
so that your resulting code is free of dealing with async/await.
Further, [ReArch] also allows you to move all of your "pipeline stages" into distinct capsules.
This in turn lets you split up and compose all of your data logic more effectively,
making [ReArch] a must for more complicated macros.
However, for simpler macros, you can get by with `late final`s just fine.


## 2. Hide Boilerplate, Not Complexity
_Do not use macros to hide away complexity_.
That is quite literally the worse use of macros; _macros should not be magic!_ 🪄✨

If you find yourself wanting to write a macro that
hides away some code/logic that the average developer wouldn't understand,
you should instead simplify your logic (and if you're a package author, your package's API).
This may sound harsh, but a macro in this context will just serve as an unmaintainable band-aid.
As you add new features, you're going to have a harder and harder time trying to update the macro
to accomodate said features while remaining bug-free.

So here's how macros should be used:
write macros that cut down on boilerplate that the average developer could easily write out by hand
given some (very basic) surrounding context.
A few examples of this would be for Flutter widgets
(so you don't need to hand-roll a class every time)
or for object serialization.

Here's an easy rule of thumb to recap:
if your coworker can't easily write out what your macro does by hand,
you shouldn't be writing that macro!
Instead, simplify your surrounding logic.


## 3. Declarative Source Code Generation
I quickly realized that you should almost never use `DeclarationCode.fromString`
unless it is a _trivial_ and _small_ code snippet that does not rely upon any external dependencies.
(And that should practically be never, because at that point, you should add that section of code
as a helper available elsewhere in your macro's package that can be referenced from your macro.)

That leaves us with `DeclarationCode.fromParts`.
This also sucks, because you're left with an unreadable mess that looks something like this:
```dart
final widgetCodeParts = [
  ...[
    'class ',
    data.widgetName,
    if (data.typeParams.isNotEmpty) ...data.typeParamsParamCode,
    ' extends ',
    if (data.isStateless) 'StatelessWidget' else 'RearchConsumer',
    ' {',
  ],
  ...['const ', data.widgetName, '(', ...constructorParams, ');'],
  ...data.classFields,
  ...[
    'Widget',
    ' build(',
    ...[
      ...['BuildContext', ' ', data.contextName, ', '],
      if (!data.isStateless) ...['WidgetHandle', ' ', data.useName],
    ],
    ') => ',
    ...[
      data.functionName,
      if (data.typeParams.isNotEmpty) ...data.typeParamsArgCode,
      '(',
      data.functionArgs,
      ');',
    ],
  ],
  '}',
];
```

I think we can all agree that the above looks horrendous[^1].
So what do we need? Well, we have two options.

[^1]: You can argue that you can split the list up into a few variables to help with readability, \
but it should be clear that we shouldn't be using a list for this.

The first is [quasiquoting](https://github.com/dart-lang/language/issues/1989).
Obviously, the Dart language does not (yet) have quasiquoting support,
and quasiquoting has been superseded by [tagged strings](https://github.com/dart-lang/language/issues/1988).
And unfortunately, this is still an open issue.

That leaves us with option #2.
The second option is a declarative dart source code generator.
The dart team provides [`code_builder`](https://github.com/dart-lang/code_builder),
but there's one catch: it's for `build_runner`, not macros.

So, there's no _proper_ solution to this problem at the moment.
In the meantime, I have developed my own "quasi-quasiquoting" approach,
which sits in at just a few dozen lines of code:
```dart
// I release this snippet to the public domain; feel free to use it in your own projects.
// If there was enough demand, I could release a package, but a copy+paste is easy enough.
'''
class ${widgetName} extends {{StatelessWidget}} {        
  const ${widgetName}({{constructorParams}} { super.key });

  {{classFields}}

  {{Widget}} build({{BuildContext}} context) =>
    ${functionName}(${functionArgs});
}
'''
  .substitute({
    'StatelessWidget': statelessWidget,
    'constructorParams': constructorParams,
    'classFields': classFields,
    'Widget': widget,
    'BuildContext': buildContext,
  })
  .flatten()
  .toList()
  .toDeclarationCode();

extension _Substitute on String {
  Iterable<Object> substitute(Map<String, Object> substitutes) sync* {
    final regex = RegExp('{{(.*?)}}');
    var lastIndex = 0;
    for (final match in regex.allMatches(this)) {
      yield substring(lastIndex, match.start);
      final key = match.group(1);
      final substitution = substitutes[key] ??
          (() => throw ArgumentError('"$key" was not provided'))();
      yield substitution;
      lastIndex = match.end;
    }
    yield substring(lastIndex);
  }
}

extension _Flatten on Iterable<Object> {
  Iterable<Object> flatten() sync* {
    for (final obj in this) {
      if (obj is Iterable<Object>) {
        yield* obj.flatten();
      } else {
        yield obj;
      }
    }
  }
}

extension _ToDeclarationCode on List<Object> {
  DeclarationCode toDeclarationCode() => DeclarationCode.fromParts(this);
}
```

While the above quasi-quasiquoting works, it's not my favorite.
I _think_ I would personally prefer a declarative source code generator
to ensure type safety at compile time,
despite it being slightly more verbose.

I would try my hand at implementing a package for declarative source code generation,
but I'm currently maintaining [Mimir], [ReArch], and [Unnested],
in addition to working a full-time job unrelated to Flutter,
so I didn't feel like taking on the responsibility of building yet another library.
However, if this is something that a lot of folks want,
I'd consider building out said library with a sponsorware model[^2],
but I have a feeling that most would prefer a quasiquoting based solution like the one above.
Alternatively, the door is always open for someone else from the community to build a solution[^3].

[^2]: Please reach out to me at contact at gsconrad dot com if you are interested in sponsoring this!

[^3]: I think Remi said a long time ago that he was working on a quasiquoting approach; follow him for updates!


## Conclusion
I discussed a few key learnings to apply when writing your own macros,
including how to structure your macro code as data flow,
hiding boilerplate instead of complexity,
and source code generation.

I'm sure there are many other best-practices that I am forgetting to list here,
but at least the three above should be a good start.


<div id="sfcztusseehdd766ugkqxxyyusw9gpxa8e1"></div>
<script type="text/javascript" src="https://counter6.optistats.ovh/private/counter.js?c=ztusseehdd766ugkqxxyyusw9gpxa8e1&down=async" async></script>
<br>
<noscript>
  <a href="https://www.freecounterstat.com" title="free hit counter code">
    <img src="https://counter6.optistats.ovh/private/freecounterstat.php?c=ztusseehdd766ugkqxxyyusw9gpxa8e1" border="0" title="free hit counter code" alt="free hit counter code">
  </a>
</noscript>


<!-- Footnotes -->
<br>

---
<br>
<!-- footnotes will be rendered here -->


[ReArch]: https://github.com/GregoryConrad/rearch-dart/
[Mimir]: https://github.com/GregoryConrad/mimir
[Unnested]: https://github.com/GregoryConrad/unnested/
