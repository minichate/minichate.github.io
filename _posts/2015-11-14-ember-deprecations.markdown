---
layout: post
title:  "Handling Ember deprecations gracefully"
date:   2015-11-14 16:04:41
categories: ember javascript
---

Once an Ember project grows to a decent size it tends to become difficult to upgrade to a newer version of Ember, since the number of deprecations introduced grows fairly large.

For example, for years the suggested way of defining properties on Controllers was via the prototype extension `.property()`. Recently though, the Ember Core Team has started recommending that developers stop using the prototype extensions and start using `Ember.computed()` instead. See [the relevant ember-guides pull request](https://github.com/emberjs/guides/pull/110) for details.

Hmph. You probably have MANY uses of `.property()` sprinkled throughout your codebase. How can you encourage developers to not use them anymore, and to refactor them as they run access them?

I think the answer is to have robots do the encouragement:

![lintreview](/images/ember-deprecations.png)

Using [`jscs-ember-deprecations`](https://www.npmjs.com/package/jscs-ember-deprecations) in conjunction with [lint-review](https://github.com/markstory/lint-review), you can opt a code base out of a set of features that you want to discourage. As seen in the above screenshot, developers will be notified if they introduce or modify a lint of code that uses a deprecated feature.

If you don't have a lint review / code review process, you can run the tool manually:

{% highlight bash %}
bash$ ./node_modules/jscs/bin/jscs --config .jscsrc app/
ObjectController is deprecated in Ember 1.11 at app/controllers/example-controller.js :
     4 |import i18nInitializer from 'app/initializers/i18n';
     5 |
     6 |var Controller = Ember.ObjectController.extend(BulkActionsMixin);
-----------------------------------^
     7 |
{% endhighlight %}

I believe that over time an application can be *gradually* upgraded in place, instead of needing to stop working on features for a week or two to upgrade Ember.

See the [docs](https://github.com/minichate/jscs-ember-deprecations/blob/master/README.md) for more details. If you like what you see, maybe you can contribute additional rules?