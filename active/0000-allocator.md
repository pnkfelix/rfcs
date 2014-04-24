- Start Date: (fill me in with today's date, YYY-MM-DD)
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

# TODO

- Refine the actual intrinsic names and so on and be more specific.

# Summary

Proposes a design for an allocator trait with the following goals:

1. Allow libraries to be generic with respect to the allocator, so
   that users can supply their own memory allocator and still make use
   of library types like `Vec` or `HashMap`.
2. Support the ability of garbage collection to identify roots even
   from within smart pointers like `Rc<T>` and so forth.
3. For those who do not care about GC support, support simple allocators
   that simply want to know the size/alignment of memory to allocate.
4. Do not require that allocators track the size of allocations.
5. Permit dynamically sized types to be cloned in a generic way9.
6. Given `x: T`, support `Smaht::new(x)` with type `Smaht<T>`, *even*
   when those cases where `T` is a dynamically sized type. (Here
   `Smaht` represents a smart pointer type, such as `Rc`, `Gc`, or
   `Heap`.)

# Motivation and use cases

## Support for full type information and traceability

We would like to permit users to embed managed pointers into other
smart pointers and vice versa. In other words, we want something like
this to work:

    let x: Rc<int> = Rc::new(3);
    let y: Gc<Rc<int>> = Gc::new(x);
    let z: Rc<Gc<Rc<int>>> = Rc::new(y);
    
For this kind of tongue twister to be safe, the GC needs to be able to
find roots to managed data both on the stack but also within smart
pointers. For example, the `Rc` pointer `z` is a root for the `Gc`
pointer `y`. This implies that the GC must be able to find and trace
through allocations that may contain `Gc` pointers.

At the same time, we don't want to impose the burden of supporting
tracing on all users. Many users may not employ GC at all, or may be
in a context where GC can't be employed, in which case they will need
to be able to support allocation without overhead. Which leads
us to the next point.

## Support for simple allocators

In those cases where GC is not needed, we want to be sure that the
same API can be used to support simple allocators that only care about
size and alignment (similar to the [other allocator RFC][alloc]
proposed by @thestinger). This means that it should be possible to write
write allocators which:

1. Cannot support tracing or enumerating of allocations.
2. Only require the size/alignment of memory to do allocations, not
   the precise types the memory will contain.
3. Do not track the size of allocations they create (unlike C `malloc`
   and `free`).
   
## The `new` function for smart pointers

We would like to permit smart pointers to support a function `new` that
works roughly as follows:

```
impl<Sized? T> Smaht<T> {
   fn new(value: T) -> Smaht<T>; 
}
```

To some extent, this interacts with the (not yet published) DST
proposal. The key point is that users can write `Smaht::new(x)` for
*any value* `x` and move it into a smart pointer, so long as `x` is
owned. This is true even if `x` has a dynamically sized type like
`[int]` or `Trait`. \[[1](#1)\]

Making this work in a generic way requires some amount of machinery
for working with smart pointers, particularly since we want to ensure
that the allocator knows the full type of the memory it contains.

## Objects vs generics

While we expect that most users of the `Alloc` trait will be generic
code that is parameterized over an allocator type, it would be best if
it is still possible to have allocator objects like `&Alloc<Foo>`
(which can be used to allocate instances of `Foo`). This permits users
to control code duplication to some extent.

# Detailed design

The design comes in four parts:

1. Generic machinery for working with smart pointers.
2. The Generic allocator traits and default allocator.
3. Making allocators that do not interoperate with GC.

## Intrinsics for working with smart pointers

First, we introduce into the standard library an opaque type called
`PointerData<U>`, which represents the extra data associated with a
(fat or thin) pointer:

    // Opaque tag that is known to the compiler describing the
    // extra data attached to a (fat or thin) pointer.
    #[deriving(Copy,Eq)]
    struct PointerData<Sized? U> {
        opaque: uint,
        marker: InvariantType<U>
    }
    
Since thin pointers to sized types don't have any extra data,
you can create one of these for any sized type:
    
    // Returns pointer data for any sized type.
    fn thin_pointer_data<T>() -> PointerData<T> {
        PointerData { opaque: 0, marker: marker::InvariantType }
    }
    
Moreover, you can use this function to create the pointer data
for an array `[T]` of length `n`:

    // Returns pointer data for any sized type.
    fn array_pointer_data<T>(length: uint) -> PointerData<[T]> {
        PointerData { opaque: length, marker: marker::InvariantType }
    }
    
We could also add an intrinsic to get the pointer data for an object
type `Trait`, but I don't see a use for it.

If you have an actual pointer `&U`, you can extract its pointer data
using the intrinsic `pointer_data`, which takes a (possibly fat)
pointer and returns the pointer data associated with it. The interface
actually takes 

    // Given fat (or thin) pointer of type U1, extract data suitable
    // for `U2`. The types U1 and U2 must require the same sort of
    // meta data: in particular U2 may be a struct type (or tuple type etc)
    // whose field field is of the type U1.
    /*intrinsic*/ fn pointer_data<Sized? U1, Sized? U2>(x: &U1) -> PointerData<U2>;

There is also the `pointer_mem` intrinsic, which extracts out the raw
thin pointer. It also returns a `*u8`.

    // Given fat (or thin) pointer, extract the memory
    /*intrinsic*/ fn pointer_mem<Sized? U>(x: &U) -> *mut u8;

Next, we have the intrinsics for computing the size and alignment
requirements of a type. Note that these operator over any type, sized
of unsized, but a pointer data is required:

    /*intrinsic*/ fn sizeof_type<Sized? U>(data: PointerData<U>) -> uint;
    /*intrinsic*/ fn alignof_type<Sized? U>(data: PointerData<U>) -> uint;
    
Finally, we have an intrinsic to make a pointer. If `U` is a sized
type, this is the same as a transmute, but if `U` is unsized, it will
pair up `p` with the given pointer data (note that pointer data is
always POD).
    
    // Make a fat (or thin) pointer from some memory
    /*intrinsic*/ fn make_pointer<Sized? U>(p: *mut u8,
                                            data: PointerData<U>)
                                            -> *mut U;

Based on the above intrinsics you can build the following user-facing
functions for computing sizes:

    fn sizeof<T>() -> uint {
        sizeof_type(thin_pointer_data::<T>())
    }

    fn sizeof_value<Sized? U>(x: &U) -> uint {
        let data = pointer_data::<U,U>(x);
        sizeof_type<U>(data)
    }

Note: intrinsics are needed because the appropriate code will depend
on whether `U` winds up being sized or not. We might be able to get
away with one core intrinsic `ty_is_dst()` or something like that.
This is relatively minor detail: the same helper functions will exist
either way.

## The basic allocator traits and default allocator

### Alloc

The core allocator trait is as follows:

    trait Alloc<Sized? U> {
        fn malloc(&self, data: PointerData<T>) -> *U;
        fn free(&self, pointer: *U);
    }

The trait as a whole is parameterized by the type `U`, which is the
type that will be allocated. Note that this type `U` is not
necessarily sized, and hence in order to find out the size and
alignment, extra information is needed.

The `malloc()` method takes the `PointerData` and, using that,
determines the size and alignment of the memory that is
required. Malloc returns a pointer to the type `U`: this means that
the return value is either thin or fat, as normal. If the returned
value is a fat pointer, its extra word will be `data`. The memory that
is returned is not necessarily initialized.

The `free()` method takes a pointer to be freed. Note that for an
array like `[T]`, this may be a fat pointer, in which case the pointer
itself carries all the information that `free()` needs to deduce the
size. **This interface means that it is illegal to change the length
of an instance of a type like `~[T]`, since then the allocator's
length would be inaccurate.**

### VecAlloc

The core allocator trait is adequate but not convenient for
implementing growable vectors. For those use cases, a second trait
`AllocVec` is provided. This trait includes a default implementation
based on the `Alloc` trait, but all allocators are expected to
override at least some parts of both, because `AllocVec` can be made
more efficient:

    trait AllocVec<T> : Alloc<[T]> {
        fn malloc_array(&self, capacity: uint) -> *mut T {...}
        
        fn realloc_array(&self, old_pointer: *mut T, old_capacity: uint,
                         new_capacity: uint) -> *mut T {...}
        
        fn free_array(&self, pointer: *mut T, capacity: uint) {...}
        
        fn shrink_to_fit(&self, pointer: *T, old_capacity: uint,
                         initialized_length: uint) -> *[T] {...}
    }
    
The meaning of the routines should be largely self-evident. Note that
these methods mostly take and return thin pointers to the element type
(`*T`). They expect the container to be tracking the lengths and
capacities separately and to be performing any bounds checking.

The only exception to the pattern is the `shrink_to_fit()` routine,
which returns a fat pointer `*[T]` whose length will be
`initialized_length`.  The idea is that this function is used when
"releasing" a complete array out of the structure.

The methods in `AllocVec` can all be implemented in terms of the base
`Alloc` trait, although `realloc` and `shrink_to_fit` are inefficient.
It is expected that implementations will want to override these two
methods.

    trait AllocVec<T> : Alloc<[T]> {
        fn malloc_array(&self, capacity: uint) -> *mut T {
            let data = array_pointer_data(capacity);
            transmute::<*mut u8,*mut T>(pointer_mem(self.malloc(data)))
        }
        
        fn realloc_array(&self, old_pointer: *mut T, old_capacity: uint,
                         new_capacity: uint) -> *mut T {
            let new_pointer = self.malloc_array(new_capacity);
            copy(new_pointer, old_pointer, min(old_capacity, new_capacity));
            self.free_array(old_pointer, old_capacity);
            new_pointer
        }
        
        fn free_array(&self, pointer: *mut T, capacity: uint) {
            let data: PointerData<[T]> = array_pointer_data(capacity);
            self.free(make_pointer(pointer, data))
        }
        
        fn shrink_to_fit(&self, pointer: *T, old_capacity: uint,
                         initialized_length: uint) -> *[T] {
            let p = self.realloc_array(pointer, old_capacity,
                                       initialized_length);
            let data: PointerData<[T]> = array_pointer_data(initialized_length);
            make_pointer(p, data);
        }
    }

### The default allocator

There's not much to say here except that the default allocator will
implement both `Alloc` and `AllocVec` for all types:

   struct DefaultAllocator;
   impl<Sized? U> Alloc<U> for DefaultAllocator { ... }
   impl<T> AllocVec<T> for DefaultAllocator { ... }

Note that the default allocator will be integrated with the GC
in such a way that memory allocated.

Allocates that wish to allocate instances of some type `U` that may
contain managed pointers must ensure that the garbage collector knows
how to find that memory later. In other words, the memory must be
registered as a root. The precise mechanism for doing this
registration is not in the scope of this RFC; the only thing that
matters for our purposes is that the default Rust allocator will do
the appropriate registration, whatever that is.

One important point about garbage collection is that, if the type
directly contains managed pointers, the `malloc()` routine of the
default allocator will have to initialize the memory to NULL. The
reason for that is that the consumer of the pointer is not obligated
to initialize it in any particular way (consider a `Vec` type, which
wishes to keep some buffer of extra space allocated but not
initialized).

## Supporting simpler allocators that do not interoperate with GC

There is a need however for simple, custom allocators that cannot
interoperate with GC and hence do not have to do any sort of
registration or management.  In this case, the allocator needs a way
to specify that it cannot allocate instances of types that may reach
managed data.

### The NotManaged trait

To allow allocators to specify types that cannot reach managed data,
we add a builtin, special marker trait `NotManaged`:

    trait NotManaged { }

Like the DST trait `Sized`, this trait is not bound by opt-in
kinds. Rather, the compiler will automatically infer whether any given
type is an instance of `NotManaged` or not.

### The SimpleAlloc trait

The basic interface for simple allocators is as follows:

    trait SimpleAlloc {
        fn alloc_mem(&self, size: uint, align: uint) -> *mut u8;
        fn realloc_mem(&self, ptr: *mut u8, size: uint, new_size: uint) -> *mut u8;
        fn free_mem(&self, ptr: *mut u8, size: uint);
    }
    
Furthermore, there is an impl that bridges between the standard
`Alloc` and `AllocVec` traits and `SimpleAlloc`. Note that this
implementation works for allocating instances of any type `U` that
cannot reach managed data:

    impl<Sized? U: NotManaged, A:SimpleAlloc> Alloc<U> for A {
        fn alloc(&self, data: PointerData<U>) -> *mut U {
            let size = sizeof_type::<U>(data);
            let align = alignof_type::<U>(data);
            let mem = self.alloc_mem(size, align);
            make_pointer(mem, data)
        }

        fn free(&self, ptr: *mut U) {
            let size = sizeof_value(&*ptr);
            let align = alignof_value(&*ptr);
            self.free_mem(ptr, size);
        }
    }
    
I will leave the implementation of `AllocVec` as an exercise to the
reader.

# Allocators in Use

## Writing a function like `Smaht::new`

Here is a definition for a function `Smaht::new(value, alloc)` that
allocates the memory. Note that we do not expect people to write the
interface quite like this, but this proves it can be done.  The
standard way to implement the smart pointer allocation methods will be
as part of a standard trait that is the subject of an upcoming
RFC. (This trait will also interoperate with the `box` keyword.)

    struct Smaht<Sized? U,A=DefaultAlloc> {
        alloc: A,
        data: *SmahtData<U>
    }
    
    struct SmahtData<Sized? U> {
        ...,
        data: U
    }

    impl<Sized? U,A:Alloc<T>> Smaht<U,A> {
        fn new(value: U, alloc: A) -> Smaht<U> {
            // Get the pointer data from `value` for a pointer to `SmahtData<T>`:
            let pointer_data: PointerData<SmahtData<U>> = pointer_data(&value);
            
            // Allocate the `SmahtData<U>`:
            let smaht_data = alloc.malloc(pointer_data);

            // Initialize the `data` field:
            let size = sizeof_value(&value);
            memcpy(&mut smaht_data.data, pointer_mem(&value), size);
            unsafe::forget(value);

            // Package up the result into a smart pointer:
            Smaht { alloc: alloc, data: smaht_data }
        }
    }

## Writing a generic `Vec` type parameterized over an allocator

*TBD*

# Alternatives

1. An allocator trait that is not aware of types.

This makes it very hard to write a garbage collector. To find roots,
the garbage collector must be able to trace the stack and then follow
*through smart pointers* into the heap. This is challenging to specify
and error prone. It also exposes the workings of the garbage collector
in a very real way, and has challenges with types like `Vec` and so
forth that peddle in uninitialized memory.

# Unresolved questions

## Querying the bucket size

What is the best way to allow users to query the bucket size? In other
words, it would be nice if vec can say "give me a vector of *at least*
this capacity" and get back feedback on how many items were actually
allocated.
   
## Alloc::realloc   

The base `Alloc` trait has no `realloc()` method. It doesn't fit
well within the interface. It would be possible to add a method
like:

    fn realloc(&self, pointer: *U, data: PointerData<T>) -> *U;
    
The only real use for this would be if `U` is a type like `[T]`,
in which case one could change the pointer data to create a longer
or shorter array.

However, this is cumbersome compared to the `AllocVec` trait and I am
not sure that it has a reasonable use. Moreover, it is unsafe. For
example, imagine that `U` is an object type `Trait`, then switching
pointer data is not a reasonable operation and may just cause garbage
to occur.

# Footnotes

**Note 1.** <a name=1> The current plan for DST is that values of
dynamically sized type may be passed as parameters, but not stored
into other kinds of local variables.

[alloc]: https://github.com/rust-lang/rfcs/pull/39
