BigMPI-paper
============

A paper about [BigMPI](https://github.com/jeffhammond/BigMPI).

# TL;DR version:

I have implemented large-count wrappers for p2p, bcast,
(all){gather,scatter}, alltoall and rma, which are the most obvious
functions where large-count support would be useful.

This is very much a work-in-progress and the unit tests are
incomplete.  However, I want some people to critique it before I get
too far along.

# Technical details and associated commentary/background:

Background: The Forum declined to add functions in MPI-3 to support
large counts, instead encouraging users to work around the INT_MAX
limitation of C int counts with datatypes and providing the minimum
required functionality (i.e. datatype query functions) to make this
possible.

Motivation: Users don't like it when the Forum punts on things like
this (e.g. http://gentryx.de/news_the_troubling_state_of_MPI.html).
Apparently Pavan told some Europeans that I was going to solve this
and so I had no choice but to defend my honor and actually finish that
project.

Interface: The API follows the pattern of `MPI_Type_size_x` w.r.t.
`s/int/MPI_Count/` for the count type, but where I use `MPIX` since
these functions are non-standard.  Currently I support only C but
Fortran is on the TODO list.

Limitations: Even though `MPI_Count` might be 128b, I am only supporting
64b counts (because of `MPI_Aint` limitations and desire to use `size_t`
in my unit tests), so BigMPI is not going to solve your problem if you
want to communicate more than 8 EiB of data in a single message.  If
you have more than 8 EiB of memory and a 1+ PiB/s interconnect in your
system, please let me know so that I can use it :-)

# Technical details

[MPI_Type_contiguous_x](https://github.com/jeffhammond/BigMPI/blob/master/src/type_contiguous_x.c)
does the heavy lifting.  It's pretty obvious how it works.  Pavan
tells me that the datatypes engine will turn this into a contiguous
datatype internally and thus it will be efficient.  MPI
implementations need to be count-safe for this to work, but they need
to be count-safe period if the Forum is serious about datatypes being
the solution rather than `MPI_Count` everywhere.

All of the communication functions follow the same pattern, which is
clearly seen in [MPIX_Send_x](https://github.com/jeffhammond/BigMPI/blob/master/src/sendrecv_x.c).
I've optimized for the common case of count<2^31 with a `likely_if`
macro (because you clearly care about latency if your app ships around
>2GiB payloads...).

The most obvious optimization I can see doing is to implement
`MPIX_Type_contiguous_x` using internals of MPICH and possibly some
other implementation instead of calling eight MPI datatype functions.
I already started working on `MPIX_Type_contiguous_x` in MPICH anyways
and will try to get that upstream in the near future.  I don't know
enough Open-MPI guts to do that one right now but I'm sure somebody
involved in that project can figure it out in short order.

# Feedback and such

I would really appreciate people telling me everything that I am doing
wrong w.r.t. BigMPI (but not other things - there just isn't enough
time :-) ).

Please also help me prioritize further development (other than a
complete set of unit tests) by telling me features are missing.  And
of course, let me know if the design is stupid or the implementation
is buggy.

While email works great, I am much more susceptible to developer shame
if you file bug reports and feature requests on Github
(https://github.com/jeffhammond/BigMPI/issues).

Finally, please feel free to use the Github fork button and/or` git-format-patch`.

IMPORTANT NOTE:  You need to undefine `BIGMPI_MAX_INT` in src/Makefile
and test/bigmpi/Makefile to do serious testing.  I set this value
artificially low to make for productive debugging and this is what is
committed right now.

# Future Work

There is a large potential user base if I provide a Fortran 77/90
interface that works with `-i8` (or equivalent).  I do not care about
satisfying the PMPI requirements here and will just drop from MPIX
Fortran into the MPIX C functions.

BigMPI does not do a great job of error checking, particularly to
confirm the implementation is count-safe.  I will add those soon.

Obviously, I need to finish the unit tests.  And so far, I have only
run them with an artificial `INT_MAX` of 10^6 because my laptop becomes
unresponsive if I allocate 4GB.  I need to hope on a workstation and
do proper `>2^31` count testing.  I apologize in advance if you beat me
to this and find obvious bugs.

The docs aren't complete but that's high on the priority list.

# Humorous addendum

Please, please refer to this as THE solution for Big Data w.r.t. MPI.
I totally love Big Data hype.

My proposed title for the Jeff Squyres Cisco blog post on this is "You
can haz moar counts!" :-)

# Conclusion

Anyways, thanks in advance for any feedback.  And a big thank you to
Pavan for preliminary discussions and Andreas Schäfer for blog-shaming
me into working on BigMPI again.
