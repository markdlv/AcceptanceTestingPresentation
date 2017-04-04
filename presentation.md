## Acceptance testing with ember-cli-mirage

<br>

Mark DeLaVergne

HSLIDE

## Overview

* Acceptance testing intro
* Ember-cli-mirage intro
* Tips & tricks
* Take-aways / opinions

HSLIDE

## Acceptance testing?

NOTE: Before getting into mirage, let's take a step back and look at the big picture with acceptance testing in Ember

VSLIDE

### In Ember

<p class="fragment fade-in">
    Three types of tests in ember-cli:<br>
    unit, integration, acceptance
</p>
<p class="fragment fade-in">Written differently; serve different purposes</p>
<p class="fragment fade-in">[Unification RFC](https://github.com/rwjblue/rfcs/blob/42/text/0000-grand-testing-unification.md)</p>

VSLIDE

### Motivation

<p class="fragment fade-in">Verify functionality, "end to end"</p>
<p class="fragment fade-in">Ability to refactor with confidence</p>
<p class="fragment fade-in">Cut down on server resources</p>

NOTE: 3rd point is especially true if you already are heavy in automation tests which test the system truly end-to-end.  We'll touch on all 3 of these at the end of the presentation.

HSLIDE

## Dummy app

If your project is "the parent app", <br>
no need to worry about this

VSLIDE

### If your project is an addon:

Step 1 is to get your "dummy app" functional

<p class="fragment fade-in">Used for both development as well as acceptance tests<p>
<p class="fragment fade-in">So, once functional, then we're ready for testing?</p>

NOTE: If you've never run your dummy app, this can take some time.  You'll run into issues you weren't aware of, like how your arrays were getting automatically wrapped in Ember.A, etc.

VSLIDE

### Depends

<p class="fragment fade-in">
If you want requests to go through to the API:<br>
```ember g http-proxy```
</p>
<br>
<p class="fragment fade-in">
To mock the requests:<br>
```ember g http-mock```
</p>

NOTE: Using http-proxy (or the general concept of having the requests actually hit your back-end) is going to depend on your project's needs I think.  For example, if your project doesn't have any other automation testing that covers true end-to-end (i.e. beyond the UI) tests, then having acceptance tests hit your APIs makes a lot of sense.  If, on the other hand, you have other automation coverage - I think it makes sense to have acceptance tests cover YOUR project, end to end.

VSLIDE

### Define acceptance

What is "end to end"?

TODO: images showing the different "end-to-end" concepts
* everything
* entire ember app
* only your addon/engine

<p class="fragment fade-in">
For now, let's say you only want to test your code<br>
Therefore, you start using http-mock
<p>

VSLIDE

### But wait

![http-mock is only for ember s](/img/mocksAreOnlyForDev.png)

<p class="fragment fade-in">From here, you could use pretender for the tests..</p>
<p class="fragment fade-in">But http-mock uses CommonJS and pretender uses ES6 modules, so sharing mocks would be cumbersome</p>

NOTE: At this point, hopefully you're drawing the conclusion that, long-term, http-mock isn't very helpful.  Useful for a quick app you're throwing together and you just need to fake a server.

HSLIDE

## Enter ember-cli-mirage

![Mirage feels like Ember Data](/img/mirageSolvesTheseProblems.png)

NOTE: Mirage allows for routes, a *client-side* database, factories and fixtures, and serializers.  All of these things might remind you of Ember Data.  If you just cringed a little bit when I said the words "Ember Data", I understand.  But I will say, when it comes to doing the "prepare" portion of tests, I've actually found Ember Data flow to be a pretty nice fit.

HSLIDE

### Handlers

```
// mirage/config.js
export default function() {
  this.namespace = 'api/v2';

  this.get('/authors', () => {
    return {
      authors: [
        {id: 1, name: 'Zelda'},
        {id: 2, name: 'Link'},
        {id: 3, name: 'Epona'},
      ]
    };
  });
}
```

VSLIDE

Great starting point to get your dummy app working.

<p class="fragment fade-in">
But the ember-cli-mirage docs are spot on:
![Need dynamic data](/img/dynamicData.png)
<p>

NOTE: For point #2: this is where I'm saying the Ember Data flow is quite nice.  Instead of duplicating the possibly-verbose format of an API across all of your mocks, use a model to hold the content you care about and a serializer to format it.

HSLIDE

### Models

```
ember g mirage-model session
```

Produces:
```
// app/mirage-models/session.js
export { default } from 'addon-name/mirage-models/session';

// tests/dummy/mirage/models/session.js
import { Model } from 'ember-cli-mirage';

export default Model.extend({});
```

NOTE: As you can see, for an addon these are only going to apply to dummy app usage: `ember s` and acceptance testing.

HSLIDE

### Factories

```
ember g mirage-factory session
```

```
// tests/dummy/mirage/factories/session.js
import { Factory, faker } from 'ember-cli-mirage';

export default Factory.extend({
    person: {
        _id: faker.random.number(),
        guid: faker.random.uuid()
    },
    user: {
        email: faker.internet.exampleEmail()
    },
    org: {
        general: {
            shortName: [ { value: faker.company.bsBuzz() } ],
            name: [ { value: faker.company.companyName() } ]
        },
        guid: faker.random.uuid()
    }
});
```

[Faker demo](https://cdn.rawgit.com/Marak/faker.js/master/examples/browser/index.html)

NOTE: Note that all the keys I care about aren't nested.  This will make it nice to override them later.

HSLIDE

### Serializers

```
ember g mirage-serializer session
```

```
// tests/dummy/mirage/serializers/session.js
import { Serializer } from 'ember-cli-mirage';

export default Serializer.extend({
    keyForModel() {
        return 'res';
    }
});
```

HSLIDE

### Associations

```
// mirage/models/author.js
import { Model, hasMany } from 'ember-cli-mirage';

export default Model.extend({
    blogPosts: hasMany()
});

// mirage/models/blog-post.js
import { Model, belongsTo } from 'ember-cli-mirage';

export default Model.extend({
    author: belongsTo()
});
```

VSLIDE

#### Response
```
{
  author: {
    id: 1,
    'first-name': 'Zelda'
  },
  'blog-posts': [
    {id: 1, 'author-id': 1, title: 'Lorem ipsum'},
    ...
  ]
}
```

NOTE: If your APIs don't behave like this (e.g. if blog-posts would be within the author object), my tendency is to build up the parent model to include those embedded objects - like we saw in the session example.

HSLIDE

## Tips & Tricks

NOTE: Now I'm going to run through some things that I wish I would've realized earlier in the process.  Some of these are right in the docs, others are things that took me a while to figure out.  But just know these are cherry-picked from the docs or experience.

VSLIDE

### How to mount an engine

```
// tests/dummy/app/router.js
Router.map(function () {
    this.mount('ember-engine-awesome', { as: 'awesome', path: 'admin/awesomeness' });
});

// tests/dummy/app/app.js
engines: {
    emberEngineAwesome: {
        dependencies: {
            services: []
        }
    }
}
```

VSLIDE

### How to mock engine dependencies

```
// tests/dummy/app/app.js
engines: {
    emberEngineAwesome: {
        dependencies: {
            services: [
                'dep1',
                'dep2'
            ]
        }
    }
}

// tests/dummy/app/resolver.js
let dep1Mock = {},
    dep2Mock = {};  // ... actually mock the services
let mockedServices = {
    'service:dep1' : dep1Mock,
    'service:dep2' : dep2Mock
};

export default Resolver.extend({
    resolveOther: function(parsedName) {
        let mockedService = mockedServices[parsedName.fullName];

        if (mockedService) {
            return mockedService;
        } else {
            return this._super(parsedName);
        }
    }
});
```

VSLIDE

### How to seed your database

```
/*
    Seed your development database using your factories.
    This data will not be loaded in your tests.

    Make sure to define a factory for each model you want to create.
*/

server.create('author');
server.create('author', { firstName: 'Mark' });
server.createList('post', 10);
server.createList('comments', 10, { comment: 'I love it!' });

server.create('account');
```

Override factory values via final argument in `create()` or `createList()`

VSLIDE

### Where to seed the database

![Important files](/img/importantFiles.png)

VSLIDE

### Dynamic paths and query params

```
this.get('/authors/:id', (schema, request) => {
    var id = request.params.id;
    var fields = request.queryParams.fields;

    return schema.authors.find(id)
    .then((author) => {
        // Filter fields
        return author;
    });
});
```

VSLIDE

### Shorthands

```
// The route handler
this.get('/authors', (schema, request) => {
    return schema.authors.all();
});

// can be written as:
this.get('/authors');
```

VSLIDE

### Common scenario for singular API

```
this.get(`${apiV2Prefix}/session`, function(schema) {
    return schema.sessions.first();
});
```

VSLIDE

### Dynamic status codes and HTTP headers

```
// app/mirage/config.js
import { Response } from 'ember-cli-mirage';

export default function() {
    this.post('/authors', function(schema, request) {
        let attrs = JSON.parse(request.requestBody).author;

        if (attrs.name) {
            return schema.authors.create(attrs);
        } else {
            return new Response(400, {some: 'header'}, {errors: ['name cannot be blank']});
        }
    });
}
```

VSLIDE

### Fully qualified URLs

```
this.get('https://api.github.com/users/samselikoff/events', () => {
    return [];
});
```

VSLIDE

### Powerful serializers

The ones I used in the first round of testing:
* [root](http://www.ember-cli-mirage.com/docs/v0.3.x/serializers/#root)
* [keyForModel(modelName)](http://www.ember-cli-mirage.com/docs/v0.3.x/serializers/#keyformodelmodelname)
* [keyForCollection(modelName)](http://www.ember-cli-mirage.com/docs/v0.3.x/serializers/#keyforcollectionmodelname)
* [keyForAttribute(attr)](http://www.ember-cli-mirage.com/docs/v0.3.x/serializers/#keyforattributeattr)
* [serialize(response, request)](http://www.ember-cli-mirage.com/docs/v0.3.x/serializers/#serializeresponse-request)

HSLIDE

## Take-aways / opinions

NOTE: Ok, final section here.  I want to leave you all with some thoughts on how much of a win this will be for each team and the company as a whole.

VSLIDE

### The more engine dependencies, <br>the less you can test

The more dependent on the parent app your project is, the less functionality you'll be able to verify

<p class="fragment fade-in">
If your engine is extremely dependent, you'll either get into "mocking the world" or these will feel more like wide-reaching integration tests
</p>

NOTE: If that’s the best we can do, that’s ok.  But what I’d recommend to avoid this issue is to ... get rid of the dependencies.  Instead, break out the code we have in web-directory into separate repos that can be consumed by all

VSLIDE

### Opinion: Verify the functionality

Not the implementation

<ul class="fragment fade-in">
<li>This is like picking an abstraction level - quite difficult - but similarly worth the time & effort</li>
<li>If you do it right, the test verifies the outcome, not individual requests etc.</li>
<li>At that point, you can do significant refactoring and the tests will continue to verify the outcome</li>
</p>

VSLIDE

### Opinion: Volume & happy-pathiness

TODO: pyramid

VSLIDE

### Preview: Stale mocks

By breaking the models out into mirage factories, serializers, etc., it’s hard to see at a glance what your ember app expects across the wire.

This is particularly a problem when attempting to mitigate whether or not mock objects have become stale in any way.

Sneak peak into MockValidator idea

## Thanks!

Questions?