## Router Concurrency

      Router
        Reaml1
          sessions
  invk <--- session1............(outHandler) <--------+
   evt <--- session2............(outHandler) <-----+  |
   evt <--- session3............(outHandler) <--+  |  |
   pub ---> session4........(inHandler)         |  |  |
  call ---> session5..(inHandler)  |            |  |  |
                          |        |pub         |  |  |
                      call|        |            |  |  |
                          |        V            |  |  |
          Broker..........)..(Handler)          |  |  |
            subscribers   |     |   |     event |  |  |
              *session2   |     |   +-----------+  |  |
              *session3   |     +------------------+  |
                          |                           |
                          V            invocation     |
          Dealer.......(Handler)----------------------+
            callees
              *session1

          Roles
            callee: session1
            subscriber: session2, session3
            publisher: session4
            caller: session5


### Rules

- Different publishers can dispatch concurrently to one subscriber.
- One publisher can dispatch concurrently to different subscribers.
- One publisher dispatches serially to one subscriber.

### Operation

Each (xHandler) is a singe goroutine+channel that persists for the lifetime of the associated object.  Therefore, order is preserved for messages between any two sessions regardless of goroutine scheduling order.

Broker and Dealer cannot dispatch an event to a new goroutine, because this would mean that messages bound for the same destination session would be sent in separate goroutines, and delivery order would be affected by goroutine scheduling order.  Instead they dispatch to the receiving session's out channel+goroutine.

NOTE: If there are too many messages sent to the same session, the session's out channel may block and the broker or dealer by not be able to continue processing until the session's message can be written to the session's out channel.  This may be solved by putting the messages on an output queue for the session.  See: https://github.com/gammazero/bigchan

Broker and Dealer have their own handler goroutines to:
1) Safely access subscription or call maps.
2) Not make session's input-handler wait for Broker when it could be dispatching other messages to Dealer. Consider using an RWMutex to access maps.

NOTE: Sessions have separate in (recv) and out (send) handlers so that processing incoming messages is not blocked while waiting for outgoing messages to finish being sent. 