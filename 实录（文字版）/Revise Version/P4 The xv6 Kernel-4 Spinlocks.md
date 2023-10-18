### P4 The xv6 Kernel-4 Spinlocks

(00:01) This video is part of a series on the xv6 operating system kernel.
(00:06) In this video, I'm going to describe spinlocks and tell you how they're
(00:09) implemented in the xv6 kernel. First let's start off with remembering
(00:14) what spinlocks are. The idea is that the spinlock is represented with a
(00:20) single word and I've called that word "locked" here.
(00:24) And that word has one of two values: it's either 0 or
(00:28) 1. If it's 0, that means the lock is free or sometimes we say it's unlocked
(00:34) or it has been released and if the value is one then the lock is
(00:38) said to be held or acquired or locked.
(00:43) These are the common values that are typically used and xv6
(00:47) does that as well. Okay, so
(00:51) here is the structure that xv6 uses for a spinlock.
(00:57) There are two other fields here: name and cpu which you can see are just used for
(01:03) debugging, so here's what a spinlock looks like. In addition to the key field
(01:08) that tells the state of the lock, we have a pointer to a string which could be
(01:14) used for debugging and we have this field called cpu which
(01:18) points to some other structure. Every
(01:22) core has a structure associated with it, called a
(01:26) cpu structure and this field contains a pointer to the
(01:31) structure for the cpu that's currently holding the lock.
(01:36) Okay, so the important functions on a spinlock or for or any lock really are
(01:42) acquire() and release(). In addition, we have a function that is
(01:46) to be called when the lock is first created that initializes the lock.
(01:51) Basically it's passed the name that we associate with the lock.
(01:56) Once the name is set, it does not change. We've also got a function that's used
(02:00) for error checking to determine whether the current core is holding the lock.
(02:05) Each of these functions is passed a pointer to one of these spinlock
(02:10) structs, so it's passed a pointer like this pointer here.
(02:15) Okay, to acquire the lock, typically what we do is we just set, we want to set
(02:21) the field to 1, but before we do that, we need to check
(02:25) to make sure that it's not currently being held, so our first pass at coding,
(02:30) the acquire() function might look like this:
(02:33) check to see whether the lock is free and if it is free then set it to 1
(02:38) and if it's not free then loop back and keep trying, so these are spinlocks, they
(02:44) spin, it's a tight loop that keeps checking until it finds that the
(02:49) field is 0. It sets it to 1 and then at that point the lock is acquired.
(02:55) Well, of course, this has a problem. If there are other threads that are
(02:59) executing concurrently, in the case of xv6, we've got multiple cores, so this one
(03:05) particular field in memory may be accessed simultaneously by other cores,
(03:11) or perhaps if there's thread switching going on within the
(03:14) single core, perhaps some other thread within the same core is trying to access
(03:18) the same memory location and we could have both, we could have
(03:22) two threads simultaneously check this field,
(03:26) and just happen to find that they are both
(03:29) that if the lock is unlocked and then simultaneously both set the lock to 1
(03:35) and so, that's going to cause a problem, because they both now think they
(03:39) are holding the lock, so we have to
(03:44) find another way to do it, this code will not work,
(03:47) and for that the RISC-V architecture has an
(03:51) instruction called amoswap, atomic memory operation swap,
(03:56) and this single instruction will do two things. It will copy a value into
(04:02) a word of memory and at the same time, it will retrieve the previous value of that
(04:06) word and it will do it without interruption. So
(04:10) it will not allow any other threads or any other
(04:14) instructions whether from this core or another core
(04:17) to do anything between
(04:20) the retrieval of the value and the setting of the value.
(04:26) So, here's how we're going to use that atomic memory swap operation to
(04:31) correct the problem that this code has. We're going to execute the atomic swap
(04:36) operation which I'm indicating schematically here. We're going to write
(04:41) a 1 into the location and we're also going to retrieve the old value, the
(04:45) previous value, if the previous value was 0, then we're
(04:49) done, we have acquired the lock. But if the previous value was not 0, well it
(04:54) must have been 1 because there are no other options,
(04:57) then somebody else was previously holding the
(05:00) lock, so we have to loop back and try again, so then we spin until finally we
(05:06) find the previous value was 0 and we then have acquired the lock.
(05:11) The release operation is pretty straightforward. We just copy
(05:15) the unlocked value of 0 into the word.
(05:20) This doesn't, it's hard to imagine how copying value
(05:24) into a single location of memory can be anything but atomic, so in some systems,
(05:29) this is implemented as just a single memory store operation.
(05:36) Okay, now let's look at the code for the acquire operation.
(05:42) First of all, we can take care of the init operation, this is code that's
(05:45) coming from the spinlock.c file,
(05:52) initialize the spinlock, we're passed a pointer to the structure
(05:55) and the name, we set the name field and it never changes after that.
(06:00) We also set the cpu field to null, because the lock is not held and we set
(06:04) its initial value to 0.
(06:07) Here is the code for the acquire() function
(06:10) and here, right here, we see the while loop that does exactly what I described
(06:15) previously. This magic `__sync_lock_test_and_set()` is
(06:20) going to be turned into some assembly code and the
(06:24) comment here is indicating that the amoswap
(06:28) instruction is going to be happening and it talks
(06:32) about some registers, but essentially we're passing it a pointer to the word
(06:36) that we want to update and we're passing it the new value which
(06:40) is 1 and this function is going to return the old value and we're checking
(06:45) to see whether it is 0 or not and if it was not 0, then we
(06:51) repeat this while loop and if it was 0, then we've acquired
(06:55) the lock and we're done now. There are a couple of other things I
(06:58) want to mention about this function. First,
(07:01) once we acquire the lock, we are storing into the cpu field.
(07:07) Every core has a structure associated with it,
(07:11) so there are in fact eight cores, so there
(07:13) are eight structures, and this function here will retrieve a pointer or return a
(07:18) pointer to the structure for the core that's currently executing, so we're just
(07:23) saving that here, up here, we are
(07:27) checking to see whether, before we acquire the lock, whether we already,
(07:32) whether it's already being held by us. so we'll see the
(07:37) holding() function in just a second, and if it is, then something's drastically the
(07:41) matter, so we cause an error message, then we've got this `__sync_synchronize()`
(07:48) going on here, so tell the C compiler and the processor
(07:51) not to move loads or stores past this point, to ensure that the critical
(07:55) sections memory references happen strictly after the lock is acquired.
(08:00) This is a fence instruction, so we want to prevent the compiler
(08:05) from doing optimizations. It might in fact reorder things,
(08:10) and that is not acceptable, though anything that's supposed to be done after we
(08:16) acquire this lock, after this while loop completes, must be done after that,
(08:21) so that's what this `__sync synchronize`() does.
(08:25) It forces the compiler to emit code and the
(08:29) processor to execute that code in such a way that
(08:32) everything that is supposed to happen before this point
(08:35) is completed and everything that is supposed to happen after this point is
(08:40) not started until after this point. We see one other thing here that's kind
(08:45) of unusual, and that is push_off(). The comment says disable interrupts to
(08:49) avoid deadlock. Holy Smoke! What's that? I mean, isn't
(08:54) this a lock situation here? Why are interrupts involved at all?
(08:58) I'll come back to that in a minute, so I'm gonna just leave you in suspense.
(09:04) But first, let's take a look at the release operation.
(09:09) We do a quick check to make sure that we are in fact holding this lock
(09:15) and if not, we print an error message, and then we're going to be releasing it down
(09:20) here, and so we might as well go ahead and set
(09:23) the cpu field to null at this point and
(09:27) here we are copying the 0 into
(09:31) this field. They're using an amoswap instruction to do it with this
(09:36) `__sync_lock_release()`, but like I said, it's hard to imagine how
(09:40) a normal stored of memory could be anything less than atomic
(09:47) and we've got pop_off().
(09:50) Well, we've got the `__sync_synchronize()` here, up here,
(09:54) which makes sure that anything that's supposed to happen before we release the
(09:57) lock actually finishes and that anything that's
(10:02) and only after we finish that, do we actually set the locked field to 0.
(10:08) So pop_off() is a partner, a dual of push_off() and so,
(10:14) what it does is it enables interrupts, and I'll come back to that in a second,
(10:20) but first let's look at this holding() function just to complete this code.
(10:26) This just checks to see whether the current core that's executing this is
(10:31) holding the lock. Okay it
(10:36) looks at the value of the field and if it's 1 and
(10:40) the cpu is the same as this core then it returns true and otherwise it returns
(10:45) false. A spinlock should never be held for a
(10:51) long period of time. Imagine a situation where one core is
(10:55) holding a lock and another core wants to acquire that lock.
(10:58) Well, the acquire() function is going to be spinning in a tight loop, waiting for the
(11:02) lock to become free, so as a general rule of thumb, you should always plan to
(11:08) release the lock soon after it's acquired.
(11:11) In particular, we don't want the thread that's holding the lock to go to sleep
(11:15) or to get time sliced. There are other techniques for locking
(11:19) in xv6. We have the sleep() and the wakeup() function and these can be used in
(11:24) situations where the lock needs to be held for a long period of time.
(11:28) Locks are used to protect shared data. Actually the xv6 documentation has some
(11:34) interesting wisdom. They point out that locks really protect
(11:39) constraints and I think that's a nice way to put it. But
(11:43) in any case, we usually or almost always see this
(11:47) particular pattern here. We acquire
(11:52) the lock then we access the shared data and
(11:56) then we release the lock. The
(12:00) code here is often called a critical section, in other words it's critical, it
(12:06) can only be executed by one thread at a time, so therefore this code is protected
(12:11) by a lock.
(12:13) Let's look at a quick example and I'm going to imagine
(12:17) input a character, input that's coming from somewhere and going to someplace
(12:22) else. So perhaps we have a keyboard and every time the user types on the
(12:26) keyboard, an interrupt handler wakes up and adds a
(12:30) character to a shared buffer. So the buffer is the
(12:34) shared memory item and it's going to be a fifo queue. So the data goes in one end
(12:40) and there's some other thread that is reading from the buffer
(12:45) when it needs a character, it just needs to access this shared data
(12:49) structure which we will need to protect and in
(12:53) this example, we will protect it with a spinlock. So
(12:57) here's the code that we might see in the handler.
(13:00) It acquires the spinlock, adds a character to the buffer and then
(13:04) releases the lock the thread that wants to
(13:07) get input is going to acquire the lock, so it can access the shared buffer
(13:11) remove a character and release the lock. I'm not going to be concerned with what
(13:15) happens if the buffer is completely full or completely empty.
(13:19) The handler is called when someone types a
(13:23) character on the keyboard and interrupt handlers
(13:27) begin with the trap processing which will disable interrupts and then they'll
(13:32) do some stuff basically this stuff here in the case of our example
(13:37) and then they will return to the interrupted code with the
(13:41) srep instruction which will re-enable interrupts so we might imagine a thread
(13:46) "T" that's running like this an interrupt happens as a result of a
(13:50) key being pressed the handler runs and then when it's done, adding the character
(13:54) to the buffer, it returns to the interrupted thread whatever it was and
(13:58) continues. Well, so you can imagine this situation
(14:02) if "T" happens to be this thread here that
(14:06) acquires a lock and then at just the wrong moment the
(14:10) interrupt comes in from the keyboard and the handler is invoked and the first
(14:15) thing the handler does is try to acquire that lock.
(14:18) Well it will wait until the lock is released,
(14:21) but unfortunately the thread that is holding the lock
(14:24) is waiting for the handler to complete, so we've got a deadlock situation.
(14:30) In operating systems, we need to remember that if anything, if any combination of
(14:35) events can possibly theoretically happen, it's best to assume that it will happen,
(14:40) no matter how unlikely or how rarely you think the event is.
(14:47) Our first approach to solving the problem of deadlock is
(14:50) to disable interrupts in the acquire() function
(14:54) and then to re-enable them in the release() function.
(14:58) So we don't have a situation like this: if "T" acquires a lock, it will
(15:03) disable interrupts, so it cannot have a situation where some other thread on
(15:08) that same core is trying to acquire the lock.
(15:13) This also has an additional benefit we don't want to hold spinlocks for
(15:17) very long, so by disabling interrupts we are
(15:21) preventing time slicing from happening while the lock is held and we are
(15:26) preventing interrupt processing from occurring
(15:29) basically we're allowing the core to focus on the thread that is holding the
(15:33) lock and to complete its critical section without
(15:37) interruption quickly and then release the lock
(15:42) but now we have another problem: What if interrupts are already disabled ?
(15:47) For example, in our handler code we have interrupts disabled and for a
(15:52) number of reasons we don't want to re-enable interrupts prematurely before
(15:56) we get to the system return instruction. we might also have
(16:03) several spinlocks that we need to acquire,
(16:06) and so we have perhaps three acquire() functions being called in a row and for
(16:11) each one of those, we'll have three release() functions,
(16:14) but the first release() will disable, sorry, we'll re-enable
(16:18) interrupts and that might not be quite what we want, so essentially we want this
(16:23) release() to return the interrupt status to
(16:26) whatever it was before the acquire happened and we want to accommodate
(16:31) nested calls that is we want to accommodate several acquire() function
(16:36) calls followed by several release() function
(16:40) calls and our solution to this problem is
(16:43) simple: We're going to use a counter and that counter happens to be called
(16:49) "noff" and it's specific to a core.
(16:53) This variable will be held in the cpu structure. Each core has its own
(16:58) cpu structure and each one will contain its own counter, so the acquire() function
(17:05) will increment the counter and then disable the interrupts,
(17:10) perhaps they were already disabled, but they will be after the acquire() for sure
(17:14) and What's release() going to do ? Well, release() is going to decrement the
(17:18) count and
(17:20) to handle the nested case if it goes back to 0, then it's time to consider
(17:24) re-enabling them. More precisely, we need to ask whether they were previously
(17:29) enabled before the first
(17:33) call to acquire(), so if the count is 0 and they were
(17:37) previously enabled then we will re-enable them
(17:42) that previously enabled status is kept in a variable called interrupt enable
(17:47) "intena", I guess, and so we all we will remember the
(17:52) previous interrupt status before the first call to acquire() in this variable
(17:56) which is a part of the cpu structure. Okay, now we can take a look at the code
(18:04) for push_off() and pop_off().
(18:08) Remember that in the acquire() function shown here,
(18:13) the first thing we did was call push_off() to disable interrupts to
(18:17) avoid deadlock, so now we know what that means
(18:20) and in the release(), we also end up calling
(18:26) pop_off(). Here's the code for release() and right
(18:31) before we return, we re-enable interrupts if appropriate. So let's take a look at
(18:37) push_off() and pop_off().
(18:43) push_off() is called by acquire(). We immediately turn interrupts off. We
(18:47) disable interrupts. Well, first we find out what the previous status of the
(18:51) interrupts was, that's called "old", and then here we're accessing the cpu
(18:56) structure and
(18:59) if this is the first call to acquire(), that is, if the counter is already, is
(19:03) previously 0, then we're going to save the old status,
(19:07) but in any case we're going to increment the counter
(19:10) pop_off() is as you'd expect, pretty similar.
(19:14) We are accessing the counter and the
(19:18) intena variable here,
(19:22) so we start by getting a pointer to the cpu structure
(19:25) and then we decrement the counter at this
(19:28) point here, and if it went to 0 and interrupts were previously enabled,
(19:34) then we turn them back on at this point. We also have some error checking here.
(19:41) If for some reason we call pop_off() and interrupts are are already enabled
(19:45) something's drastically the matter, so we print an error message,
(19:49) and we also make sure that the counter is not 0 or smaller
(19:55) that would also be some sort of an error condition.
(19:58) Okay that's it for spinlocks. See you in the next video.
