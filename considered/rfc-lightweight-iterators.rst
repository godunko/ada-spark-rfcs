- Feature Name: lightweight-iterators
- Start Date: 2019-10-03
- RFC PR:
- RFC Issue:

Summary
=======

This RFC proposes lightweight iterators, which are closer to well-known
iterator pattern. Such iterators may not provide direct access to elements
stored in a container, so internal container representation is free
to use any optimization it wants, including packing, comression, character
encoding conversion, etc.

Motivation
==========

Iterators in the new form are less expensive than iterators from Ada 2012
in term of CPU and memory consumption. Their design matches iterator pattern
from well-known GoF design patterns, so meet to natural expectation of
most programmers. Author of the container is able to sacrifice an ability
to update current element in the container through syntax sugar in those
cases when his container implementation choose alternative representation
of stored elements.

Iterators from Ada 2012 standard use two types:
- a Cursor type to keep position in the container and, perhaps, iteration state;
- an Iterator type to perform operations like First, Next and so on.

In case of simple container implementation cursor could be as simple as index in
the array or pointer to the element.

In Ada 2005 Cursor is used as kind of reference to the container element:

.. code:: ada

   function Element (Position : Cursor) return Element_Type;

In Ada 2012 access to the element provided through Reference_Type:

.. code:: ada

   function Reference
     (Container : aliased in out Vector;
      Position  : Cursor) return Reference_Type;

In this case Cursor may (should?) contain a pointer to the container
to prevent using cursors from other containers. Iterator provides
a function to move the cursor:

.. code:: ada

   function Next
     (Object   : Forward_Iterator;
      Position : Cursor) return Cursor;

In some cases cursor becomes much more complex type.

In the first example consider definition of abstract interface of
some elements, implementation of which is defined latter or there are
several alternative implementations. We don't know implementation details
in advance, so the only way we can define Cursor is a class-wide type.
As result on each iteration new class-wide object is created, copied and
old one is destroyed. This is rater expensive.

.. code:: ada

   type Element_Vector is interface;

   type Abstract_Cursor is interface;

   function Has_Item (Self : Abstract_Cursor) return Boolean
     is abstract;

   function Has_Element (Self : Abstract_Cursor'Class)
     return Boolean is (Self.Has_Item);

   package Iterator_Interfaces is new
      Ada.Iterator_Interfaces (Abstract_Cursor'Class, Has_Element);

   function Iterate (Self : Element_Vector) return
      Iterator_Interfaces.Reversible_Iterator'Class is abstract;

For the second example consider a case when container keeps elements in some
other form than elements are visible in the interface. It could be
UTF-8 string, packed array or compressed representation of the elements.
In this case we can't return element access in a Reference_Type:

.. code:: ada

   type Reference_Type (Element : not null access Wide_Wide_Character) is
   private with
      Implicit_Dereference => Element;

We have no choise except having a functions that accept Cursor and return
an Element and its properties (for a character of string user may need to
known offset in UTF-8 and UTF-16 units, for list of element in AST it
could be a separator token, etc). If we decide to keep an element and
all its properties in the cursor the we do an extra work while copying
the element and calculating properties in advance, even when user don't
uses them. Otherwise cursor keeps a reference to the containter
and to avoid dangling references we should make Cursor controlled.
Iteration with controlled cursor is expensive.

We can overcome this by dropping Cursor type altogether and keep iteration
state in the iterator itself. Has_Element becomes function of the Iterator
itself, while Next operation becomes a procedure:

.. code:: ada

   package Ada.Iterators is
      type Iterator is limited interface;

      function Has_Element (Self : Iterator)
        return Boolean is abstract;

      procedure Next (Self : in out Iterator)
        with Pre => Self.Has_Element is abstract;

   end Ada.Interfaces;

   type Element_Vector is interface;

   type Element_Iterator is limited interface
     and Ada.Iterators.Iterator;

   function Element (Self : Element_Iterator)
     return My_Element is abstract;

   function Separator (Self : Element_Iterator)
     return Lexical_Element is abstract;

   function Iterate (Self : Element_Vector)
     return Element_Iterator'Class is abstract;

   procedure Print (Vector : Elelemt_Vector) is
   begin
      for Iter in Vector.Iterate loop
         Print (Iter.Element);
         Print (Iter.Separator);
      end loop;
   end Print;
   
No need to instantiate a generic here to define new iterator, because
iteration doesn't depend on element type or cursor. For a loop
statement compiler creates just one object of Iterator'Class, no
assigments, creation and finalization on each iterator.

Why are we doing this? What use cases does it support? What is the expected
outcome?

Guide-level explanation
=======================

Explain the proposal as if it was already included in the language and you were
teaching it to another Ada/SPARK programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.

- Explaining how Ada/SPARK programmers should *think* about the feature, and
  how it should impact the way they use it. It should explain the impact as
  concretely as possible.

- If applicable, provide sample error messages, deprecation warnings, or
  migration guidance.

For implementation-oriented RFCs (e.g. for RFCS that have no or little
user-facing impact), this section should focus on how compiler contributors
should think about the change, and give examples of its concrete impact.

For "bug-fixes" RFCs, this section should explain briefly the bug and why it
matters.

Reference-level explanation
===========================

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

Rationale and alternatives
==========================

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?
- How does this feature meshes with the general philosophy of the languages ?

Drawbacks
=========

- Why should we *not* do this?


Prior art
=========

Discuss prior art, both the good and the bad, in relation to this proposal.

- For language, library, and compiler proposals: Does this feature exist in
  other programming languages and what experience have their community had?

- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other languages, provide readers of your RFC with a fuller
picture.

If there is no prior art, that is fine - your ideas are interesting to us
whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does
not on its own motivate an RFC.

Unresolved questions
====================

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?

- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?

- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

Future possibilities
====================

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
