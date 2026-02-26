.. _event-loop:

Event Loop
==========

FRR uses a custom event loop (``lib/event.c``, ``lib/frrevent.h``) as the
central scheduling mechanism for all daemons.  Every thread of execution —
the main thread and any ``frr_pthread`` worker threads — runs the same
two-function loop:

.. code-block:: c

   /* lib/frr_pthread.c:366  (pthread variant) */
   /* lib/libfrr.c:1257      (main thread variant) */
   while (running) {
       if (event_fetch(master, &task))
           event_call(&task);
   }

``event_fetch()`` picks the next piece of work; ``event_call()`` executes it.
They alternate forever until the thread is told to stop.


Event types
-----------

Four kinds of work are multiplexed through a single ready queue:

+------------------+---------------------------+------------------------------------+
| Type             | Scheduled with            | Triggered when                     |
+==================+===========================+====================================+
| ``EVENT_READ``   | ``event_add_read()``      | fd becomes readable (epoll/poll)   |
+------------------+---------------------------+------------------------------------+
| ``EVENT_WRITE``  | ``event_add_write()``     | fd becomes writable (epoll/poll)   |
+------------------+---------------------------+------------------------------------+
| ``EVENT_TIMER``  | ``event_add_timer()``     | monotonic clock passes deadline    |
+------------------+---------------------------+------------------------------------+
| ``EVENT_EVENT``  | ``event_add_event()``     | promoted to ready queue immediately|
+------------------+---------------------------+------------------------------------+

All four end up on the same ready queue and are dispatched in FIFO order.


``event_fetch()`` — select the next event
------------------------------------------

The real work is in ``event_fetch_inner_loop()`` (``lib/event.c``).  One
iteration proceeds as follows:

1. **Lock ``m->mtx``.**

2. **Drain the ready queue.**  If something is already queued, pop it,
   copy it into the caller's stack-local ``fetch`` struct via
   ``event_run()`` (which is just a ``*fetch = *event`` — it does **not**
   call the handler), unlock ``m->mtx``, and return immediately.

3. **If nothing is ready, prepare to poll.**  Promote any pending
   ``EVENT_EVENT`` items to the ready queue.  Compute the poll timeout
   from the nearest pending timer.

4. **Unlock ``m->mtx``.**

5. **Block in ``fd_poll()``** — ``epoll_wait()`` or ``poll()``.  This is
   where the thread sleeps until an fd fires or a wake-up signal arrives.

6. **Lock ``m->mtx``** again.

7. **Process results:** fire expired timers into the ready queue; move
   triggered I/O fds into the ready queue.

8. **Unlock ``m->mtx``.**

9. Loop back to step 1.  The ready queue now has work, so step 2 will
   pop an event and return it to the caller.

.. important::

   ``event_run()`` (called in step 2 while ``m->mtx`` is held) does
   **not** invoke the handler.  It only copies the event struct into
   the caller's stack variable so the original can be recycled.


``event_call()`` — run the handler
-----------------------------------

``event_call()`` (``lib/event.c``) is straightforward:

- Record timestamps for CPU-hog detection.
- **Invoke the handler:** ``(*event->func)(event)`` (line 2730).
- Measure wall/CPU time and update statistics.
- Warn if the handler exceeded the time-slice threshold.

**No locks are held during handler execution.**  ``m->mtx`` was released
before ``event_fetch()`` returned.


Cross-thread scheduling
-----------------------

Any thread may schedule events on another thread's event loop.  For example,
the BGP main thread can call ``event_add_read(io_master, ...)`` to schedule
a read on the I/O thread.

The mechanism:

1. ``_event_add_read_write()`` acquires the target master's ``m->mtx``,
   registers the fd in the poll set (epoll or poll array), and releases
   ``m->mtx``.

2. ``AWAKEN(m)`` writes a byte to ``m->io_pipe``.  This pipe fd is
   registered in the poll set, so the write wakes the target thread out
   of ``fd_poll()`` and causes it to notice the newly registered fd.


Threading and lock ordering
---------------------------

Each ``event_loop`` (master) has its own mutex ``m->mtx``.  The
``event_add_*`` and ``event_cancel*`` functions acquire this mutex
internally.

When writing code that interacts with the event loop from a different
thread, be mindful of lock ordering:

- **Do not call ``event_add_*()`` while holding a lock that the handler
  itself will acquire**, unless you can prove the target thread never
  holds ``m->mtx`` and the contested lock simultaneously.  While the
  current code may not deadlock, it creates fragile ordering dependencies
  on event-loop internals.

- **Prefer setting a flag under your lock and calling ``event_add_*()``
  after releasing it.**  This pattern avoids nested locks entirely.

Example from BGP (``bgpd/bgp_packet.c``):

.. code-block:: c

   bool rearm_reads = false;

   frr_with_mutex (&connection->io_mtx) {
       connection->curr = stream_fifo_pop(connection->ibuf);
       if (connection->curr &&
           CHECK_FLAG(connection->thread_flags,
                      PEER_THREAD_READS_BLOCKED) &&
           connection->ibuf->count < bm->inq_limit) {
           UNSET_FLAG(connection->thread_flags,
                      PEER_THREAD_READS_BLOCKED);
           rearm_reads = true;
       }
   }

   /* Call outside the mutex to avoid nesting io_mtx -> m->mtx */
   if (rearm_reads)
       bgp_reads_rearm(connection);


BGP I/O — a concrete example
-----------------------------

BGP runs two event loops concurrently:

**Main thread** (``bm->master``)
   Runs ``bgp_process_packet()`` to process parsed BGP messages, drive the
   FSM, perform route selection, etc.

**I/O pthread** (``bgp_pth_io->master``)
   Runs ``bgp_process_reads()`` and ``bgp_process_writes()`` to perform
   non-blocking socket I/O.

The data flow for incoming packets:

1. The I/O thread's ``bgp_process_reads()`` fires when the peer's fd is
   readable.  It calls ``bgp_read()`` (under ``io_mtx``) to pull bytes
   into the ring buffer, then ``read_ibuf_work()`` to frame complete BGP
   messages and push them onto ``connection->ibuf``.

2. If ``ibuf->count`` reaches ``bm->inq_limit``, ``read_ibuf_work()``
   sets ``PEER_THREAD_READS_BLOCKED`` and returns ``-ENOMEM``.
   ``bgp_process_reads()`` skips re-arming the read event — the I/O
   thread stops reading from this peer's socket.

3. After pushing packets, the I/O thread schedules
   ``bgp_process_packet()`` on the main thread via ``event_add_event()``.

4. The main thread's ``bgp_process_packet()`` pops packets from
   ``connection->ibuf`` (under ``io_mtx``).  After each pop, if the
   ``PEER_THREAD_READS_BLOCKED`` flag is set and the queue has dropped
   below the limit, it clears the flag and calls ``bgp_reads_rearm()``
   to schedule a new read event on the I/O thread — resuming I/O.

This back-pressure mechanism prevents unbounded memory growth when the
main thread cannot keep up with incoming data, while ensuring the I/O
thread is re-armed once the queue has drained.
