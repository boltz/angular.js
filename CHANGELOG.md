<a name="1.0.0rc1"></a>
# 1.0.0rc1 moiré-vision (2012-03-13)

## $compile rewrite

The compiler was completely rewritten from scratch using ideas from this
[design document](https://docs.google.com/document/d/1PNh4lxlYpSRK2RhEwD4paJLMwdcnddcYJn3rsDsdayc/edit).
Please check out the [$compile] and
[$compileProvider.directive](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive)
docs. The biggest improvements and changes are listed below.

- the compiler now transparently supports several directive syntaxes. For example while before there
  was just one way to use `ng:include` directive: `<ng:include src="someSrc"></ng:include>`. The new
  compiler treats all of the following as equivalent:

  - `<ng:include src="someSrc"></ng:include>`
  - `<ng-include src="someSrc"></ng-include>`
  - `<x-ng-include src="someSrc"></x-ng-include>`
  - `<div ng:include src="someSrc"></div>`
  - `<div ng-include src="someSrc"></div>`
  - `<div data-ng-include src="someSrc"></div>`
  - `<div ng:include="someSrc"></div>`
  - `<div ng-include="someSrc"></div>`
  - `<div data-ng-include="someSrc"></div>`
  - `<div class="ng-include: someSrc"></div>`

  This will give template creators great flexibility to consider the tradeoffs between html code
  validity and code conciseness and pick the syntax that works the best for them.

- we are switching all of our code/docs/examples to use `ng-foo` directive name style instead of
  `ng:foo`. The new compiler doesn't distinguish between these and other name styles (all of them
  are [equally supported](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive)),
  the main difference is that `ng-foo` is easier to select with css selectors. Check out the
  [Internet Explorer Compatibility](http://docs-next.angularjs.org/guide/ie.ngdoc)
  doc to learn about various IE-related requirements for different directive naming styles.

- `angular.directive`, `angular.widget`, `angular.attrWidget` were merged into a single concept: a
  `directive` which is registered via
  [myModule.directive](http://docs-next.angularjs.org/api/angular.Module#directive) or
  [$compileProvider.directive](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive).
  You can control execution priority of multiple directives on the same element (previously the main
  difference between a attribute widget and a directive) via a directive priority setting.

- previously the linking functions of directives were called top to bottom following the DOM tree,
  to enable a linking fn to work child DOM nodes that were already processed by child linking fns
  the order was changed as follows: compile functions run top to bottom following the DOM tree, but
  linking functions run bottom-up following the DOM tree. In some rare cases it is desirable for
  linking fns to be called top to bottom and for these it is possible to register "prelinking"
  functions (check out
  [the docs](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive)
  for the return value of the compile function).

- `angular.markup` and `angular.attrMarkup` were replaced with interpolation via `$interpolate`
  service.

  - In the past `{{foo}}` markup was getting translated to `<span ng-bind="foo"></span>` during the
    early stage of template compilation. Addition of this extra node was in some cases undesirable
    and caused problems. The new compiler with the help of the $interpolate service removes the need
    for these artificial nodes.

  - As a side-effect of not using artificial nodes available for all bindings, the `html` filter
    which used to innerHTML (sanitized) html into the artificial node was converted into a directive.
    So instead of `{{ someRawHtml | html }}` use `<div ng-bind-html="someRawHtml"></div>` and
    instead of `{{ someRawHtml | html:"unsafe" }}` use `<div ng-bind-html-unsafe="someRawHtml"></div>`.
    Please check out the
    [ng-bind-html](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-bind-html)
    and
    [ng-bind-html-unsafe](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-bind-html-unsafe)
    directive docs.

  - Custom markup has been used by developers only to switch from `{{ }}` markup to `(( ))` or
    something similar in order to avoid conflicts with server-side templating libraries. We made it
    easier to do this kind of customization by making the start and end symbol of the interpolation
    configurable via [$interpolateProvider](http://docs-next.angularjs.org/api/angular.module.ng.$interpolateProvider).

- [template loader](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.script)
  loads template fragments from script elements and populates the $templateCache with them. Templates
  loaded in this way can be then used with `ng-include`, `ng-view` as well as directive templates
  (see the `templateUrl` property of the
  [directive config object](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive)).


## Forms / input controls / two-way data binding

The implementation of forms and input bindings was modified to address issues around composability,
ease of adding custom validation and formatting. Please check out the
[forms dev guide article](http://docs-next.angularjs.org/guide/dev_guide.forms) to learn about forms,
form control bindings and input validation. The biggest changes are listed below.

- any directive can add formatter/parser (validators, convertors) to an input type. This allows
  better composability of input types with custom validators and formatters. So instead of creating
  new custom input type for everything, it's now possible to take existing input type and add an
  additional formatter and/or validator to it via a custom directive.

- inputs propagates changes only on the blur event by default (use new `ng-model-instant` directive
  if you want to propagate changes on each keystroke).

- no more custom input types, use directives to customize existing types.

- removed $formFactory.

- removed parallel scope hierarchy (forms, widgets).

- removed `list` input type (use `ng-list` directive instead).

- removed integer input type.


## Controller-scope separation

Controllers are now standalone objects, created using the "new" operator, and not mixed with scope
object anymore. This addresses many issues including:
[#321](https://github.com/angular/angular.js/issues/321) and
[#425](https://github.com/angular/angular.js/issues/425).

The [design doc](https://docs.google.com/document/pub?id=1SsgVj17ec6tnZEX3ugsvg0rVVR11wTso5Md-RdEmC0k)
explains the reasoning for this major change and how it solves many issues.

### Before:

<pre>
function MyCtrl() {
  var self = this;

  this.model = 'some model of any type';

  this.fnUsedFromTemplate = function() {
    someApiThatTakesCallback(function callbackFn() {
      self.model = 'updatedModel';
    });
  };
}
</pre>

### After:

<pre>
function MyCtrl($scope) {
  $scope.model = 'some model of any type';

  $scope.fnUsedFromTemplate = function() {
    someApiThatTakesCallback(function() {
      $scope.model = 'updatedModel';
    });
  }
}
</pre>

Temporary backwards compatibility: Load the following module in your app to recreate the previous
behavior and migrate your controllers one at a time: <https://gist.github.com/1649788>


## $route service changes

- As advertised in the past we moved the $route configuration from the run phase of the application
  to the config phase. This means that instead of defining routes via `$route.when`/`$route.otherwise`
  you should use `$routeProvider.when`/`$routeProvider.otherwise` instead.

- route scope is now being created by the `ng-view` rather than by `$route`, this resolved many
  issues we've previously faced. For more info, read the
  [commit message](https://github.com/angular/angular.js/commit/60743fc52aea9eabee58258a31f4ba465013cb4e).

- removed `$route.parent()` - it's unnecessary because the scope is properly created in the scope
  hierarchy by `ng-view`.

- new `$viewContentLoaded` and `$includeContentLoaded` events which directives can use to be
  notified when a template content is (re)loaded.

- `ng-view` now has `onload` attribute which behaves similarly to the one on `ng-include`.


## Directives

- `ng-model` binding on select[multiple] element should support binding to an array
  ([commit](https://github.com/angular/angular.js/commit/85b2084f578652cc0dcba46c689683fc550554fe))
- event object is now accessible as `$event` in `ng-click` and other directives
  ([commit](https://github.com/angular/angular.js/commit/1752c8c44a7058e974ef208e583683eac8817789),
   issue [#259](https://github.com/angular/angular.js/issues/259)
- `ng-class` directive now support map of classnames and conditions
  e.g. `<div ng-class="{'hide': !visible, 'warning': isAlert()}"...` (contributed by Kai Groner)
  ([commit](https://github.com/angular/angular.js/commit/56bcc04c54ed24c19204f68de52b8c30c00e08f0))


## Scope changes

- `scope.$emit`/`$broadcast` return the event object, add cancelled property
  ([commit](https://github.com/angular/angular.js/commit/6e635012fb30905e5fe659a024864e275f1c14b5))

- `scope.$new()` takes one argument - a boolean indicating if the newly-created child scope should be
  isolated (not prototypically inheriting from the current scope). Previously the first argument was
  reference to the controller constructor, but because of the scope/controller separation the
  controllers should be instantiated via the `$controller` service.
  ([commit](https://github.com/angular/angular.js/commit/78656fe0dfc99c341ce02d71e7006e9c05b1fe3f))

- fn signature change for change listener functions registered via `scope.$watch` - this means that
  the scope object can be listed in the arguments list only if its needed and skipped otherwise.
  ([commit](https://github.com/angular/angular.js/commit/0196411dbe179afe24f4faa6d6503ff3f69472da))

  - before: `scope.$watch('someModel', function(scope, newVal, oldVal) {})`
  - after: `scope.$watch('someModel', function(newVal, oldVal, scope) {})`

- `scope.$watch` now compares object by reference and only if extra boolean flag is passed
  comparison by equality is used. This was done to avoid unintended performance issues.
  ([commit](https://github.com/angular/angular.js/commit/d6e3e1baabc3acc930e4fda387b62cbd03e64577))

  - before: `scope.$watch('expression', function(scope, newVal, oldVal) {})`
  - after: `scope.$watch('expression', function(newVal, oldVal, scope) {}, true)`

- `scope.$destroy` doesn't cause the `$destroy` event to be emitted any more - this event was
   primarily used by the old forms implementation and is not needed any more. We are considering
   broadcasting this event in the future, which could then be used by directives and child scopes to
   be notified of their scope destruction.


## New directives:

- [ng-mouseleave](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-mouseleave)
- [ng-mousemove](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-mousemove)
- [ng-mouseover](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-mouseover)
- [ng-mouseup](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-mouseup)
- [ng-mousedown](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-mousedown)
- [ng-dblclick](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-dblclick)
- [ng-model-instant](http://docs-next.angularjs.org/api/angular.module.ng.$compileProvider.directive.ng-model-instant)


## $injector / modules

- `$injector.instantiate` should return the object returned from constructor if one was returned
  ([commit](https://github.com/angular/angular.js/commit/776739299b698a965ef818eeda75d4eddd10c491))
- `$injector.instantiate` should support array annotations for Type argument (e.g. instantiate(['dep1', 'dep2', Type]))
  ([commit](https://github.com/angular/angular.js/commit/eb92735c9ea3e5ddc747b66d8e895b6187a5f9e0))
- quickly fail if circular dependencies are detected during instantiation
  ([commit](https://github.com/angular/angular.js/commit/fbcb7fdd141c277d326dc3ed34545210c4d5628f))
- added [$provide.constant](http://docs-next.angularjs.org/api/angular.module.AUTO.$provide#constant)
  to enable registration of constants that are available in both the config and run phase
  ([commit](https://github.com/angular/angular.js/commit/80edcadb1dd418dcf5adf85704c6693940c8bb28))
- `$provide.service` was renamed to $provide.provider
  ([commit](https://github.com/angular/angular.js/commit/00d4427388eeec81d434f9ee96bb7ccc70190923))
- `$provide.service` takes a constructor fn and creates a service instance by using $injector.instantiate


## New services:

- [$sanitize](http://docs-next.angularjs.org/api/angular.module.ng.$sanitize)
- [$interpolate](http://docs-next.angularjs.org/api/angular.module.ng.$interpolate)


## jqLite (angular.element)

- added `contents()` ([commit](https://github.com/angular/angular.js/commit/97dae0d0a0226ee527771578bfad1342d51bf4dd))
- added `wrap()` ([commit](https://github.com/angular/angular.js/commit/4a051efb89cf33e30d56f1227d1f6084ead4cd42))
- fix memory leaking in IE8 (remove monkey patched methods on Event)
  ([commit](https://github.com/angular/angular.js/commit/3173d8603db4ae1c2373e13a7a490988126bb1e7),
   [commit](https://github.com/angular/angular.js/commit/230f29d0a78a04a6963514da8b1e34cc03e553d0))


## Docs

- new [Modules dev guide article](http://docs-next.angularjs.org/guide/module.ngdoc)


## Small bug fixes

- fix incorrect comparison of dates by angular.equals
  ([commit](https://github.com/angular/angular.js/commit/ffa84418862a9f768ce5b9b681916438f14a0d79))
- `scope.$watch` support watching functions
  ([commit](https://github.com/angular/angular.js/commit/7da2bdb82a72dffc8c72c1becf6f62aae52d32ce),
   [commit](https://github.com/angular/angular.js/commit/39b3297fc34b6b15bb3487f619ad1e93c4480741))
- `$http` should not json-serialize File objects, instead just send them raw
  ([commit](https://github.com/angular/angular.js/commit/5b0d0683584e304db30462f3448d9f090120c444))
- `$compile` should ignore content of style and script elements
  ([commit](https://github.com/angular/angular.js/commit/4c1c50fd9bfafaa89cdc66dfde818a3f8f4b0c6b),
   [commit](https://github.com/angular/angular.js/commit/d656d11489a0dbce0f549b20006052b215c4b500))
- `TzDate#getDay()` should take into account the timezone offset (contributed by Stephane Bisson)
  ([commit](https://github.com/angular/angular.js/commit/e86bafecd212789cde61050073a69c1e49ffd011))


## Small features

- `$parse` service now supports local vars in expressions
  ([commit](https://github.com/angular/angular.js/commit/761b2ed85ad9685c35f85513e17363abf17ce6b3))



<a name="0.10.6"></a>
# 0.10.6 bubblewrap-cape (2012-01-17) #

## Features:

- [Dependency injection subsystem][guide2.di] rewrite. This is a huge change to the Angular core
  that was necessary for many reasons. Please read the full
  [design doc](https://docs.google.com/document/d/1hJnIqWhSt7wCacmWBB01Bmc6faZ8XdXJAEeiJwjZmqs/edit?hl=en_US)
  to understand the changes and reasoning behind them.
- Added [angular.bootstrap] for manual bootstrapping of the app. Also see
  [Initializing Angular App][bootstrapping] doc.
- Helper functions [inject] and [module] that make testing with DI and jasmine a lot easier.
- [jqLite][jqLite2] and jQuery were extended with helper method `injector()` that simplifies the
  access to the application injector during debugging.
- Rewrite of $xhr service and its dependencies, which was replaced with [$http] service.
  The $browser.xhr and its mock were replaced by [$httpBackend] and its
  [unit testing][unit-testing $httpBackend] and [end-to-end testing][e2e-testing $httpBackend]
  mocks. The $resource service api and functionality was preserved, with the exception of caching,
  which is not happening automatically as it used it in the past (verifyCache has no effect).
- [$q] - Q-like deferred/promise implementation
  ([commit](https://github.com/angular/angular.js/commit/1cdfa3b9601c199ec0b45096b38e26350eca744f))
- Transparent data-binding to promises in templates. [Example](http://jsfiddle.net/IgorMinar/aNSWu/)
  ([commit](https://github.com/angular/angular.js/commit/78b6e8a446c0e38075c14b724f3cdf345c01fa06))
- New [$anchorScroll] service that watches url hash and navigates to the html anchor even if the
  content was loaded via [ng:view]  (for [ng:include] you have to opt into this behavior using
  autoscroll attribute)
- New LRU cache factory - [$cacheFactory] service
- jQuery 1.7 compatibility


## Bug Fixes:

- Directive names are now case insensitive
  ([commit](https://github.com/angular/angular.js/commit/1e00db8daa5c09e7f8f9134f5c94b9a18c7dc425))
- $location#url setter fix (Issue [#648](https://github.com/angular/angular.js/issues/648))
- [ng:include] - prevent race conditions by ignoring stale http callbacks
  ([commit](https://github.com/angular/angular.js/commit/1d14760c6d3eefb676f5670bc323b2a7cadcdbfa))
- [ng:repeat] - support repeating over array with null
  ([commit](https://github.com/angular/angular.js/commit/cd9a7b9608707c34bec2316ee8c789a617d22a7b))
- [angular.copy] - throw Error if source and destination are identical
  ([commit](https://github.com/angular/angular.js/commit/08029c7b72a857ffe52f302ed79ae12db9efcc08))
- Forms should not prevent POST submission if the action attribute is present
  ([commit](https://github.com/angular/angular.js/commit/c9f2b1eec5e8a9eaf10faae8a8accf0b771096e0))


## Breaking Changes:

- App bootstrapping works differently (see [angular.bootstrap] and [ng:app] and [bootstrapping])
- scope.$service is no more (because injector creates scope and not the other way around),
  if you really can't get services injected and need to fetch them manually then, get hold of
  [$injector] service and call $injector.get('serviceId')
- angular.service style service registration was replaced with module system, please see
  [angular.module] api and [DI documentation][guide2.di] for more info.
- the $xhr service was replaced with [$http] with promise based apis.
- [unit-testing $httpBackend]'s expect method (the replacement for $browser.xhr.expect) is stricter -
  the order of requests matters and a single request expectation can handle only a single request.
- compiler
  - compiler is a service, so use [$compile] instead of angular.compile to compile templates
  - $compile (nee angular.compile) returns the linking function which takes one mandatory argument -
    the scope. previously this argument was optional and if missing, the compiler would create a new
    root scope, this was a source of bugs and was removed
- filters
  - filters need to be registered either via [moduleName.filter][angular.Module] or
    [$filterProvider.filter][$filterProvider]
  - filters don't have access to the dom element
  - currency filter doesn't make negative values red
  - json filter doesn't print out stuff in monospace
- type augmentation via angular.Array, and angular.Object is gone. As a replacement use filters
  ([filter], [limitTo], [orderBy]), ES5 apis (e.g. Array#indexOf), or create custom filters (e.g.
  as a replacement for $count and $sum).
- [$browser.defer.flush] now throws an exception when queue is empty
  ([commit](https://github.com/angular/angular.js/commit/63cca9afbcf7a772086eb4582d2f409c39e0ed12))
- scope.$apply and scope.$digest throws an exception if called while $apply or $digest is already
  in progress (this is a programming error, you should never need to do this)
  ([commit](https://github.com/angular/angular.js/commit/0bf611087b2773fd36cf95c938d1cda8e65ffb2b))


<a name="0.10.5"></a>
# 0.10.5 steel-fist (11-11-08) #

## Features:

- [ng:autobind]: drop angular.js file name restrictions
  ([commit](https://github.com/angular/angular.js/commit/d7ba5bc83ba9a8937384ea677331c5156ed6772d))
- [Scope]: better logging of infinite digest error
  ([commit](https://github.com/angular/angular.js/commit/ef875ad0cf4349144cb4674e050dd160564f6dd9),
  issue [#621](https://github.com/angular/angular.js/issues/621))
- enable [widget] styling in IE8 and below using
  [html5shiv](http://code.google.com/p/html5shiv/)-like approach
  ([commit](https://github.com/angular/angular.js/commit/163c799effd5cfadc57990f4d4127651bae3fbdb),
  issue [#584](https://github.com/angular/angular.js/issues/584))
- [ng:style]: compatibility + perf improvements
  ([commit](https://github.com/angular/angular.js/commit/e2663f62b0fbb8b9ce2e706b821a135e0bc7e885))


## Bug Fixes:
- [ng:view]: ignore stale xhr callbacks - fixes issues caused by race-conditions which occured when
  user navigated to a new route before the current route finished loading
  (issue [#619](https://github.com/angular/angular.js/issues/619))
- [ng:form] should always be a block level (css) element
  ([commit](https://github.com/angular/angular.js/commit/02dc81bae0011b7ae4190363be5fdd5db420aca9))
- Fixes for [e2e test runner]'s `$location` dsl
  ([commit](https://github.com/angular/angular.js/commit/dc8ffa51b7ebe5fb9bc1c89087c8b3c9e65d1006))
- [ng:repeat] when iterating over arrays ignore non-array properties + when iterating over objects
  sort keys alphabetically
  ([commit](https://github.com/angular/angular.js/commit/3945f884c5777e629b57c9ab0e93b9d02b9840d0))

## Docs:
- experimental [disqus.com](http://disqus.com/) integration for all docs-next.angularjs.org pages
  ([commit](https://github.com/angular/angular.js/commit/28ed5ba46595a371bd734b92a6e4bb40d1013741),
  contributed by Dan Doyon)
- [e2e test runner] docs were moved to the dev guide



<a name="0.10.4"></a>
# 0.10.4 human-torch (2011-10-22) #

## Features:

- New validation options for
  [input widgets](http://docs-next.angularjs.org/api/angular.widget.input): `ng:minlength` and
  `ng:maxlength`
  ([commit](https://github.com/angular/angular.js/commit/78f394fd17be581c84ecd526bb786ed1681d35cb))
  (contributed by Konstantin Stepanov)
- HTML sanitizer was updated to recognize all safe HTML5 elements
  (Issue [#89](https://github.com/angular/angular.js/issues/89))
- [ng:options]' blank option is now compiled and data-bound as any other template
  (Issue [#562](https://github.com/angular/angular.js/issues/562))
  (contributed by tehek)
- [$defer](http://docs-next.angularjs.org/api/angular.service.$defer) service now exposes `cancel`
  method for task cancellation
  ([commit](https://github.com/angular/angular.js/commit/ad90c3574f8365ee4a1a973d5e43c64fe9fcda2c))


## Bug Fixes:

- [ng:options] should select correct element when '?'-option (invalid value) was previously selected
  (Issue [#599](https://github.com/angular/angular.js/issues/599)) (contributed by Tehek)
- Fix data-binding of radio button's value property
  (Issue [#316](https://github.com/angular/angular.js/issues/316))
- Input with type `password` should no be turned into a readable text field
  ([commit](https://github.com/angular/angular.js/commit/e82e64d57b65d9f3c4f2e8831f30b615a069b7f6))
  (contributed by Konstantin Stepanov)
- [ng:repeat] should ignore object properties starting with `$`
  ([commit](https://github.com/angular/angular.js/commit/833eb3c84445110dc1dad238120573f08ed8d102))
- Correctly parse out inlined regexp from the input field's `ng:pattern` attribute.
  ([commit](https://github.com/angular/angular.js/commit/5d43439dbe764a4c7227f51b34a81b044f13901b))
- $location service in html5 mode should correctly rewrite links that contain nested elements
  ([commit](https://github.com/angular/angular.js/commit/9b85757102fbd44e88d0a3909fdf8b90f191b593))


## Breaking Changes:

- the [date] filter now uses 'mediumDate' format if none is specified. This was done to deal with
  browser inconsistencies (each browser used to use different format)
  (Issue [#605](https://github.com/angular/angular.js/issues/605),
   [commit](https://github.com/angular/angular.js/commit/c6c3949b14f4003ecab291243edfca61262f2c3d),
   [commit](https://github.com/angular/angular.js/commit/e175db37c6f52bba4080efeec22a7120a896099e))
- calling the linker function returned by [angular.compile][compile] doesn't automatically run
  `$digest` on the linked scope any more. This behavior was briefly introduced in 0.10.3 but was
  causing issues and inefficiencies in production apps so we reverted it. See:
  [commit](https://github.com/angular/angular.js/commit/f38010d3a2f457a53798212ef72418637dabe189)




<a name="0.10.3"></a>
# 0.10.3 shattering-heartbeat (2011-10-13) #

## Features:

- New forms, validation, support for HTML5 input widgets. Please check out:
  - [Forms overview](http://docs-next.angularjs.org/guide/dev_guide.forms)
  - [form widget](http://docs-next.angularjs.org/api/angular.widget.form)
  - [input widget](http://docs-next.angularjs.org/api/angular.widget.input)
  - [$formFactory service](http://docs-next.angularjs.org/api/angular.service.$formFactory)
  - [angular.inputType](http://docs-next.angularjs.org/api/angular.inputType)
  - [commit](https://github.com/angular/angular.js/commit/4f78fd692c0ec51241476e6be9a4df06cd62fdd6)

- [ng:repeat] now has element-model affinity, which makes it more friendly to third-party code that
  is not aware of angular's DOM manipulation. This is also the pre-requisite for supporting
  animations.
  ([commit](https://github.com/angular/angular.js/commit/75f11f1fc46c35a28c0905f7316ea6779145e2fb))


## Bug Fixes:

- The select widget with [ng:options] directive now correctly displays selected option (regression
  from 0.10.2).
- Fix for jqLite's removeClass, which under certain circumstances could clobber class names.
  ([commit](https://github.com/angular/angular.js/commit/b96e978178a6acbf048aa6db466ed845e1395445))
- Other small fixes and documentation improvements.


## Breaking Changes:

- Due to changes in how forms and validation works the following were replaced with new apis:
  - `angular.formatter` - use `angular.inputType` or form's `$createWidget`
  - `angular.validator` - use `angular.inputType` or form's `$createWidget`
  - changes to `<input>` and `<select>` elements
    - `ng:model` directive is now required for data-binding to kick in
    - the `name` attribute is now optional and is used only as an alias when accessing the input
      widget via the form object.
    - view can't affect the model without a user interaction, so the `value` attribute of the
      `<input>` element and `selected` attribute of the `<option>` element if specified in the
      template is ignored.
- Removed decoration of DOM elements when:
  - an exception occurs - when an exception happens, it will be passed to the $exceptionHandler
    service, which can decide what to do with it.
  - an input widget contains invalid input - in this case the forms validation apis can be used to
    display a customized error message.
- The $hover service was removed (it was needed only for the DOM decoration described above).




<a name="0.10.2"></a>
# 0.10.2 sneaky-seagull (2011-10-08) #

## Features:

- jQuery 1.6.4 support (Issue [#556](https://github.com/angular/angular.js/issues/556))
- [jqLite](http://docs-next.angularjs.org/api/angular.element) improvements:
  - Added support for `prop` method
    ([commit](https://github.com/angular/angular.js/commit/3800d177030d20c5c3d04e3601f892c46e723dc2))
  - Added support for `unbind` method
    ([commit](https://github.com/angular/angular.js/commit/6b7ddf414de82720bbf547b2fa661bf5fcec7bb6))


## Bug Fixes:

- Added support for short-circuiting of && and || operators in in angular expressions
  (Issue [#433](https://github.com/angular/angular.js/issues/433))
- Fix for [$limitTo] to properly handle excessive limits (contributed by tehek)
  (Issue [#571](https://github.com/angular/angular.js/issues/571))
- [jqLite]'s css() method now converts dash-separated css property names to camelCase in order to
  support dash-separated properties on Firefox
  (Issue [#569](https://github.com/angular/angular.js/issues/569))
- action defaults for [$resource]s now take precedence over resource defaults (contributed by
  Marcello Nuccio)
  ([commit](https://github.com/angular/angular.js/commit/bf5e5f7bc9ebc7dc6cf8fdf3c4923498b22a8654))
- Fixed escaping issues in [$route] matcher
  ([commit](https://github.com/angular/angular.js/commit/2bc39bb0b4f81b77597bb52f8572d231cf4f83e2))
- Fixed two issues in $browser.defer.cancel mock
  ([commit](https://github.com/angular/angular.js/commit/62ae7fccbc524ff498779564294ed6e1a7a3f51c),
   [commit](https://github.com/angular/angular.js/commit/8336f3f0ba89b529057027711ab4babd6c2cb649))
- Fix for ng:options, which under certain circumstances didn't select the right option element
  ([commit](https://github.com/angular/angular.js/commit/555f4152909e1c0bd5400737a62dc5d63ecd32d3))


## Docs:

- migrated the docs app to use [$location]'s HTML5 mode (hashbang urls no more)
  ([commit](https://github.com/angular/angular.js/commit/13f92de6246a0af8450fde84b209211a56397fda))


## Breaking Changes

- If Angular is being used with jQuery older than 1.6, some features might not work properly. Please
  upgrade to jQuery version 1.6.4.

## Breaking Changes
- ng:repeat no longer has ng:repeat-index property. This is because the elements now have
  affinity to the underlying collection, and moving items around in the collection would move
  ng:repeat-index property rendering it meaningless.


<a name="0.10.1"></a>
# 0.10.1 inexorable-juggernaut (2011-09-09) #

## Features

- complete rewrite of the $location service with HTML5 support, many API and semantic changes.
  Please see:
  - [$location service API docs](http://docs-next.angularjs.org/#!/api/angular.module.ng.$location)
  - [$location service dev guide article](http://docs-next.angularjs.org/#!/guide/dev_guide.services.$location)
  - [location.js source file](https://github.com/angular/angular.js/blob/master/src/service/location.js)
  - breaking changes section of this changelog


## Bug Fixes

- $xhr should not covert HTTP status 0 to 200
  ([commit](https://github.com/angular/angular.js/commit/b0eb831bce7d0ea066fd0758124793ed3db6d692))
- fixed several doc examples that were broken on IE
- ng:change should be called after the new val is set
  (Issue [#547](https://github.com/angular/angular.js/issues/547))
- currency filter should return an empty string for non-numbers


## Breaking Changes

- $location related changes - for complete list of api changes see:
  [Migrating from earlier AngularJS releases](http://docs-next.angularjs.org/#!/guide/dev_guide.services.$location)
  - $location api changes:
    - $location.href -> $location.absUrl()
    - $location.hash -> $location.url()
    - $location.hashPath -> $location.path()
    - $location.hashSearch -> $location.search()
    - $location.search -> no equivalent, use $window.location.search (this is so that we can work in
      hashBang and html5 mode at the same time, check out the docs)
    - $location.update() / $location.updateHash() -> use $location.url()
    - n/a -> $location.replace() - new api for replacing history record instead of creating a new one

  - $location semantic changes:
    - all url pieces are always in sync ($location.path(), $location.url(), $location.search(), ...) -
      this was previously true only if you used update* methods instead of direct assignment
      ($location.hashPath = 'foo')
    - we now use (window.history.pushState || onHashChange event || polling) for detecting url changes
      in the browser (we use the best one available).



<a name="0.10.0"></a>
# 0.10.0 chicken-hands (2011-09-02) #

## Features

- complete rewrite of the Scope implementation with several API and semantic changes. Please see:
  - [angular.scope API docs](http://docs-next.angularjs.org/#!/api/angular.scope)
  - [scopes dev guide article](http://docs-next.angularjs.org/#!/guide/dev_guide.scopes)
  - [scope.js source file](https://github.com/angular/angular.js/blob/master/src/Scope.js)
  - breaking changes section of this changelog
- added event system to scopes (see [$on], [$emit] and [$broadcast])
- added i18n and l10n support for date, currency and number filters see [i18n] docs for more info
- added localizable [ng:pluralize] widget
- added [ng:cloak] directive for hiding uncompiled templates


## Bug Fixes

- make [ng:class] friendly towards other code adding/removing classes
  ([commit](https://github.com/angular/angular.js/commit/2a8fe56997fddbad673748ce02abf649a709c4ca))
- several [jqLite] bugfixes and improvements
- [ng:href], [ng:src] and friends now work properly when no expression is present in the attribute
  value.
  (Issue [#534](https://github.com/angular/angular.js/issues/534))
- expose missing [lowercase], [uppercase] and [isDate] APIs.


## Docs

- many (but not all just yet) api docs were proof-read and improved


## Breaking Changes:

- many scope related changes:
  - $onEval is no more (use $watch with a fn as the only param if you really miss it)
  - $eval without params doesn't trigger model mutation observations (use $apply/$digest instead)
  - $digest propagates through the scope tree automatically (this is the desired behavior anyway)
  - $watch various API changes
    - scope is now the first argument passed into the $watch listener
    - `this` in the $watch listener is undefined instead of current scope
    - objects and arrays are watched and compared by equality and not just identity
    - the initial execution of the $watch listener now executes asynchronously with respect to the
      code registering it via $watch
    - exceptionHandler argument is no more
    - initRun argument is no more
  - angular.scope does not create child scopes by taking parent as the first argument - use $new
    instead
  - scope.$set and scope.$get were removed, use direct property assignment instead or $eval
- $route.onChange was removed and replaced with $beforeRouteChange, $afterRouteChange and
  $routeUpdate events that can be used together with the new $routeParams service
- `angular.equals()` now uses `===` instead of `==` when comparing primitives



<a name="0.9.19"></a>
# 0.9.19 canine-psychokinesis (2011-08-20) #

## Features
- added error handling support for JSONP requests (see error callback param of the [$xhr] service)
  ([commit](https://github.com/angular/angular.js/commit/05e2c3196c857402a9aa93837b565e0a2736af23))
- exposed http response headers in the [$xhr] and [$resource] callbacks
  ([commit](https://github.com/angular/angular.js/commit/4ec1d8ee86e3138fb91543ca0dca28463895c090)
  contributed by Karl Seamon)
- added `reloadOnSearch` [$route] param support to prevent unnecessary controller reloads and
  resulting flicker
  ([commit](https://github.com/angular/angular.js/commit/e004378d100ce767a1107180102790a9a360644e))


## Bug Fixes
- fixed memory leak found in [ng:options] directive
  ([commit](https://github.com/angular/angular.js/commit/6aa04b1db48853340d720e0a1a3e325ac523a06f))
- make ng:class-even/odd compatible with ng:class
  (Issue [#508](https://github.com/angular/angular.js/issues/508))
- fixed error handling for resources that didn't work in certain situations
  ([commit](https://github.com/angular/angular.js/commit/c37bfde9eb31556ee1eb146795b0c1f1504a4a26)
  contributed by Karl Seamon)


## Docs
- [jsFiddle](http://jsfiddle.net/) integration for all docs.angularjs.org examples (contributed by
  Dan Doyon).


## Breaking Changes
- removed [jqLite] show/hide support. See the
  [commit](https://github.com/angular/angular.js/commit/4c8eaa1eb05ba98d30ff83f4420d6fcd69045d99)
  message for details. Developers should use jquery or jqLite's `css('display', 'none')` and
  `css('display', 'block'/'inline'/..)` instead


<a name="0.9.18"></a>
# 0.9.18 jiggling-armfat (2011-07-29) #

### Features
- [ECMAScript 5 Strict Mode](https://developer.mozilla.org/en/JavaScript/Strict_mode) compliance
- [jqLite]
  - added `show()`, `hide()` and `eq()` methods to jqlite
    ([commit](https://github.com/angular/angular.js/commit/7a3fdda9650a06792d9278a8cef06d544d49300f))
- added $defer.cancel to support cancelation of tasks defered via the [$defer] service
- [date] filter
  - added support for `full`, `long`, `medium` and `short` date-time format flags
    ([commit](https://github.com/angular/angular.js/commit/3af1e7ca2ee8c2acd69e5bcbb3ffc1bf51239285))
  - added support for `z` flag, which stands for short string timezone identifier, e.g. PST
  - internal improvements to enable localization of date filter output
- [number] filter
  - internal improvements to enable localization of number filter output
- [currency] filter
  - support for custom currency symbols via an optional param
  - internal improvements to enable localization of number filter output
- added [angular.version] for exposing the version of the loaded angular.js file
- updated angular.js and angular.min.js file headers with angular version and shorter & updated
  license info
- [ng:options]
  - support binding to expression (Issue [#449](https://github.com/angular/angular.js/issues/449))
  - support iterating over objects (Issue [#448](https://github.com/angular/angular.js/issues/448))
  - support ng:change (Issue [#463](https://github.com/angular/angular.js/issues/463))
  - support option groups (`<optgroup>`)
    (Issue [#450](https://github.com/angular/angular.js/issues/450))
- [$xhr] and [$resource] support for per-request error callbacks (Issue
  [#408](https://github.com/angular/angular.js/issues/408)) (contributed by Karl Seamon)


### Bug Fixes
- make injector compatible with Rhino (HtmlUnit) (contributed by M√•rten Dolk)
  [commit](https://github.com/angular/angular.js/commit/77ba539f630c57b17d71dbf1e9c5667a7eb603b7)
- `ie-compat.js` fixes and improvements related to fetching this file on the fly on legacy browsers
- [jqLite]
  - fix `bind()` when binding to more events separated by space
    [commit](https://github.com/angular/angular.js/commit/9ee9ca13da3883d06733637f9048a83d94e6f1f8)
  - non-existing attributes should return undefined just like in jQuery
    [commit](https://github.com/angular/angular.js/commit/10da625ed93511dbf5d4e61ca4e42f6f2d478959)
  - set event.target for IE<8
    [commit](https://github.com/angular/angular.js/commit/ce80576e0b8ac9ed5a5b1f1a4dbc2446434a0002)
- improved implementation of [ng:show] and [ng:hide] directives by using jqLite/jQuery hide and
  show methods
- [ng:options]
  - fix incorrect re-growing of options on datasource change
    (Issue [#464](https://github.com/angular/angular.js/issues/464))


### Docs
- added full offline support for docs (click on the link in the footer of docs.angularjs.org)
- many content improvements and corrections across all docs (reference api, tutorial, dev guide)
- many small design improvements


### Other
- doubled our e2e test suite by running all angular e2e tests with jqLite in addition to jQuery


### Breaking changes
- [commit](https://github.com/angular/angular.js/commit/3af1e7ca2ee8c2acd69e5bcbb3ffc1bf51239285)
  removed support for the `MMMMM` (long month name), use `MMMM` instead. This was done to align
  Angular with
  [Unicode Technical Standard #35](http://unicode.org/reports/tr35/#Date_Format_Patterns) used by
  Closure, as well as, future DOM apis currently being proposed to w3c.
- `$xhr.error`'s `request` argument has no `callback` property anymore, use `success` instead



<a name="0.9.17"></a>
# <angular/> 0.9.17 vegetable-reanimation (2011-06-30) #

### New Features
- New [ng:options] directive to better bind a model to `<select>` and `<option>` elements.
- New [ng:disabled], [ng:selected], [ng:checked], [ng:multiple] and [ng:readonly] directives.
- Added support for string representation of month and day in [date] filter.
- Added support for `prepend()` to [jqLite].
- Added support for configurable HTTP header defaults for the [$xhr] service.


### Bug Fixes
- Number filter would return incorrect value when fractional part had leading zeros.
- Issue #338: Show error when template with with multiple DOM roots is being compiled.
- Issue #399: return unsorted array if no predicate.
- Fixed issues with incorrect value of $position in ng:repeat when collection size changes.
- Fixed JSONP support in [$xhr] which didn't work without jquery since v0.9.13.


### Documentation
- various small fixes and improvements


### Breaking changes
- $service now has $service.invoke for method injection ($service(self, fn) no longer works)
- injection name inference no longer supports method curry and linking functions. Both must be
  explicitly specified using $inject property.
- Dynamic iteration (ng:repeat) on `<option>` elements is no longer supported. Use ng:options
- Removal of index formatter (`ng:format="index"`) since its only use was with repeated `<options>`
  (see above).
- Calling [$orderBy] without a predicate now returns the original unsorted array, instead of
  ordering by natural order.



<a name="0.9.16"></a>
# <angular/> 0.9.16 weather-control (2011-06-07) #

### Features
- [JsTD Scenario Adapter] for running scenario tests with jstd (from command line and in multiple
  browsers)


### Documentation
- brand new template for <http://docs.angularjs.org/>
- brand new tutorial that describes how to build a typical angular app
  <http://docs.angularjs.org/#!/tutorial>
- lots of new content for the dev guide (still work in progress)
  <http://docs.angularjs.org/#!/guide>


### Bug Fixes
- ng:href produces unclickable links on IE7 [#352](https://github.com/angular/angular.js/issues/352)
- IE 8 in compatibility mode breaks routing [#353](https://github.com/angular/angular.js/issues/353)
- IE translates a 204 response code to 1223 [#357](https://github.com/angular/angular.js/issues/357)
- Fixed unit test in IE7 [#360](https://github.com/angular/angular.js/pull/360)
- Fixed unit tests on FF4, Opera [#364](https://github.com/angular/angular.js/pull/364)
- Fixed opera date.toISOString issue [#367](https://github.com/angular/angular.js/pull/367)


### Breaking changes
- html scenario runner requires ng:autotest script attribute to start tests automatically
  ([example](https://github.com/angular/angular.js/blob/master/example/personalLog/scenario/runner.html#L5))



<a name="0.9.15"></a>
# <angular/> 0.9.15 lethal-stutter (2011-04-11) #

### Features
- IE9 support


### Bug Fixes
- reverted [ng:view] sync cache fix due to regression in the order of initialization of parent
  and child controllers. (commits 9bd2c396 and 3d388498)
- [$resource] success callback is now executed whenever the http status code is `<200,300>`


### Docs
- fixed intentation code that caused some of the snippets on docs.angularjs.org to be mangled.
- many small improvements of the api docs.



<a name="0.9.14"></a>
# <angular/> 0.9.14 key-maker (2011-04-01) #

### Performance
- [ng:repeat] grows (adds children) significantly faster. (commit 15ec78f5)
- [$xhr.cache] optionally executes callbacks synchronously. (commit c06c5a36)
- [ng:view] and [ng:include] use sync [$xhr.cache]


### Bug Fixes
- Fixed [$resource] encoding of query params. (commits e1d122a4, 78a0f410)


### House cleaning
- code cleanup
- better minification (min is now 2.5% or almost 1kb smaller)
- minor documentation fixes
- JsTestDriver 1.3.2 upgrade with fixed coverage support



<a name="0.9.13"></a>
# <angular/> 0.9.13 curdling-stare (2011-03-13) #

### New Features
- Added XSRF protection for the [$xhr] service. (commit c578f8c3)
- Targeted auto-bootstrap – [ng:autobind] now takes an optional value which specifies an element id
  to be compiled instead of compiling the entire html document. (commit 9d5c5337)


### Bug Fixes
- Fixed IE7 regression which prevented angular from bootstrapping in this browser.
- Cookies which contain unescaped '=' are now visible via the [$cookies] service. (commit 26bad2bf)
- [$xhr] service now executes "success" callback for all 2xx responses, not just 200.
  (commit 5343deb3)
- Always remove the script tag after successful JSONP request. (commit 0084cb5c)
- Removal of all `document.write` statements to make angular compabile with async script loaders.
  (commit 3224862a)


### Breaking changes
- The `post` parameter of [$browser.xhr][$browser] is now non-optional. Since everyone should be
  using the [$xhr] service instead of $browser.xhr, this should not break anyone. If you do use
  $browser.xhr then just add null for the post value argument where post was not passed in.




<a name="0.9.12"></a>
# <angular/> 0.9.12 thought-implanter (2011-03-03) #

### API
- Added a delay parameter to the [$defer] service. (commit edbe9d8c)
- Added `scope()` method to [angular.element][element] (jQuery) instances to retrieve a [scope]
  associated with a given DOM element. (commit 0a5c00ab)
- Added inference of DI dependencies from function signature. This feature is experimental, check
  out [dependency injection][guide.di] docs. (commit 7d4aee31)


### New Features
- Angular now correctly recognizes and uses jQuery even if it was loaded after angular's script.
  More info at [angular.element][element]. (commit a004d487)
- All built-in angular services are now lazy-loaded. (commit a070ff5a)
- To make styling of custom html tags created via [widgets][widget] and [directives][directive]
  easier, all of these elements now contain a css class with name in form of
  `<namespace>-<directive/widget name>`, e.g. `<ng:include class="ng-include">`. (commit c7998f5f)
- [$xhr] service now automatically detects and strips google-style JSON security prefix from http
  responses. (commit cd139f57)


### Bug Fixes
- Rewrite of JQuery lite implementation for better supports operations on multiple nodes when
  matched by a selector and remove other bugs. (commit 00cc9eb3)
- Corrected an issue where properties inherited from \_\_proto\_\_ show up in ng:repeat.
  (commit 9e67da42)
- Fixed url encoding issue affecting [$resource] service. (commits e9ce2259 + 9e30baad)
- Removed `$eval()` call from the [$cookies] factory function, which was causing duplicate
  instances of singleton services to be created. (commit 65585a2d)


### Docs
- New docs [contribution guidelines][contribute].
- New [description of release artifacts][downloading].
- Lots of improvements and other new content.


### Breaking changes
- Removed the `$init()` method that used to be called after compilation of a template. This should
  affect only fraction of angular apps because the api was primarily being used by low level widgets
  tests.

  The old way of compiling the DOM element was angular.compile(element).$init(); The $init was there
  to allow the users to do any work to the scope before the view would be bound. This is a left over
  from not having proper MVC. The new recommended way to deal with initializing scope is to put it
  in the root constructor controller. To migrate simply remove the call to $init() and move any code
  you had before $init() to the root controller.

  (commit 23b255a8)
- Changed [angular.compile][compile] API from `angular.compile(element[, scope])` to
  `angular.compile(element)([scope], [cloneAttachFn])` (commits ef4bb28b + 945056b1)
- Removed ng:watch directives since it encourages logic in the UI. (commit 87cbf9f5)




<a name="0.9.11"></a>
# <angular/> 0.9.11 snow-maker  (2011-02-08) #

### Documentation
- completed migration of docs from the wiki site to
  [http://docs.angularjs.org/](http://docs.angularjs.org/)
- many, but by far not all, docs were updated, improved and cleaned up

### Features
- [$route] service now supports these features:
  - route not found handling via `#otherwise()`
  - redirection support via `#when('/foo', {redirectTo: '/bar'})` (including param interpolation)
  - setting the parent scope for scopes created by the service via `#parent()`
  - reloading the current route via `#reload()`

### API
- added `angular.element(...).scope()` method to retrieve scope for a given element.

### Bug Fixes
- <option> value attribute gets clobbered when the element contains new line character(s).
- <ng:view> widget now works when nested inside an <ng:include> widget
- other various small fixes

### Breaking changes
- mock [`$browser`](http://docs.angularjs.org/#!/api/angular.mock.service.$browser) now throws an
  exception if the `flush()` method is called when there are no requests to be flushed. If you
  experience `No xhr requests to be flushed!` errors in your tests, it's because you called
  `$browser.xhr.flush()` unexpectedly. To make the error go away, either make sure your code makes a
  request via the `$xhr` service or remove all unneeded `flush()` calls.


<a name="0.9.10"></a>
# <angular/> 0.9.10 flea-whisperer  (2011-01-26) #

### Features
- new [`ng:view`](http://docs.angularjs.org/#!/api/angular.widget.ng-view) widget to simplify integration
with the `$route` service
- the content of all standard HTML widgets is now being processed
  (e.g. `<button>{{foo}}</button>` works now) (commit 1d7b9d56)
- new [`$log`](http://docs.angularjs.org/#!/api/angular.mock.service.$log) and
  [`$exceptionHandler`](http://docs.angularjs.org/#!/api/angular.mock.service.$exceptionHandler) service
  mocks now part of `angular-mocks.js` (commit f5d08963)

### Bug Fixes
- <select> (one/multiple) could not chose from a list of objects (commit 347be5ae)
- null and other falsy values should not be rendered in the view (issue #242)

### Docs
- rewrite of several major portions of angular.service.*, angular.Array.*, angular.Object.* docs
- added support for [sitemap]((http://docs.angularjs.org/sitemap.xml) to make the docs indexable by
  search crawlers
- transition of Developer Guide docs from the wiki into docs.angularjs.org
- lots of improvements related to formatting of the content of docs.anguarjs.org


<a name="0.9.9"></a>
# <angular/> 0.9.9 time-shift (2011-01-13) #

### Security
- Added a just in case security check for JSON parsing. (commit 5f080193)
- Completed security review with the Google Security Team.

### Performance
- $location and $cookies services are now lazily initialized to avoid the polling overhead when
  not needed.
- $location service now listens for `onhashchange` events (if supported by browser) instead of
  constant polling. (commit 16086aa3)
- input widgets known listens on keydown events instead of keyup which improves perceived
  performance (commit 47c454a3)
- angular boots significantly sooner by listening for DOMContentLoaded event instead of
  window.load when supported by browser (commit c79aba92)
- new service $updateView which may be used in favor of $root.$eval() to run a complete eval on
  the entire document. This service bulks and throttles DOM updates to improve performance.
  (commit 47c454a3)

### Docs
- Major improvements to the doc parser (commit 4f22d686)
- Docs now offline enabled (all dependencies are bundled in the tarball) (commit 4f5d5029)
- Added support for navigating the docs app with keyboard shortcuts (tab and ctrl+alt+s)

### Bugfixes
- `angular.Object.equals` now properly handless comparing an object with a null (commit b0be87f6)
- Several issues were addressed in the `$location` service (commit 23875cb3)
- angular.filter.date now properly handles some corner-cases (issue #159 - fix contributed by Vojta)

### Breaking changes
- API for accessing registered services ‚Äî `scope.$inject` ‚Äî was renamed to
  [`scope.$service`](http://docs.angularjs.org/#!/api/angular.scope.$service). (commit b2631f61)

- Support for `eager-published` services was removed. This change was done to make explicit
  dependency declaration always required in order to allow making relatively expensive services
  lazily initialized (e.g. $cookie, $location), as well as remove 'magic' and reduce unnecessary
  scope namespace pollution. (commit 3ea5941f)

  Complete list of affected services:

  - $location
  - $route
  - $cookies
  - $window
  - $document
  - $exceptionHandler
  - $invalidWidgets

  To temporarily preserve the 'eager-published' status for these services, you may use `ng:init`
  (e.g. `ng:init="$location = $service('$location'), ...`) in the view or more correctly create
  a service like this:

      angular.service('published-svc-shim', function($location, $route, $cookies, $window,
          $document, $exceptionHandler, $invalidWidgets) {
        this.$location = $location;
        this.$route = $route;
        this.$cookies = $cookies;
        this.$window = $window;
        this.$document = $document;
        this.$exceptionHandler = $exceptionHandler;
        this.$invalidWidgets = $invalidWidgets;
      }, {$inject: ['$location', '$route', '$cookies', '$window', '$document', '$exceptionHandler',
                    '$invalidWidgets'],
          $eager: true});

- In the light of the `eager-published` change, to complete the cleanup we renamed `$creation`
  property of services to `$eager` with its value being a boolean.
  To transition, please rename all `$creation: 'eager'` declarations to `$eager: true`.
  (commit 1430c6d6)

- `angular.foreach` was renamed to `angular.forEach` to make the api consistent. (commit 0a6cf70d)

- The `toString` method of the `angular.service.$location` service was removed. (commit 23875cb3)


<a name="0.9.8"></a>
# <angular/> 0.9.8 astral-projection (2010-12-23) #

### Docs/Getting started
- angular-seed project to get you hacking on an angular apps quickly
  https://github.com/angular/angular-seed

### Performance
- Delegate JSON parsing to native parser (JSON.parse) if available

### Bug Fixes
- Ignore input widgets which have no name (issue #153)


<a name="0.9.7"></a>
# <angular/> 0.9.7 sonic-scream (2010-12-10) #

### Bug Fixes
- $defer service should always call $eval on the root scope after a callback runs (issue #189)
- fix for failed assignments of form obj[0].name=value (issue #169)
- significant parser improvements that resulted in lower memory usage
  (commit 23fc73081feb640164615930b36ef185c23a3526)

### Docs
- small docs improvements (mainly docs for the $resource service)

### Breaking changes
- Angular expressions in the view used to support regular expressions. This feature was rarely
  used and added unnecessary complexity. It not a good idea to have regexps in the view anyway,
  so we removed this support. If you had any regexp in your views, you will have to move them to
  your controllers. (commit e5e69d9b90850eb653883f52c76e28dd870ee067)


<a name="0.9.6"></a>
# <angular/> 0.9.6 night-vision (2010-12-06) #

### Security
- several improvements in the HTML sanitizer code to prevent code execution via `href`s and other
  attributes.
  Commits:
  - 41d5938883a3d06ffe8a88a51efd8d1896f7d747
  - 2bbced212e2ee93948c45360fee00b2e3f960392

### Docs
- set up http://docs.angularjs.org domain, the docs for the latest release will from now on be
  deployed here.
- docs app UI polishing with dual scrolling and other improvements

### Bug Fixes
- `select` widget now behaves correctly when it's `option` items are created via `ng:repeat`
  (issue #170)
- fix for async xhr cache issue #152 by adding `$browser.defer` and `$defer` service

### Breaking Changes
- Fix for issue #152 might break some tests that were relying on the incorrect behavior. The
  breakage will usually affect code that tests resources, xhr or services/widgets build on top of
  these. All that is typically needed to resolve the issue is adding a call to
  `$browser.defer.flush()` in your test just before the point where you expect all cached
  resource/xhr requests to return any results. Please see 011fa39c2a0b5da843395b538fc4e52e5ade8287
  for more info.
- The HTML sanitizer is slightly more strinct now. Please see info in the "Security" section above.


<a name="0.9.5"></a>
# <angular/> 0.9.5 turkey-blast (2010-11-25) #

### Docs
- 99% of the content from the angular wiki is now in the docs

### Api
- added `angular.Array.limitTo` to make it easy to select first or last few items of an array


<a name="0.9.4"></a>
# <angular/> 0.9.4 total-recall (2010-11-18) #

### Docs
- searchable docs
- UI improvements
- we now have ~85% of the wiki docs migrated to ng docs
- some but not all docs were updated along the way


### Api
- ng:include now supports `onload` attribute (commit cc749760)

### Misc
- Better error handling - compilation exception now contain stack trace (commit b2d63ac4)


<a name="0.9.3"></a>
# <angular/> 0.9.3 cold-resistance (2010-11-10) #

### Docs
- prettier docs app with syntax highlighting for examples, etc
- added documentation, examples and scenario tests for many more apis including:
  - all directives
  - all formatters
  - all validators
  - some widgets

### Api
- date filter now accepts strings that angular.String.toDate can convert to Date objects
- angular.String.toDate supports ISO8061 formated strings with all time fractions being optional
- ng:repeat now exposes $position with values set to 'first', 'middle' or 'last'
- ng:switch now supports ng:switch-default as fallback switch option

### Breaking changes
- we now support ISO 8601 extended format datetime strings (YYYY-MM-DDTHH:mm:ss.SSSZ) as defined
  in EcmaScript 5 throughout angular. This means that the following apis switched from
  YYYY-MM-DDTHH:mm:ssZ to YYYY-MM-DDTHH:mm:ss.SSSZ (note the added millis) when representing dates:
  - angular.Date.toString
  - angular.String.fromDate
  - JSON serialization and deserialization (used by json filter, $xhr and $resource)
- removed SSN validator. It's unlikely that most people will need it and if they do, it can be added
  simple RegExp validator.


<a name="0.9.2"></a>
# <angular/> 0.9.2 faunal-mimicry (2010-11-03) #

### Docs
- created documentation framework based on jsdoc syntax (commit 659af29a)
  - jsdoc parser
  - template generator
  - json generator
  - angular doc viewer app
  - scenario runner for all example code
- documentation for all angular filters (commits 1fe7e3a1 & 1ba8c2a33)
  - docs
  - example code
  - scenario tests for example code

### Testability
#### Scenario Runner
- binding DSL in Scenario can now match bindings without specifying filters
- dsl statements now accept a label argument to make test output more readable (issue #94)
- dsl element() statement now implements most of the jQuery API (issue #106)
- new browser() dsl statement for getting info about the emulated browser running the app
  (issue #109)
- scenario runner is now compatible with IE8 (issue #93)
- scenarior runner checks if URL would return a non-success status code (issue #100)
- binding() DSL now accepts regular expressions
- new textarea() scenario runner DSL for entering text into textareas

### Misc
- lots of small bugfixes

### Breaking changes
#### Scenario Runner
- navigating to about:blank is no longer supported. It results in a sandbox error
- navigateTo() is now browser().navigateTo(). Old code must be updated
- file:// URLs are no longer supported for running a scenario. You must use a web server that
  implements HEAD


<a name="0.9.1"></a>
# <angular/> 0.9.1 repulsion-field (2010-10-26) #

### Security
- added html sanitizer to fix the last few known security issues (issues #33 and #34)

### API
- new ng:submit directive for creating onSubmit handlers on forms (issue #76)
- the date filter now accepts milliseconds as well as date strings (issue #78)
- the html filter now supports 'unsafe' option to bypass html sanitization

### Testability
- lots of improvements related to the scenario runner (commit 40d7e66f)

### Demo
- added a new demo application: Personal Log (src example/personalLog)

### Chores
- lots of fixes to get all tests pass on IE
- added TzDate type to allow us to create timezone idependent tests (issue #88)

### Breaking changes
- $cookieStore service is not globally published any more, if you use it, you must request it via
  $inject as any other non-global service
- html filter now sanitizes html content for XSS attacks which may result in different behavior


<a name="0.9.0"></a>
# <angular/> 0.9.0 dragon-breath (2010-10-20) #

### Security
- angular.fromJson not safer (issue #57)
- readString consumes invalid escapes (issue #56)
- use new Function instead of eval (issue #52)

### Speed
- css cleanup + inline all css and images in the main js (issue #64)

### Testability
- initial version of the built-in end-to-end scenario runner (issues #50, #67, #70)

### API
- allow ng:controller nesting (issue #39)
- new built-in date format filter (issue #45)
- $location needs method you call on updates (issue #32)


### Chores
- release versioning + file renaming (issue #69)

### Breaking changes
- $location.parse was replaced with $location.update
- all css and img files were inlined into the main js file, to support IE7 and older app must host
  angular-ie-compat.js file

### Big Thanks to Our Community Contributors
- Vojta Jina




[lowercase]: http://docs.angularjs.org/#!/api/angular.lowercase
[uppercase]: http://docs.angularjs.org/#!/api/angular.uppercase
[isDate]: http://docs.angularjs.org/#!/api/angular.isDate
[scope]: http://docs.angularjs.org/#!/api/angular.scope
[compile]: http://docs.angularjs.org/#!/api/angular.compile
[element]: http://docs.angularjs.org/#!/api/angular.element
[widget]: http://docs.angularjs.org/#!/api/angular.widget
[ng:repeat]: http://docs.angularjs.org/#!/api/angular.widget.@ng:repeat
[ng:view]: http://docs.angularjs.org/#!/api/angular.widget.ng-view
[ng:include]: http://docs.angularjs.org/#!/api/angular.widget.ng-include
[ng:options]: http://docs.angularjs.org/#!/api/angular.directive.ng-options
[ng:disabled]: http://docs.angularjs.org/#!/api/angular.directive.ng-disabled
[ng:selected]: http://docs.angularjs.org/#!/api/angular.directive.ng-selected
[ng:checked]: http://docs.angularjs.org/#!/api/angular.directive.ng-checked
[ng:multiple]: http://docs.angularjs.org/#!/api/angular.directive.ng-multiple
[ng:readonly]: http://docs.angularjs.org/#!/api/angular.directive.ng-readonly
[ng:show]: http://docs.angularjs.org/#!/api/angular.directive.ng-show
[ng:hide]: http://docs.angularjs.org/#!/api/angular.directive.ng-hide
[ng:class]: http://docs.angularjs.org/#!/api/angular.directive.ng-class
[ng:src]: http://docs.angularjs.org/#!/api/angular.directive.ng-src
[ng:href]: http://docs.angularjs.org/#!/api/angular.directive.ng-href
[ng:style]: http://docs.angularjs.org/#!/api/angular.directive.ng-style
[$defer]: http://docs.angularjs.org/#!/api/angular.module.ng.$defer
[$cookies]: http://docs.angularjs.org/#!/api/angular.module.ng.$cookies
[$xhr]: http://docs.angularjs.org/#!/api/angular.module.ng.$xhr
[$xhr.cache]: http://docs.angularjs.org/#!/api/angular.module.ng.$xhr.cache
[$resource]: http://docs.angularjs.org/#!/api/angular.module.ng.$resource
[$route]: http://docs.angularjs.org/#!/api/angular.module.ng.$route
[$orderBy]: http://docs.angularjs.org/#!/api/angular.Array.orderBy
[date]: http://docs.angularjs.org/#!/api/angular.filter.date
[number]: http://docs.angularjs.org/#!/api/angular.filter.number
[currency]: http://docs.angularjs.org/#!/api/angular.filter.currency
[directive]: http://docs.angularjs.org/#!/api/angular.directive
[ng:autobind]: http://docs.angularjs.org/#!/api/angular.directive.ng-autobind
[guide.di]: http://docs.angularjs.org/#!/guide/dev_guide.di
[downloading]: http://docs.angularjs.org/#!/misc/downloading
[contribute]: http://docs.angularjs.org/#!/misc/contribute
[jqLite]: http://docs.angularjs.org/#!/api/angular.element
[angular.version]: http://docs.angularjs.org/#!/api/angular.version
[Jstd Scenario Adapter]: https://github.com/angular/angular.js/blob/master/src/jstd-scenario-adapter/Adapter.js
[i18n]: http://docs-next.angularjs.org/#!/guide/dev_guide.i18n
[ng:pluralize]: http://docs-next.angularjs.org/#!/api/angular.widget.ng-pluralize
[ng:form]: http://docs-next.angularjs.org/api/angular.widget.form
[ng:cloak]: http://docs-next.angularjs.org/#!/api/angular.directive.ng-cloak
[$on]: http://docs-next.angularjs.org/#!/api/angular.scope.$on
[$emit]: http://docs-next.angularjs.org/#!/api/angular.scope.$emit
[$broadcast]: http://docs-next.angularjs.org/#!/api/angular.scope.$broadcast
[$limitTo]: http://docs-next.angularjs.org/api/angular.Array.limitTo
[$location]: http://docs-next.angularjs.org/api/angular.service.$location
[e2e test runner]: http://docs-next.angularjs.org/guide/dev_guide.e2e-testing
[$injector]: http://docs-next.angularjs.org/api/angular.module.AUTO.$injector
[$http]: http://docs-next.angularjs.org/api/angular.module.ng.$http
[$httpBackend]: http://docs-next.angularjs.org/api/angular.module.ng.$httpBackend
[unit-testing $httpBackend]: http://docs-next.angularjs.org/api/angular.module.ngMock.$httpBackend
[e2e-testing $httpBackend]: http://docs-next.angularjs.org/api/angular.module.ngMockE2E.$httpBackend
[$q]: http://docs-next.angularjs.org/api/angular.module.ng.$q
[angular.bootstrap]: http://docs-next.angularjs.org/api/angular.bootstrap
[$anchorScroll]: http://docs-next.angularjs.org/api/angular.module.ng.$anchorScroll
[$cacheFactory]: http://docs-next.angularjs.org/api/angular.module.ng.$cacheFactory
[bootstrapping]: http://docs-next.angularjs.org/guide/dev_guide.bootstrap
[angular.copy]: http://docs-next.angularjs.org/api/angular.copy
[ng:app]: http://docs-next.angularjs.org/api/angular.directive.ng-app
[$compile]: http://docs-next.angularjs.org/api/angular.module.ng.$compile
[$filterProvider]: http://docs-next.angularjs.org/api/angular.module.ng.$filterProvider
[angular.Module]: http://docs-next.angularjs.org/api/angular.Module
[angular.module]: http://docs-next.angularjs.org/api/angular.module
[filter]: http://docs-next.angularjs.org/api/angular.module.ng.$filter.filter
[limitTo]: http://docs-next.angularjs.org/api/angular.module.ng.$filter.limitTo
[orderBy]: http://docs-next.angularjs.org/api/angular.module.ng.$filter.orderBy
[$browser.defer.flush]: http://docs-next.angularjs.org/api/angular.module.ngMock.$browser#defer.flush
[inject]: http://docs-next.angularjs.org/api/angular.mock.inject
[module]: http://docs-next.angularjs.org/api/angular.mock.module
[guide2.di]: http://docs-next.angularjs.org/guide/dev_guide.di
[jqLite2]: http://docs.angularjs.org/#!/api/angular.element
