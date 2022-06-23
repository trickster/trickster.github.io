+++
title = "Worker Queues"
date = 2021-05-27T08:10:55+09:00
type = "post"
description = "Different job queue systems"
in_search_index = true
[taxonomies]
tags = ["Dev", "Tools"]
+++

Tried Beanstalkerd, for a job queue, realized the language support is not very good. But, I realized Redis has Lists. We can use them to push (from producer) and pop (worker/consumer)

Producer pushes

```sh
127.0.0.1:6379> RPUSH list2 0
(integer) 1
127.0.0.1:6379> RPUSH list2 1
(integer) 2
127.0.0.1:6379> RPUSH list2 2
(integer) 3
127.0.0.1:6379> RPUSH list2 3
(integer) 4
127.0.0.1:6379> RPUSH list2 4
(integer) 5
```

Consumer pulls from the queue

```sh
127.0.0.1:6379> BLPOP list2 0
1) "list2"
2) "0"
127.0.0.1:6379> BLPOP list2 0
1) "list2"
2) "1"
127.0.0.1:6379> BLPOP list2 0
1) "list2"
2) "2"
127.0.0.1:6379> BLPOP list2 0
1) "list2"
2) "3"
127.0.0.1:6379> BLPOP list2 0
1) "list2"
2) "4"
```

We can also push multiple items to the list, and have multiple consumers listening to the same list with no duplication. P.S. - "B" in "BLPOP" is blocking operation, waits for the message from the queue
