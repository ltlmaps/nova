# Documentation

## Table of Contents

- [Linking collections](#linking-collections)
- [Query-ing](#query-ing)
- [Relational Filtering and Sorting](#relational-filtering-and-sorting)
- [Dynamic Filters](#dynamic-filters)
- [Reducers](#reducers)
- [Aliased Collections](#aliased-collections)
- [GraphQL Integration](#graphql-integration)
- [Limitations](#limitations)
- [Hypernova](#hypernova)

## Linking collections

Collections are linked through `addLinks(collection, linkingInformation)`, collections are instances of `mongodb.Collection`, if you are using, for example `mongoose` or `Meteor` you can access it like this:

```typescript
// mongoose
Model.collection;

// Meteor
Collection.rawCollection();
```

```typescript
import { addLinks } from "@kaviar/nova";

addLinks(Patients, {
  medicalProfile: {
    collection: () => MedicalProfiles,
    // Where to read the linking information
    field: "medicalProfileId",
    // Whether it's an array of ids, or just a single id, in our case
    // By default it's false.
    many: false
  }
});
```

This would allow us to run the following query:

```typescript
import { query } from "@kaviar/nova";

const results = await query(Patients, {
  aField: 1,
  someOtherField: 1,
  someNestedField: {
    someValue: 1,
  }
  // This is the actual link
  medicalProfile: {
    // This is a field that belongs inside the MedicalProfile collection document
    bloodPresure: 1,
  }
}).fetch();

// You can also run .fetchOne() instead of .fetch() if you expect a single result
```

When we talk about inversed links, we mean that we want to query `MedicalProfiles` and go to `Patients`, to do that we have to setup an inversed link:

```typescript
addLinks(MedicalProfiles, {
  patient: {
    collection: () => Patients,

    // The actual link name, that was configured from Patients
    inversedBy: "medicalProfile"
  }
});
```

After we do this, we can easily do:

```typescript
import { query } from "@kaviar/nova";

const medicalProfile = await query(MedicalProfiles, {
  patient: {
    name: 1
  }
}).fetchOne();
```

However, this time you will notice that `medicalProfile.patient` is actually an array. And that's because it doesn't know it's unique, because this strategy can also be applied to having many. Imagine that if you want to link `Comments` with `Post` via a `postId` stored in the `Comments` collection, you would do the same type of linking.

In our case, the solution is to add `unique: true` when defining the `medicalProfile` link inside `Patients` collection:

```typescript
addLinks(Patients, {
  medicalProfile: {
    collection: () => MedicalProfiles,
    field: "medicalProfileId",
    unique: true
  }
});
```

### Linking Shortcuts

```typescript
import { oneToOne, oneToMany, manyToOne, manyToMany } from "@kaviar/nova";

// The link is stored in the first mentioned collection
manyToOne(Comments, Post, {
  linkName: "post",
  inversedLinkName: "comments"
  // field will be `postId`
});

oneToOne(Users, GameProfile, {
  linkName: "gameProfile",
  inversedLinkName: "user"
  // field will be `gameProfileId`
});

oneToMany(Posts, PostWarnings, {
  linkName: "postWarnings",
  inversedLinkName: "post" // single
  // field will be `postWarningsIds`
});

manyToMany(Posts, Tags, {
  linkName: "tags",
  inversedLinkName: "posts"
  // field will be `tagsIds`
});
```

If you want custom configuration, use the `addLinks()` functions.

Notes:

- You can go as deeply as you want with your queries
- Field storage can be stored in a nested field. Specify `field: "profile.medicalProfileId"`
- Indexes are automatically set. If you want custom indexes use `index: false` when defining the direct link.

## Query-ing

We showed a little bit of how we can query stuff, but we need to dive a little bit deeper, so let's explore together how we can filter, sort, and paginate our query.

We introduce a special `$` field that isn't really a field, it's more like a meta-ish way to describe the current node. Let's explore the `filters` and `options` parameters of this special "field".

```typescript
import { query, oneToMany } from "@kaviar/nova";

manyToOne(Comments, Posts, {
  linkName: "post",
  inversedLinkName: "comments"
});

query(Posts, {
  $: {
    // MongoDB Filters
    filters: {
      isPublished: true
    },

    // MongoDB Options
    options: {
      sort: {
        createdAt: -1
      },
      limit: 100,
      skip: 0
    }
  },
  title: 1,
  comments: {
    $: {
      // You can also use it for your nested collections
      // This will only apply to the comments for the post
      filters: {
        isApproved: true
      },
      options: {
        sort: {
          createdAt: -1
        }
      }
    },
    text: 1
  }
});
```

## Relational Filtering and Sorting

While working with MongoDB, another pain-point was what we call `relational filtering`, which simply means, I want to get all employees that belong in a company that is verified. Then what if I want to paginate them?

All of these problems are currently solved, and we belive the API is far superior than SQL in terms of clarity.

```typescript
import { oneToMany, query, lookup } from "@kaviar/nova";

manyToOne(Employees, Companies, {
  linkName: "company",
  inversedLinkName: "employees"
});

query(Employees, {
  $: {
    // This allows us to hook into the pipeline and filter the employee result
    pipeline: [
      // This performs the link from Employees to "company"
      // You don't have to worry about how it's linked, you will use your natural language
      lookup(Employees, "company"),
      {
        $match: {
          "company.verified": true
        }
      }
    ],
    options: {
      limit: 10,
      skip: 10
    }
  },
  firstName: 1
});
```

What did just happen?

- `pipeline: []` option allows us to hook into the aggregation pipeline used for fetching the employees
- `lookup()` function creates a smart \$lookup aggregator and projects the result as the link name

Why not take it a little bit further? What if we want all the employees from companies that have at least 5 departments? This implies that we need to go deeper into the pipeline:

```typescript
const result = await query(Employees, {
  $: {
    pipeline: [
      lookup(Employees, "company", {
        // Here's the nice part!
        pipeline: [
          lookup(Companies, "departments"),
          // I'm expanding the "company" to contain a "departmentsCount" field
          {
            $addFields: {
              departmentsCount: { $size: "$departments" }
            }
          }
        ]
      }),
      // I'm performing this match at user because this is where we need to decide whether the user will exist or not in the result set
      {
        $match: {
          "company.departmentsCount": {
            $gte: 2
          }
        }
      }
    ]
  },
  _id: 1,
  companyId: 1
}).fetch();
```

Pipeline will work in the same manner on nested collections:

```typescript
query(Companies, {
  employees: {
    $: {
      pipeline: [
        // Some logic to include or exclude the employee
      ]
    }
  }
});
```

## Dynamic Filters

What if for example, I want the nested collection filters/options to perform differently based on the parent? To do so, you transform \$ into a `function()` that passes the parent element.

```js
query(Users, {
  comments: {
    $(user) {
      return {
        filters: {},
        options: {},
        pipeline: {}
      };
    },
    author: {
      name: 1
    }
  }
});
```

This comes with a penalty cost. Because the filters depend on the parent, we will have to query the database for each `user`. However, if you go deeper, when extracting `authors` it will do it in one query.

## Reducers

Reducers are a way to expand the request query and compute the values, then get you a cleaned version. Imagine them as virtual, on-demand computed query fields:

```typescript
import { addReducers } from "@kaviar/nova";

addReducers(Users, {
  fullName: {
    dependency: {
      // Here we define what it needs to resolve
      // Yes, you can specify here links, fields, nested fields, and other reducers as well
      profile: {
        firstName: 1,
        lastName: 1
      }
    },
    // Reducers can also work with parameters
    async reduce(obj, params) {
      let fullName = `${obj.profile.firstName} ${obj.profile.lastName}`;
      if (params.withPrefix) {
        fullName = `Mr ${fullName}`;
      }

      return fullName;
    }
  }
});
```

```typescript
query(Users, {
  fullName: 1
});

query(Users, {
  fullName: {
    // Or with params
    $: {
      withPrefix: true
    }
  }
});
```

Reducers have the power to expand the pipeline, for example, if you want to get all the posts and their comments count in one request, you have 2 ways:

1. You create a reducer that performs a count for each
2. You create a reducer that extends the pipeline to give you a `commentsCount` field

```typescript
addReducers(Posts, {
  commentsCount: {
    dependency: { _id: 1 },
    pipeline: [
      lookup(Posts, "comments"),
      {
        $addFields: {
          commentsCount: { $size: "$comments" }
        }
      }
    ]
  }
});
```

Notes:

- Do not specify nested fields, use instead: `profile.name: 1` => `profile: { name: 1 }`
- Reducers can use other links and other reducers naturally.
- Just be careful when extending the pipeline it may have unexpected results.
- We recommend that you initially do not focus on performance, rather on code-clarity

## Aliased Collections

Sometimes you may find useful to fetch the same link but in different contexts, for example you want to get a Company with the last 3 invoices and with overdue invoices without much hassle:

```typescript
manyToOne(Invoices, Company, {
  linkName: "company",
  inversedLinkName: "invoices"
});

query(Company, {
  lastInvoices: {
    $alias: "invoices",
    $: {
      options: {
        sort: { createdAt: -1 },
        limit: 3
      }
    },
    number: 1,
  },
  overdueInvoices: {
    $alias: "invoices",
    $: {
      filters: {
        isPaid: false,
        paymentDue: { $lt: Date.now() }
      }
    }
    number: 1,
  }
});
```

We currently don't offer a built-in way to handle these aliases, like an `addAlias()` function, we don't do that because in this realm it will depend a lot of your business logic and query-ing strategies. You could compose them something like this:

```js
const overdueInvoicesAlias = {
  $alias: "invoices",
  $: { filters: { '...' } }
};

query(Company, {
  overdueInvoices: {
    ...overdueInvoicesAlias,
    number: 1,
  }
})
```

## GraphQL Integration

The integration removes the necessity of writing custom resolvers to fetch related data. Everything is computed efficiently.

```typescript
import { query } from "@kaviar/nova";

// Define your query resolvers
const Query = {
  users(_, args, context, info) {
    return query
      .graphql(A, info, {
        // Manipulate the transformed body
        // Here, you would be able to remove certain fields, or manipulate the Nova Query body
        // This happens before creating the nodes, so it gives you a chance to do whatever you wish
        embody(body, getArguments) {
          body.$ = {
            // Set some options here
          };

          // You can get the arguments of any path
          const commentsArgument = getArguments("comments");

          // Comments author's arguments
          const authorArguments = getArguments("comments.author");
        },

        // Intersection is what they have in common
        // This is the best way to secure your graph. By stating explicitly what you allow query-ing
        intersect: {},

        // This is useful to avoid a nested attack
        // Depth applies to deeply nested fields, not only collection links
        maxDepth: 10,

        // Automatically enforces a maximum number of results
        maxLimit: 10,

        // Simply removes from the graph what fields it won't allow
        // Can work with deep strings like 'comments.author'
        deny: [] // String[]
      })
      .fetch();
  }
};
```

If you do however want your resolving to happen in GraphQL Resolvers not inside a reducer, we introduce a new concept, called `Expanders`.

Let's say you have a resolver at `User` level called `fullName` that uses `firstName` and `lastName`:

```js
import { addExpanders } from "@kaviar/nova";

addExpanders(Users, {
  // Full name will not appear in the result set
  fullName: {
    profile: {
      firstName: 1,
      lastName: 1
    }
  }
});

query(Users, {
  fullName: 1
}).fetchOne();
```

## Limitations

### Limit/Skip in the nested collections

Let's take this post for example, we want to find all posts, then we want the latest 5 comments from each.

Currently, we store `postId` inside the comments collection:

```js
query(Posts, {
  comments: {
    $: {
      options: {
        sort: { createdAt: -1 },
        limit: 5
      }
      author: {
        name: 1,
      }
    }
    name: 1,
  }
}
```

Hypernova is not able to retrieve all comments for all posts in one-go (because of limit/skip). Therefore it has to do it iteratively for each found post. However, hypernova works afterwards when we need to fetch the authors of comments. It will fetch all authors in one-go, and properly assemble it.

### Dynamic Filterings

When you are using a dynamic filter for your nested collections:

```js
query(Posts, {
  comments: {
    $(post) {
      return {
        filters: {
          // Some filters
        }
      }
    }
  }
}
```

Hypernova will be disabled, and it will run the query for fetching the comments for each individual Post. Because most likely your filters and options depend on it.

### Top-level fields for linking information

We allow storing link storages within nested objects such as:

```typescript
{
  profile: {
    paymentProfileId: []; // <- Like this
  }
}
```

```typescript
// WORKS
addLinks(Users, {
  collection: () => PaymentProfiles,
  field: "profile.paymentProfileId"
});
```

However, we currently can't work with fields in arrays of objects, or have array of objects connect to certain collections:

```js
// NOT POSSIBLE to link with a schema as such:
{
  users: [
    {userId: 'xxx', roles: 'ADMIN'},
    ...
  ]
}
```

## Hypernova

This is the crown jewl of Nova. It has been engineered for absolute performance. We had to name this whole process somehow, and we had to give it a bombastic name. Hypernova is the one that stuck.

To understand what we're talking about let's take this example of a query:

```js
query(Posts, {
  categories: {
    name: 1
  },
  author: {
    name: 1
  },
  comments: {
    $options: { limit: 10 },
    author: {
      name: 1
    }
  }
});
```

### Counting Queries

In a normal scenario, to retrieve this data graph we need to:

1. Fetch the posts
2. length(posts) x Fetch categories for each post
3. length(posts) x Fetch author for each post
4. length(posts) x Fetch comments for each post
5. length(posts) _ length(comments) _ Fetch author for each comment

Assuming we have:

- 10 posts
- 2 categories per post
- 1 author per post
- 10 comments per post
- 1 author per comment

We would have blasted the database with:

- Posts: 1
- Categories: 10
- Post authors: 10
- Post comments: 10
- Post comments authors: 10\*10

This means `131` database requests.

Ok, you can cache some stuff, maybe some authors collide, but in order to write a performant code,
you would have to write a bunch of non-reusable code.

But this is just a simple query, imagine something deeper nested. For Nova, it's a breeze.

**How many requests we do via Hypernova?**

- 1 for Posts
- 1 for all authors inside Posts
- 1 for all categories inside Posts
- 1 for all comments inside Posts
- 1 for all authors inside all comments

The number of database requests is predictable, because it represents the number of collection nodes inside the graph.
(If you use reducers that make use of links, take those into consideration as well)

It does this by aggregating filters at each level, fetching the data, and then it reassembles data to their
propper objects.

Not only it makes **5 requests** instead of 131, but it smartly re-uses categories and authors at each collection node,
meaning you will have much less bandwidth consumed.

Making it more efficient in terms of bandwidth than SQL or other relational databases.

Example:

```js
{
  posts: {
    categories: {
      name: 1;
    }
  }
}
```

Let's assume we have 100 posts, and the total number of categories is like 4. Hypernova does 2 requests to the database,
and fetches 100 posts, and 4 categories. If you would have used `JOIN` functionality in SQL, you would have received
the categories for each post.

## Go back to [Table of Contents](#table-of-contents)
