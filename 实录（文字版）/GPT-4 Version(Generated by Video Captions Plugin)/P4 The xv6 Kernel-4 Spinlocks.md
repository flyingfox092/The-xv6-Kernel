### P4 The xv6 Kernel-4 Spinlocks

(00:01) this video is part of a series on the xv6 operating system kernel
(00:06) in this video i'm going to describe spin locks and tell you how they're
(00:09) implemented in the xv6 kernel first let's start off with remembering
(00:14) what spin locks are the idea is that the spin lock is represented with a
(00:20) single word and i've called that word locked here
(00:24) and that word has one of two values it's either zero or
(00:28) one if it's zero that means the lock is free or sometimes we say it's unlocked
(00:34) or it has been released and if the value is one then the lock is
(00:38) said to be held or acquired or locked
(00:43) these are the common values that are typically used and xv6
(00:47) does that as well okay so
(00:51) here is the structure that xv6 uses for a spin lock
(00:57) there are two other fields here name and cpu which you can see are just used for
(01:03) debugging so here's what a spin lock looks like in addition to the key field
(01:08) that tells the state of the lock we have a pointer to a string which could be
(01:14) used for debugging and we have this field called cpu which
(01:18) points to some other structure every
(01:22) core has a structure associated with it called a
(01:26) cpu structure and this field contains a pointer to the
(01:31) structure for the cpu that's currently holding the lock
(01:36) okay so the important functions on a spin lock or for or any lock really are
(01:42) acquire and release in addition we have a function that is
(01:46) uh to be called when the lock is first created that initializes the lock
(01:51) uh basically it's past the name that we associate with the lock
(01:56) once the name is set it does not change we've also got a function that's used
(02:00) for error checking to determine whether the current core is holding the lock
(02:05) each of these functions is passed a pointer to one of these spin lock
(02:10) structs so it's past a pointer like this pointer here
(02:15) okay uh to acquire the lock typically what we do is we just set we want to set
(02:21) the field to one but before we do that we need to check
(02:25) to make sure that it's not currently being held so our first pass at coding
(02:30) the acquire function might look like this
(02:33) check to see whether the lock is free and if it is free then set it to one
(02:38) and if it's not free then loop back and keep trying so these are spin locks they
(02:44) spin it's a tight loop that keeps checking until it finds that the
(02:49) field is zero it sets it to one and then at that point the lock is acquired
(02:55) well of course this has a problem if there are other threads that are
(02:59) executing concurrently in the case of xc6 we've got multiple cores so this one
(03:05) particular field in memory may be accessed simultaneously by other cores
(03:11) or perhaps if there's uh thread switching going on within the
(03:14) single core perhaps some other thread within the same core is trying to access
(03:18) the same memory location and we could have both uh we could have
(03:22) two threads simultaneously check this field
(03:26) and just happen to find that they are both
(03:29) uh that if the lock is unlocked and then simultaneously both set the lock to one
(03:35) and and so that's going to cause a problem because they both now think they
(03:39) are holding the lock so we have to uh
(03:44) find another way to do it this code will not work
(03:47) and for that the risk 5 architecture has an
(03:51) instruction called amo swap atomic memory operation swap
(03:56) and this single instruction will do two things it will copy a value into
(04:02) a word of memory and at the same time it will retrieve the previous value of that
(04:06) word and it will do it without interruption so
(04:10) it will not allow any other threads or any other
(04:14) instructions whether from this core or another core
(04:17) to do anything between
(04:20) the retrieval of the value and the setting of the value
(04:26) so here's how we're going to use that atomic memory swap operation to
(04:31) correct the problem that this code has we're going to execute the atomic swap
(04:36) operation which i'm indicating schematically here we're going to write
(04:41) a 1 into the location and we're also going to retrieve the old value the
(04:45) previous value if the previous value was 0 then we're
(04:49) done we have acquired the lock but if the previous value was not 0 well it
(04:54) must have been one because there are no other options
(04:57) then somebody else was previously holding the
(05:00) lock so we have to loop back and try again so then we spin until finally we
(05:06) find the previous value was zero and we then have acquired the lock
(05:11) the release operation is pretty straightforward we just copy
(05:15) the unlocked value of zero into the word
(05:20) uh this doesn't it's hard to imagine how copying value
(05:24) into a single location of memory can be anything but atomic so in some systems
(05:29) this is uh implemented as just a single memory store operation
(05:36) okay now let's look at the code for the acquire operation
(05:42) uh first of all we can take care of the init operation this is code that's
(05:45) coming from the spinlock.c file
(05:52) initialize the spin lock we're passed a pointer to the structure
(05:55) and the name we set the name field and it never changes after that
(06:00) we also set the cpu field to null because the lock is not held and we set
(06:04) its initial value to zero
(06:07) here is the code for the acquire function
(06:10) and here right here we see the while loop that does exactly what i described
(06:15) previously this magic sync lock test and set is
(06:20) going to be turned into some assembly code and the
(06:24) comment here is indicating that the amo swap
(06:28) instruction is going to be happening and it talks
(06:32) about some registers but essentially we're passing it a pointer to the word
(06:36) that we want to update and we're passing it the new value which
(06:40) is one and this function is going to return the old value and we're checking
(06:45) to see whether it is zero or not and if it was not zero uh then we
(06:51) repeat this while loop and if it was zero then we've acquired
(06:55) the lock and we're done now there are a couple of other things i
(06:58) want to mention about this function first
(07:01) once we acquire the lock we are storing into the cpu field
(07:07) every core has a structure associated with it
(07:11) so there are in fact eight cores so there
(07:13) are eight structures and this function here will retrieve a pointer or return a
(07:18) pointer to the structure for the core that's currently executing so we're just
(07:23) saving that here up here we are
(07:27) uh checking to see whether before we acquire the lock whether we already
(07:32) whether it's already um being held by us so we'll see the
(07:37) holding function in just a second and if it is then something's drastically the
(07:41) matter so we um cause an error message then we've got this uh synchronized
(07:48) thing going on here so tell the c compiler and the processor
(07:51) not to move loads or stores past this point to ensure that the critical
(07:55) sections memory references happen strictly after the lock is acquired
(08:00) this is a fence instruction so we want to prevent the compiler for
(08:05) doing up from doing optimizations it might in fact reorder things
(08:10) and that is not acceptable the anything that's supposed to be done after we
(08:16) acquire this lock after this while loop completes must be done after that
(08:21) so that's what this sync synchronize does
(08:25) it forces the compiler to emit code and the
(08:29) processor to execute that code in such a way that
(08:32) everything that is supposed to happen before this point
(08:35) is completed and everything that is supposed to happen after this point is
(08:40) not started until after this point we see one other thing here that's kind
(08:45) of unusual and that is push off the comment says disable interrupts to
(08:49) avoid deadlock holy smoke what's that i mean uh isn't
(08:54) this a lock situation here why why are interrupts involved at all
(08:58) i'll come back to that in a minute so i'm gonna just uh leave you in suspense
(09:04) but first uh let's take a look at the release operation
(09:09) we do a quick check to make sure that we are in fact holding this lock
(09:15) and if not we print an error message and then we're going to be releasing it down
(09:20) here and so we might as well go ahead and set
(09:23) the cpu field to null at this point and
(09:27) here we are copying uh the zero into
(09:31) this field they're using an amo swap instruction to do it with this sync
(09:36) glock release but like i said it's hard to imagine how
(09:40) a and a normal stored of memory could be anything less than atomic
(09:47) and we've got pop off
(09:50) well we've got the synchronize here up here
(09:54) which makes sure that anything that's supposed to happen before we release the
(09:57) lock actually finishes and that anything that's
(10:02) and only after we finish that do we actually set the locked field to zero
(10:08) so pop off is a partner a dual of uh push off and so
(10:14) what it does is it enables interrupts and i'll come back to that in a second
(10:20) but um first let's look at this holding function just to complete this uh code
(10:26) this just checks to see whether the current core that's executing this is
(10:31) holding the lock okay it
(10:36) looks at the value of the field and if it's one and
(10:40) the cpu is the same as this core then it returns true and otherwise it returns
(10:45) false a spin lock should never be held for a
(10:51) long period of time imagine a situation where one core is
(10:55) holding a lock and another core wants to acquire that lock
(10:58) well the acquire function is going to be spinning in a tight loop waiting for the
(11:02) lock to become free so as a general rule of thumb you should always uh plan to
(11:08) release the lock soon after it's acquired
(11:11) in particular we don't want the thread that's holding the lock to go to sleep
(11:15) or to get time sliced there are other techniques for locking
(11:19) in xv6 we have the sleep and the wake up function and these can be used in
(11:24) situations where the lock needs to be held for a long period of time
(11:28) locks are used to protect shared data actually the xv6 documentation has some
(11:34) interesting wisdom they point out that locks really protect
(11:39) constraints and i think that's a nice way to put it but
(11:43) in other in any case we usually or almost always see this
(11:47) particular pattern here we acquire
(11:52) the lock then we access the shared data and
(11:56) then we release the lock the
(12:00) code here is often called a critical section in other words it's critical it
(12:06) can only be executed by one thread at a time so therefore this code is protected
(12:11) by a lock
(12:13) let's look at a quick example and i'm going to imagine
(12:17) input a character input that's coming from somewhere and going to someplace
(12:22) else so perhaps we have a keyboard and every time the user types on the
(12:26) keyboard an interrupt handler wakes up and adds a
(12:30) character to a shared buffer so the buffer is the
(12:34) shared memory item and it's going to be a fifo queue so the data goes in one end
(12:40) and there's some other thread that is reading from the buffer
(12:45) when it needs a character it just needs to access this shared data
(12:49) structure which we will need to protect and in
(12:53) this example we will protect it with a spin lock so
(12:57) here's the code that we might see in the handler
(13:00) it acquires the spin lock adds a character to the buffer and then
(13:04) releases the lock the thread that wants to
(13:07) get input is going to acquire the lock so it can access the shared buffer
(13:11) remove a character and release the lock i'm not going to be concerned with what
(13:15) happens if the buffer is completely full or completely empty
(13:19) the handler is called when someone types a
(13:23) character on the keyboard and interrupt handlers
(13:27) begin with the trap processing which will disable interrupts and then they'll
(13:32) do some stuff basically this stuff here in the case of our example
(13:37) and then they will return to the interrupted code with the
(13:41) s rep instruction which will re-enable interrupts so we might imagine a thread
(13:46) t that's running like this an interrupt happens as a result of a
(13:50) key being pressed the handler runs and then when it's done adding the character
(13:54) to the buffer it returns to the interrupted thread whatever it was and
(13:58) continues well so you can imagine this situation
(14:02) if t happens to be this thread here that
(14:06) acquires a lock and then at just the wrong moment the
(14:10) interrupt comes in from the keyboard and the handler is invoked and the first
(14:15) thing the handler does is try to acquire that lock
(14:18) well it will wait until the lock is released
(14:21) but unfortunately the thread that is holding the lock
(14:24) is waiting for the handler to complete so we've got a deadlock situation
(14:30) operating systems we need to remember that if anything if any combination of
(14:35) events can possibly theoretically happen it's best to assume that it will happen
(14:40) no matter how unlikely or how rarely you think the event is
(14:47) our first approach to solving the problem of deadlock is
(14:50) to disable interrupts in the acquire function
(14:54) and then to re-enable them in the release function
(14:58) so we don't have a situation like this if t acquires a lock it will
(15:03) disable interrupts so it uh cannot have a situation where some other thread on
(15:08) that same core is trying to acquire the lock
(15:13) this also has an additional benefit we don't want to hold spin locks for
(15:17) very long so by disabling interrupts we are
(15:21) preventing time slicing from happening while the lock is held and we are
(15:26) preventing interrupt processing from occurring
(15:29) basically we're allowing the core to focus on the thread that is holding the
(15:33) lock and to complete its critical section without
(15:37) interruption quickly and then release the lock
(15:42) but now we have another problem what if interrupts are already disabled
(15:47) for example in our handler code we have interrupts disabled and for a
(15:52) number of reasons we don't want to re-enable interrupts prematurely before
(15:56) we get to the system return instruction we might also have
(16:03) several spin locks that we need to acquire
(16:06) and so we have perhaps three acquire functions being called in a row and for
(16:11) each one of those we'll have three release functions
(16:14) but the first release will uh disable sorry we'll re-enable
(16:18) interrupts and that might not be quite what we want so essentially we want this
(16:23) release to return the interrupt status to
(16:26) whatever it was before the acquire happened and we want to accommodate
(16:31) nested calls that is we want to accommodate several acquire function
(16:36) calls followed by several release function
(16:40) calls and our solution to this problem is
(16:43) simple we're going to use a counter and that counter happens to be called in
(16:49) off and it's specific to a core
(16:53) this variable will be held in the cpu structure each core has its own
(16:58) cpu structure and each one will contain its own counter so the acquire function
(17:05) will increment the counter and then disable the interrupts
(17:10) perhaps they were already disabled but they will be after the acquire for sure
(17:14) and what's release going to do well release is going to decrement the
(17:18) count and
(17:20) to handle the nested case if it goes back to zero then it's time to consider
(17:24) re-enabling them more precisely we need to ask whether they were previously
(17:29) enabled before the first
(17:33) call to acquire so if the count is zero and they were
(17:37) previously enabled then we will re-enable them
(17:42) that previously enabled status is kept in a variable called interrupt enable
(17:47) int ena i guess and so we all we will remember the
(17:52) previous interrupt status before the first call to acquire in this variable
(17:56) which is a part of the cpu structure okay now we can take a look at the code
(18:04) for push off and pop off
(18:08) remember that in the acquire function shown here
(18:13) the first thing we did was call push off to disable interrupts to
(18:17) avoid deadlock so now we know what that means
(18:20) and in the release we also uh end up calling
(18:26) pop off here's the code for release and right
(18:31) before we return we re-enable interrupts if appropriate so let's take a look at
(18:37) push off and pop off
(18:43) push-off is called by acquire we immediately turn interrupts off we
(18:47) disable interrupts well first we find out what the previous status of the
(18:51) interrupts was that's called old and then here we're accessing the cpu
(18:56) structure and
(18:59) if this is the first call to acquire that is if the counter is already is
(19:03) previously zero then we're going to save the old status
(19:07) but in any case we're going to increment the counter
(19:10) pop off is as you'd expect pretty similar
(19:14) we are accessing the counter and the
(19:18) int enabled variable here
(19:22) so we start by getting a pointer to the cpu structure
(19:25) and then we decrement the counter at this
(19:28) point here and if it went to zero and interrupts were previously enabled
(19:34) then we turn them back on at this point we also have some uh error checking here
(19:41) if for some reason we call pop off and interrupts are are already enabled
(19:45) something's drastically the matter so we print an error message
(19:49) and we also make sure that the counter is not zero or smaller
(19:55) that would also be some sort of an error condition
(19:58) okay that's it for spin locks uh see you in the next video

