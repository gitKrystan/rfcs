- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

One paragraph explanation of the feature.

# Motivation

The point of services—and dependency injections in general—is precisely to make them easily swappable in tests (and/or other configuration/environments). However, when it comes to stubbing services in Ember tests, that is easier said than done.

This RFC proposes that we add the ability to add mock services to a `tests/services` folder, and that these services will automatically be used in place of the equivalent app services in tests. For example, `tests/services/my-service.js` would be used in place of `app/services/my-service.js` in tests.

Currently, in order to stub a service in tests, we need to repeat the `this.owner.register` incantation in each relevant module. This necessity leads to redundant test code and can cause unexpected test behavior. For example, let's say we use a `date` service, which we stub in tests to ensure that we are always using the same frozen timestamp. If we forget to stub the service for one module, we may introduce intermittent test failures that are date or time-dependent.

Often, we need to first `unregister` the service, lest we run into the error `Assertion Failed: Cannot re-register: 'service:my-service', as it has already been resolved.`

### A Simple Example

#### Current
```js
// tests/integration/components/location-map-test.js
import Service from '@ember/service';
// ...

const StubMapsService = Service.extend({
  getMapElement(location) {
    this.set('calledWithLocation', location);
    return document.createElement('div');
  }
});

module('Integration | Component | location map', function(hooks) {
  hooks.beforeEach(function() {
    this.owner.unregister('service:maps');
    this.owner.register('service:maps', StubMapsService);
    this.mapsService = this.owner.lookup('service:maps');
  });

  test('should call service with location', async function(assert) {
    this.set('myLocation', 'New York');
    await render(hbs`{{location-map location=myLocation}}`);
    assert.equal(this.mapsService.calledWithLocation, 'New York', 'should call service with New York');
  });
});
```

#### Proposed
```js
// tests/services/maps.js
import Service from '@ember/service';

export default Service.extend({
  getMapElement(location) {
    this.set('calledWithLocation', location);
    return document.createElement('div');
  }
});
```

```js
// tests/services/components/location-map-test.js
// ...

module('Integration | Component | location map', function(hooks) {
  hooks.beforeEach(function() {
    this.mapsService = this.owner.lookup('service:maps');
  });

  test('should call service with location', async function(assert) {
    this.set('myLocation', 'New York');
    await render(hbs`{{location-map location=myLocation}}`);
    assert.equal(this.mapsService.calledWithLocation, 'New York', 'should call service with New York');
  });
});
```

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?

# Drawbacks

Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

















```js
// app/services/date.js
export default Service.extend({
  timeOffset: 0,

  frozenNow: null,

  now: computed('frozenNow', function() {
    let frozenNow = this.frozenNow;
    if (frozenNow) { return frozenNow; }
    return Date.now();
  }).volatile(),

  adjustedNow: computed('now', 'timeOffset', function() {
    return this.now + this.timeOffset;
  }).volatile(),

  isFrozen: bool('frozenNow'),

  freeze(frozen) {
    if (typeof frozen === 'string') {
      frozen = Date.parse(frozen);
    } else if (frozen.constructor === Date) {
      frozen = frozen.getTime();
    }

    this.set('frozenNow', frozen);
  },

  unfreeze() {
    if (!this.frozenNow) { throw "time isn't frozen"; }
    this.set('frozenNow', null);
  },

  frozenNowDidChange: observer('frozenNow', function() {
    this.notifyPropertyChange('now');
    this.notifyPropertyChange('adjustedNow');
  })
});
```

```js
// in a test
that.owner.lookup('service:date').freeze('2016-12-18T11:24:12.000Z');
```

could become

```js
// app/services/date.js
export default Service.extend({
  timeOffset: 0,

  now: computed(function() {
    return Date.now();
  }).volatile(),

  adjustedNow: computed('now', 'timeOffset', function() {
    return this.now + this.timeOffset;
  }).volatile(),
});
```

```js
// tests/services/date.js
import DateService from 'app/services/date';

export default DateService.extend({
  now: computed(function() {
    return Date.parse('2016-12-18T11:24:12.000Z');
  })
});
```

and we would get the `adjustedNow` computed property for free.
