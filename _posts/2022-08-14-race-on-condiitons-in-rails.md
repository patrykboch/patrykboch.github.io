---
layout: post
title: Race (on) condition(s) in Rails
description: How to solve race condition problems in rails
tags: [ruby, rails, race condition, optimistic lock, pesimistic lock]
comments: true
published: true
---

Some argue that Ruby on Rails applications cannot scale. That is a common response given by candidates in job interviews when asked about the disadvantages of the framework. I don't want to choose a side. I might respond, "it depends" - as always. Anyhow, if you want to prepare your app for receiving numerous requests at once, you need to consider the _race condition_ issue which is liable to cause us troubles.

I don't want to delve into the definition of "race condition" as I'm sure my readers understand what it is because it is fundamental to software engineering. I'd like to concentrate on ways to get rid of this. If you're unfamiliar and you want to discover more about the term's origins, go [here](https://devopedia.org/race-condition-software). Anyways, one line will adequately describe the problem.

> When your app allows multiple requests to interact with the same records, and one request overrides changes made by another without taking them into account.

For instance, if a large number of requests modify the same record at the same time in multi-users scenario.

As my final point in this introduction, I want to emphasize that a database must ensure integrity of data, particularly when performing concurrent operations. Integrity introduces the ACID-compliant idea of "locking" into the picture. Because of this, both optimistic and pessimistic locking are considered as means of addressing race condition issues.

## Good old _optimistic locking_

Optimistic locking is when the version attribute maintained in a database column is taken into account. It's the implicit technique of dodging the issue IMO. If we intend to use it, the special _lock_version_ column needs to be added and Active Record, which is Rails' default component, takes care of minimizing data conflicts.

```ruby
add_column :table_name, :lock_version, :integer
```

Each time a record is updated, the `lock_version` entity is increased and the locking features make sure that records will only allow the last one stored to throw a `StaleObjectError` if there is a parallel update of any kind. Then, we can attempt updating the data again.

<script src="https://gist.github.com/patrykboch/99296a4395b362ed3dbd279adedf02b1.js"></script>

Rails are adaptable; if you want to use your own table with a name that differs from _lock_version_ you can do so by specyfing in your model:

```ruby
self.locking_column = :custom_lock_version
```

Why is this type of dealing with race condition regarded as optimistic? Because it is assumed that database conflicts occur less frequently. Optimistic locking performs by simply comparing the "version" column value. As a result, it does not represent a true database lock.

## Good old _pessimistic locking_

Pessimistic locking, on the other hand, is a technique that counts on more frequent database conflicts. Since it offers an exclusive lock on the record, it is regarded as more explicit. When a single request modifies a record, it locks it until a transaction is completed. For this purpose a built-in methods `with_lock` and `#lock!` are used. Both operate similarly to each other. The main difference is that the `#lock!` needs to be used within an Active Record's transaction since it is unlocked again when the surrounding transaction is finished; sadly, using it outside the transaction block is ineffective. The `with_lock` method initiates a database transaction by itself. Anyways, all methods prevent others from reading or writing a record until the transaction is completed.

<script src="https://gist.github.com/patrykboch/0ec6ae5943fc43331771a34ac1df9fe2.js"></script>

[See the `with_lock` API reference](https://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html).

## Summing up good old techniques

Both locking methods are regarded as useful; however, the cost of a transaction retry must be considered when selecting a suitable locking scheme. It is determined by app requirements and business logic. Let's summarize these two approaches briefly.
<br>
<br>

| Optimistic locking                                                | Pessimistic locking                                                         |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Locks a record once changes are comitted to db                    | Locks record once it is edited                                              |
| Considers data conflicts less frequently                          | Considers data conflicts more frequently                                    |
| Needs a version number stored in a db column for locking a record | Needs an invocation of `with_lock` or `#lock!` methods for locking a record |
| Allows a conflict to occur and may retry or throw an error        | Blocks conflicts until a record is unlocked (a transaction is done)         |
| Is used once a cost of retry is low                               | Is used once a cost of retry is high                                        |

<br>
Rather than locking a record for entire transactions by pessimistic way, I'd take the optimistic approach and that's what I recommend. As alwyas, at the end, it depends on the use case that may force the pessimistic approach.

There are more methods besides optimism and pessimism to stay away from race conditions. Since more libraries need to be installed, let's quickly go over the details of two additional, more particular approaches.

## Advisory locking

Another useful technique is the advisory locking. It doesn't lock records but guarantees that no two processes operate another process at the same time (by adding mutexes). In order to do so you'd need to extend an app with [_with_advisory_lock_](https://github.com/ClosureTree/with_advisory_lock) gem. See official docs for details.

## Background processing and queues

I'm sure that all readers know what background processing is as it's fundamental. In ruby apps Sidekiq is used frequently to handle background jobs that may alter db records. An extension to Sidekiq - [SidekiqUniqueJobs](https://github.com/mhenrixon/sidekiq-unique-jobs) adds extra constraints and prevents from race conditions there. The configuration is pretty straight forward since only an extra middleware must be configured. See official docs for details.

It may be difficult to create consistent systems without problems with data integrity so knowing strategies that let us avoid race conditions are desirable. Of course, there are more locking strategies, but I've only shown those that I find most useful in my day-to-day work as a developer.
