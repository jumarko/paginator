# Paginator

[![Clojars Project](https://img.shields.io/clojars/v/org.clojars.roklenarcic/paginator.svg)](https://clojars.org/org.clojars.roklenarcic/paginator)

A lot of APIs solve the problem with large lists by paging. A complex scenario where you want
to load all pages of one entity, then loading all pages of another type of entity, concurrently,
while also limiting max concurrency, is not an easy thing to code. This is why this library exists.

You interact with this library by importing `[org.clojars.roklenarcic.paginator :as p]`.

## Pagination state

This library uses a core async loop that takes paging state maps via input channel
and places these maps with items on the output channel.

You create a paging state for an entity by calling `p/paging-state` function:

```clojure
(p/paging-state :entity-type 1)
```

This creates: 

```clojure
(paging-state :customer 1)
=> {:id 1, :entity-type :customer, :pages 0, :items []}
```

Pages of items are concatenated into `:items` vector. 

**After the first request there will also be a key `:page-cursor`. If page-cursor is present but `nil` the
paging is considered finished and the state map will be placed into the output channel.**

## Paginate

The pagination is done via `p/paginate*!`. It takes an `engine` and `params`. Engine will be
described later. Params can be anything and it will be passed to your page loading function.

```clojure
(let [{:keys [out in]} (p/paginate*! engine params)]
  (async/onto-chan! in paging-states)
  ...)
```

**If input channel is not closed the paginate will never terminate.** Out channel will
get paging states with `:items` and potentially `:exception`, an exception that happened that fetch call.

The reason why `in` and `out` channels are used is because you can use them to chain together
multiple pagination processes.

But most of the time you'll want to use a collection for input and the output to be collected
into a collection.

To do that, use one of the convenience wrappers:

### paginate!

This wrapper adds to base `paginate*!`:
- takes a coll of entity IDs and creates page states for you
- takes a coll and pushes page states into the input channel and closes it
- blockingly reads the result from the output channel into a collection of page states
- throws an exception if a page state has an exception

```clojure
(let [states (paginate! engine {} [[:projects-by-person 1] 
                                   [:projects-by-person 2] 
                                   [:projects-by-person 3]])]
  states)
```

### paginate-coll!

This wrapper does even more than `paginate!` by making more assumptions about what you have and
what you want. Besides what `paginate!` does, this also:

- takes a single entity type and a coll of IDs (i.e. assumes all your entities are of the same type)
- it guarantees that the order of outputs matches the order of input IDs
- it will not return any new entities returned only the initial ones
- it will return a vector of vectors, where the inner vector is the resulting `:items` from each final paging state

### paginate-one!

This wrapper is like `paginate-coll` but it operates on a single paging state and returns a single vector of results.

## Pagination engine

An engine is a map that describes aspects of your paging process.

```clojure
(p/engine result-parser get-items-fn async-fn)
```

Only the result-parser is the required parameter.

### Result parser

Result parser is an object of `org.clojars.roklenarcic.paginator.protocols/ResultParser`.

There's convenience wrappers `p/result-parser` and `p/result-parser1` that you will want to use.

Result parser specifies how the responses generated by your get items function parse into 3 things:
- map of `[entity-type id]` to cursor to next page for each entity being paged
- map of `[entity-type id]` to coll of items for each entity being paged
- any new entities that we want to introduce to paging process (or paging process result)

### Get Items Fn

This defaults to `p/get-items` multimethod, but it can be any method that takes
a `params` map and `paging-states`, a coll of paging states, a batch.

## async-fn

This optional parameter is a function that takes a no-arg function and runs it asynchronously.

Defaults to `clojure.core/future-call`. You can use this to provide an alternative threadpool.

## with-batcher

Sets batching configuration for the engine. As paging states are queued up for processing they are
arranged into batches. The `get-items-fn` will be called with one such batch each time.

Each paging state queued up for (more) processing will be evaluated with `batch-fn` which should return
the batch ID for the paging stage. It will be added to that batch. 

If batch reached maximum size for batches it will enter processing. If 100ms has passed since
any event in the system one of the unfinished batches will enter processing if there's unused
concurrency in the engine. If a sorted batcher is chosen then the unfinished batch that will enter
processing will be the one with the lowest ID. In case of a sorted batcher, the IDs must be comparable.

The parameters are:
- sorted
- max-items (defaults to 1)
- batch-fn (defaults to :entity-type)

## with-result-buf

Sets a different buffer size on output channel, the default being 100.

## with-concurrency

Sets maximum concurrency for paged get-items calls. Engine will not queue additional
`get-items-fn` calls if maximum concurrency is reached. Default is 1.

## with-items-fn

Sets `get-items-fn` for the engine.

## Linear pagination with next page links

A common scenario is to request a list from an API and you get in the response some kind of token,
to use to request the next page. This usually precludes any parallelization.

E.g. https://docs.microsoft.com/en-us/rest/api/azure/devops/core/projects/list?view=azure-devops-rest-6.1

In the result you'll get `x-ms-continuationtoken` header. Provide it when calling the list endpoint in
`continuationToken` query parameter.

This is a fairly common scenario in paging results.

If `p` is required of `org.clojars.roklenarcic.paginator`.

```clojure


(defmethod p/get-items ::projects
  [{:keys [auth-token]} states]
  (get-projects auth-token (-> states first :page-cursor)))

(p/paginate-one!
  (p/engine (p/result-parser1
              (comp :items :body)
              #(get-in % [:headers "x-ms-continuationtoken"])))
  {:auth-token "MY AUTH"} 
  ::projects nil)

```

## Offset + count

A common pattern is that API endpoint takes an `offset` parameter (also called `skip` sometimes)
and a `count` (or `page-size`) parameter. It then returns `count` items from `offset` onward.

The offset itself can be a number of items or a number of pages. It doesn't make a difference to a paging algorithm. 

```clojure

(defmethod p/get-items ::projects-offset
  [{:keys [auth-token]} states]
  (get-projects-with-offset auth-token (-> states first :page-cursor)))

(p/paginate-one!
   (p/engine (p/result-parser1
               (comp :items :body)
               (comp :offset :body)))
   {:auth-token "MY AUTH"} ::projects-offset nil)
```

## Swapping control to call-site

In the previous examples, the operation done was selected by `:entity-type` in the
paging states and the implementations provided by defining the multimethod.

We can change this to specify a function at the spot where we call
paginate. We can provide a fn to `engine` constructor to use instead of get-items 
multimethod.

If we have a generic request function which varies to operation based on parameters we
might want to specify them at the callsite.

Imagine you're working with Bitbucket REST API. Some API calls will
return a body with two keys `:values` and `:next` as a way of pagination. And there are many such
functions.

```clojure
(defn api-call [auth-token method url params]
  ...)

(def paged-api
  (p/engine
    (p/result-parser1 (comp :values :body) (comp :next :body))
    (fn [[auth-token method url params] paging-states]
      (if-let [cursor (-> paging-states first :page-cursor)]
        (api-call auth-token :get cursor {})
        (api-call auth-token method url params)))))

(p/paginate-one! paged-api ["auth-token" :get "/projects" {}] ::projects2 nil)
```
## Emitting new entities or paging states during the process

Imagine you paginate accounts, and then you want to paginate users, and then paginate their emails.
You want to generate new paging states for accounts and return them into process. But for emails you want to
return them into the results stream. **This won't work with `paginate-one!` and `paginate-coll!` because
those assume that the input and output entities are the same**.

Here's an example from tests:

```clojure
;; mock data
(def accounts
  [{:account-name "A"} {:account-name "B"} {:account-name "C"} {:account-name "D"} {:account-name "E"} {:account-name "F"}])

(def repositories
  {"A" [{:repo-name "A/1"} {:repo-name "A/2"} {:repo-name "A/3"} {:repo-name "A/4"} {:repo-name "A/5"}]
   "B" []
   "C" [{:repo-name "C/1"}]
   "D" [{:repo-name "D/1"} {:repo-name "D/2"} {:repo-name "D/3"} {:repo-name "D/4"} {:repo-name "D/5"} {:repo-name "D/6"}]
   "E" [{:repo-name "E/1"} {:repo-name "E/2"}]
   "F" [{:repo-name "F/1"} {:repo-name "F/2"} {:repo-name "F/3"}]})

(defn get-from-vector [v cursor]
  (let [p (or cursor 0)]
    {:body {:items (take 2 (drop p v))
            :offset (when (< (+ 2 p) (count v))
                      (+ 2 p))}}))

;; mock response functions
(defmethod p/get-items
  ::accounts
  [params [{:keys [page-cursor]}]]
  (assoc (get-from-vector accounts page-cursor)
    :resp-type ::accounts))

(defmethod p/get-items
  ::account-repos
  [params [{:keys [page-cursor id]}]]
  (get-from-vector (repositories id) page-cursor))

(def e
  (p/engine
    (p/result-parser1
      (comp :items :body)
      (comp :offset :body)
      (fn [resp]
        (case (:resp-type resp)
          ::accounts (map #(p/paging-state ::account-repos (:account-name %)) (-> resp :body :items))
          nil)))))

(p/paginate! e {} [[::accounts nil]])
=>
[{:id "B", :entity-type ::account-repos, :pages 1, :items [], :page-cursor nil}
 {:id nil,
  :entity-type ::accounts,
  :pages 3,
  :items [{:account-name "A"}
          {:account-name "B"}
          {:account-name "C"}
          {:account-name "D"}
          {:account-name "E"}
          {:account-name "F"}],
  :page-cursor nil}
 {:id "C",
  :entity-type ::account-repos,
  :pages 1,
  :items [{:repo-name "C/1"}],
  :page-cursor nil}
 {:id "E",
  :entity-type ::account-repos,
  :pages 1,
  :items [{:repo-name "E/1"} {:repo-name "E/2"}],
  :page-cursor nil}
 {:id "A",
  :entity-type ::account-repos,
  :pages 3,
  :items [{:repo-name "A/1"} {:repo-name "A/2"} {:repo-name "A/3"} {:repo-name "A/4"} {:repo-name "A/5"}],
  :page-cursor nil}
 {:id "F",
  :entity-type ::account-repos,
  :pages 2,
  :items [{:repo-name "F/1"} {:repo-name "F/2"} {:repo-name "F/3"}],
  :page-cursor nil}
 {:id "D",
  :entity-type ::account-repos,
  :pages 3,
  :items [{:repo-name "D/1"}
          {:repo-name "D/2"}
          {:repo-name "D/3"}
          {:repo-name "D/4"}
          {:repo-name "D/5"}
          {:repo-name "D/6"}],
  :page-cursor nil}]

```

## A more complex example

[Listing branches via GitLab GraphQL API](doc/branches-example.md)

# License

Copyright © 2021 Rok Lenarčič

Licensed under the term of the Eclipse Public License - v 2.0, see LICENSE.