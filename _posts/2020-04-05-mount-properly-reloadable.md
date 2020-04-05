---
title: "The importance of being reloadable, properly"
categories: [blog, software engineering]
tags: [clojure, reloadable workflow, mount, tools.namespace]
---

If you write Clojure code, you probably heard of the 
[reloaded workflow](http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded).
Also, there's a big chance you handle state management with tools 
such as `mount` or `component`. In this blog post, we'll discuss one
edge case of integrating `mount` library with `tools.namespace`, namely
when applied in projects containing multiple applications.

![butterfly](/blog/assets/butterfly.jpg)

## Sample Clojure project

Let's imagine a Clojure project that contains two applications: a web service
and some worker. The project depends on `mount` version `0.1.16` to handle
states.

```
clj-sample
|__dev
|     user.clj
|__src
   |__clj_sample
         core.clj
         db.clj
         rest.clj
         worker.clj
``` 

`db.clj` encapsulates database clients.

```clojure
(ns clj-sample.db
  (:require [mount.core :refer [defstate]]))

(defstate elasticsearch
          :start (println "Starting connection: elasticsearch")
          :stop (println "Stopping connection: elasticsearch"))

(defstate mongodb
          :start (println "Starting connection: mongodb")
          :stop (println "Stopping connection: mongodb"))

```

`rest.clj` is the entry-point for the web service. The web service 
depends on database connections managed by `db.clj`.

```clojure
(ns clj-sample.rest
  (:require [clj-sample.db :as db]
            [mount.core :refer [defstate]]))

(defstate app
          :start (println "Starting app")
          :stop (println "Stopping app"))
```

`worker.clj` represents an abstract worker process. In the example, it 
depends on a certain environment state and cannot be started.

```clojure
(ns clj-sample.worker
  (:require [mount.core :refer [defstate]]))

(defstate worker
          :start (throw (ex-info "Cannot start worker" {}))
          :stop (println "Stopping worker"))
```

Also, we have the `user` namespace used as the REPL entry point. It defines
two functions: `go` and `reset` to manage the state lifecycle. I factored them according to (the recommendations)[https://github.com/tolitius/mount/tree/5d992042e45449e4f13c030a5200834b4f8904a9#the-importance-of-being-reloadable]
of `mount` library. 

Note that the `mount` library relies on the compiler to define the mount order.
Hence, when we require `clj-sample.rest`, the contained states start 
after those in `clj-sample.db`.

```clojure
(ns user
  (:require [clojure.tools.namespace.repl :refer [refresh
                                                  set-refresh-dirs]]
            [mount.core :as mount]
            [mount.tools.graph :refer [states-with-deps]]
            [clj-sample.rest]))

(alter-var-root #'*warn-on-reflection* (constantly true))

(set-refresh-dirs "src")

(defn go []
  (assoc (mount/start) :ready true))

(defn reset []
  (mount/stop)
  (refresh :after 'user/go))
```

## Trying out the reloaded workflow

Now, let's launch a new REPL session and start the web service.

```clojure
(go)
Starting connection: elasticsearch
Starting connection: mongodb
Starting app
=>
{:started ["#'clj-sample.db/elasticsearch" "#'clj-sample.db/mongodb" "#'clj-sample.rest/app"],
 :ready true}
```

The mount order follows the namespace dependencies. Now, let's do a test `(reset)`.

```clojure
Stopping app
Stopping connection: mongodb
Stopping connection: elasticsearch
:reloading (clj-sample.db clj-sample.rest clj-sample.core clj-sample.worker)
Starting connection: elasticsearch
Starting connection: mongodb
Starting app
Execution error (ExceptionInfo) at clj-sample.worker/eval3523$fn (worker.clj:5).
Cannot start worker
```

And there it is, in the initial refresh, `tools.namespace` has scanned the `src` folder
and loaded one additional state defined by `worker.clj`. The consequent `mount/start` then
started this additional state, resulting in `Execution error (ExceptionInfo): Cannot start worker`.

## What can we do?

As of now, `mount` [does not support](https://github.com/tolitius/mount/tree/5d992042e45449e4f13c030a5200834b4f8904a9#dependencies) dependency
graphs. In a world of Spring, I would provide the container with a context, and
would be able to retrieve any bean along with it's dependencies. `mount`, on the other hand,
is perfectly simple software. Simple tools are often open, set fewer preconditions, and thus 
are more powerful. So, it takes more thinking to master `mount` then canonical dependency injection.

If our states are rather stable, we can explicitly tell `mount` which 
states to load.

```clojure
(def states ["#'clj-sample.db/elasticsearch"
             "#'clj-sample.db/mongodb"
             "#'clj-sample.rest/app"])

(defn go []
  (assoc (mount/start states) :ready true))
```

Sometimes excludes work better.

```clojure
(def states (mount/except ["#'clj-sample.rest/worker"]))
```

But in large codebases, there can be numerous states which continuously change.
In that case, we can remember the states that had been loaded before the 
first `refresh`. 

```clojure
(defonce states (->> (states-with-deps) (map :name)))
``` 

## And still...

And still, in the real world, a project can host a dozen of apps. 
All the solutions tie `user.clj` to the single application, i.e., the web service. 
How can we separate the REPL workflow by an app? Ideally, I want to start and stop everything 
related to `rest.clj` or `worker.clj` independently. 

At least for me, this question is still open. All ideas look clumsy. So, if you've already 
solved this problem, please [let me know](https://twitter.com/shapiydev).
