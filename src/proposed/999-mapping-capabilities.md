<!--
  SPDX-License-Identifier: CC-BY-SA-4.0
  Copyright 2025 Trustworthy Systems
-->

# Alternative to (software) ASIDPools - mapping capabilities

- Author: Julia
- Proposed: not yet

## Summary

The ASIDpools have a few issues:

-   They are named confusingly, as the "ASID" does not refer to hardware ASIDs but
    instead to a special internal ID used by seL4 to identify an address space.
    In addition to this, their purpose/reason for existence is not explained
    well in the seL4 manual.

-   It requires some global kernel state and imposes a fixed limit on the number
    of active address spaces ("VSpaces"), e.g. 2^16 on RISC-V. Furthermore,
    re-assigning these software ASIDs requires re-creating the VSpace, so they
    cannot be easily changed at runtime.

-   The actual ASID IDs used cannot be easily controlled from userspace as their
    position in the global ASIDPoolTable is done via a first-free iteration.

-   Issues with stale ASID ID mappings (see [rationale](#rationale-and-alternatives)).


This RFC proposes an alternate solution which creates "mapped frame" and "mapped page table"
capabilities. These are capabilities which "link" together a child (e.g. a frame, but sometimes
a page table) into a parent page table. Because these capabilities do not need
to the physical address of the frame (i.e. "non-physical") --- they convey just
authority of a single index in a page table --- we have enough space to link the two
types of parent/child capabilities together.

(The existing Capability Derivation Tree only allows us to store *one* relationship,
 that of derivation, but not this additional, secondary relationship between
 frames and their parent page tables).

## Motivation

As above in Summary, but more precisely: for time protection, this is shared
kernel state that must be removed or partitioned. Removing the need for this
entirely would be nice.


## Guide-level explanation

<!--
Explain the change or feature as you would to another user of seL4.

This section should:

- clearly outline new named **concepts**
- provide some **examples** of how it will be used, and
- explain how **users** should think about it.

You should describe the change in a manner that is clear to both existing seL4
users and to new users of the ecosystem. For instance, if the change is to the
seL4 API, this section should include the prose of the corresponding new
paragraphs or sections int the seL4 manual.

Any changes that existing users need to make either as a result of the RFC or
such that the can use the feature should be clearly stated here.
!-->

***Before***: constructing a VSpace looks something like this:

1.  Retype enough untypeds into page tables and frames (and ASIDPools)
2.  Assign a software ASID to the root page table ("VSpace" cap)
3.  Map the page tables into the VSpace and any children etc (using a PT cap and vspace cap and vaddr)
4.  Map the frames into the page tables (using a frame cap and vspace cap and vaddr)

Creating shared frames, then the user derives a copy of the *same frame cap*
that had already been mapped, which creates an *unmapped* frame cap. (Internally
the kernel has used space in that frame cap for storage of the mapping info).
This can be then mapped in. Passing around the mapped frame caps conveys
authority over the mapping itself and the frame (as it could be remapped) later.

To cleanup the address space, whilst deleting the VSpace is allowed, it doesn't cleanup up the other
capabilities — instead these must be explicitly cleaned up one by one from bottom to top otherwise
they are "leaked" (no way to unmap otherwise — although one can probably revoke the root untyped).

***Now***: constructing the VSpace looks almost identical:

1.  Retype enough untypeds as page tables and frames (*no ASIDpools*)
2.  Map page tables into the VSpace (vspace cap, PT cap, vaddr, *and* destination cap slot)
3.  Map frames into the page tables (vspace cap, frame cap, vaddr, *and* destination cap slot)

This produces *new* capabilities representing *just* the mapping.
One can then perform the same invocation on the same original frame caps to create new mappings
in different VSpaces for shared frames, rather than needing to copy the cap first and then map.
To cleanup mappings, one can delete/revoke a mapping and it will remove that mapping as expected.[^2]
Revoking untypeds will still clean up every single child and clear all the mappings.


[^2]: It's possible that the behaviour of delete/revoke would automatically be
      cleaning up *all* the children of a mapping as it tries to remove all the references.
      This may/may not be desirable; this is not a limitation of this implementation however,
      as this was essentially required by the previous ASID pool implementation anyway because
      of how `capFMappedASID` was tracked.

      It's possible that it might be desirable to say, take a "mapped page table" and move it into
      another slot or another VSpace entirely without cleaning the children.
      I don't see why this couldn't be done; although it might not be the quickest
      operation because of needing to walk the parent to remove it (though this
      is the same issue as when deleting).

## Reference-level explanation

<!--
Explain the change or feature as you would to the **developers and maintainers**
of the seL4 ecosystem. For instance, if it is a change to the seL4 API, this
section would contain the part that should go into API reference of the seL4
manual.

This section should provide sufficient technical detail to guide any related
implementation and ongoing maintenance. Where relevant, it should discuss
expected maintenance, performance, and verification impact.

This section should clearly describe how this change will interact with the
existing ecosystem, describe particular complex examples that may complicate the
implementation, and describe how the implementation should support the examples
in the previous section.

-->

We create new non-physical capabilities. These use the existing CDT for deriving
from the frame/page table object, but then we can use the two words of information
in the cap itself *purely* for a next/parent pointer to the parent page table,
and the next.

From a mapped frame we can then follow the parent pointer up to where it was mapped (and we know it's index).
We can also follow the next pointer from a PT to the mapped frames, following
the chain of nexts. We can follow the CDT to go from unmapped frame to the (derived)
mapped capabilities, which then allows us to unmap them.

Before::

```
 Capability Land                                                                     Object / Memory Land
┌───────────────────────────────────────────────────────────────────────────────┐   ┌──────────────────────────────────────────────────┐
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│          ┌──────────────────────────────────────────────────────┐  (hiding ─► │   │                                                  │
│          │                       Untyped                        │   for       │   │                                                  │
│          └┬─────────┬─────────┬─────────┬─────────────┬─────────┘   simplicity)   │                                                  │
│           │         │         │         │             │                       │   │                                                  │
│           │         │         │         │CDT tracking │                       │   │                                                  │
│           │         │         │         │             │                       │   │            hardware memory layouts               │
│           │         │         │         │             │                       │   │         ┌──────────────────────────────────┐     │
│           │ ┌────┐  │         │         │             │                       │   │         │                                  │     │
│           │ │    ┼──┼─────────┼─────────┼─────────────┼───────────────────────┼───┼────────►│       page table memory         ◄┼──┐  │
│           └─► PT │  │         │         │             │                       │   │         │  │                               │  │  │
│             │    │  │         │         │             │                       │   │         └──┼───────────────────────────────┘  │  │
│             └───┬┘  │ ┌────┐  │         │             │  capPTBasePtr         │   │         ┌──┼───────────────────────────────┐  │  │
│                 │   │ │    ┼──┼─────────┼─────────────┼───────────────────────┼───┼────────►│  │                               │  │  │
│    also stored  │   └─► PT │  │         │             │                       │   │         │  ▼    page table memory      │ │ │  │  │
│    is the mapped│     │    │  │         │             │                       │   │         └──────────────────────────────┼─┼─┘  │  │
│    vaddr        │     └───┬┘  │ ┌────┐  │             │   capFBasePtr         │   │         ┌──────────────────────────────┼─┼─┐  │  │
│                 │         │   │ │    ┼──┼─────────────┼───────────────────────┼───┼────────►│                              │ │ │  │  │
│                 │         │   └─► Fr │  │             │                       │   │         │            frame             │ ▼ │  │  │
│                 │         │     │    │  │             │                       │   │         └──────────────────────────────┼───┘  │  │
│                 │         │     └───┬┘  │ ┌────┐      │                       │   │         ┌──────────────────────────────┼───┐  │  │
│                 │         │         │   │ │    ┼──────┼───────────────────────┼───┼────────►│                              │   │  │  │
│                 │         │         │   └─► Fr ┼───┐  │                       │   │    ┌───►│            frame             ▼   │  │  │
│                 │         │         │     │    │   │  │                       │   │    │    └──────────────────────────────────┘  │  │
│                 │         │         │     └───┬┘   │  │ ┌────────┐ capASIDPool│   │    │                                          │  │
│                 │         │         │         │    │  └─►ASIDPool┼────────────┼───┼────┼───────┐                                  │  │
│                 │         │         │         │    │    └────────┘            │   │    │       │                                  │  │
│                 │         │         │         │    │                          │   │    │       │                                  │  │
│                 │         │         │         │    │   CDT derived            │   │    │       │                                  │  │
│                 │         │         │         │    │ ┌──────────────┐         │   │    │       │                                  │  │
│                 │         │         │         │    │ │  Frame cap   ┼─────────┼───┼────┘       │                                  │  │
│                 │         │         │         │    └─► (with null   │         │   │          ┌─▼────────────────────────────┐     │  │
│                 │         │         │         │      │  mapped info)│         │   │          │                              │     │  │
│                 │         │         │         │      └──────────────┘         │   │          │      ASID Pool            ───┼─────┘  │
│                 │         │         │         │                               │   │          │                            ▲ │        │
│                 │         │         │         │                               │   │          └────────────────────────────┼─┘        │
│                 │         │         │         │                               │   │                                       │          │
│                 │         │         │         │                               │   │                                       │          │
│                 │         │         │         │ ASID id (capFMappedASID       │   │                                       │          │
│                 └─────────▼─────────▼─────────▼───────────────────────────────┼───┼───────────────────────────────────────┘          │
│                                                 or capPTMappedASID) post-map  │   │                                                  │
│                                                                               │   │                                                  │
└───────────────────────────────────────────────────────────────────────────────┘   └──────────────────────────────────────────────────┘
```

After::

```
 Capability Land                                                                     Object / Memory Land
┌───────────────────────────────────────────────────────────────────────────────┐   ┌──────────────────────────────────────────────────┐
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│          ┌──────────────────────────────────────────────────────┐  (hiding ─► │   │                                                  │
│          │                       Untyped                        │   for       │   │                                                  │
│          └┬─────────┬─────────┬─────────┬───────────────────────┘   simplicity)   │                                                  │
│           │         │         │         │                                     │   │                                                  │
│           │         │         │         │CDT tracking                         │   │                                                  │
│           │         │         │         │                                     │   │           hardware memory layouts                │
│           │         │         │         │                                     │   │         ┌──────────────────────────────────┐     │
│           │ ┌────┐  │         │         │                                     │   │         │                                  │     │
│           │ │    ┼──┼─────────┼─────────┼─────────────────────────────────────┼───┼────────►│       page table memory          │     │
│           └─►U.PT│  │         │         │                                     │   │         │  │                               │     │
│             │    │  │         │         │                                     │   │         └──┼───────────────────────────────┘     │
│             └▲───┘  │ ┌────┐  │         │                capPTBasePtr         │   │         ┌──┼───────────────────────────────┐     │
│              │      │ │    ┼──┼─────────┼─────────────────────────────────────┼───┼────────►│  │                               │     │
│              │      └─►U.PT│  │         │                                     │   │         │  ▼    page table memory      │ │ │     │
│       ┌──────┐        │    │  │         │                                     │   │         └──────────────────────────────┼─┼─┘     │
│       │Mapped│        └▲───┘  │ ┌────┐  │                 capFBasePtr         │   │         ┌──────────────────────────────┼─┼─┐     │
│       │  PT  │         │      │ │    ┼──┼─────────────────────────────────────┼───┼────────►│                              │ │ │     │
│       │      ┼──┐  CDT │      └─►U.Fr│  │                                     │   │         │            frame             │ ▼ │     │
│       └─▲────┘  │      │        │    │  │                                     │   │         └──────────────────────────────┼───┘     │
│         │   capPTNext  │        └▲───┘  │ ┌────┐                              │   │         ┌──────────────────────────────┼───┐     │
│         │       │ ┌──────┐       │      │ │    ┼──────────────────────────────┼───┼────────►│                              │   │     │
│    capPT│       └─►Mapped│       │      └─►U.Fr│                              │   │         │            frame             ▼   │     │
│    parent         │  PT  │       │        │    │               other VSpace   │   │         └──────────────────────────────────┘     │
│         └─────────┼      │       │        └▲──▲┘              (paging elided  │   │                                                  │
│                   └▲─────┘       │         │  │              ┬──────┐ for     │   │                                                  │
│                    │    │capPT   │         │  │ CDT          │Mapped│ clarity)│   │                                                  │
│                    │    │  Next  │         │  └──────────────┼Frame │         │   │                                                  │
│                    │    │        │         └───────┐         │      │         │   │                                                  │
│                    │    │  ┌─────┴┐                │         └──────┘         │   │                                                  │
│                    │    └──►Mapped│                │                          │   │                                                  │
│                    │       │Frame │ capFNext       │                          │   │                                                  │
│                    ├───────┼      ┼───┐            │                          │   │                                                  │
│                    │       └──────┘   │            │                          │   │                                                  │
│                    │                  │        ┌───┼──┐                       │   │                                                  │
│                    │                  └────────►Mapped│                       │   │                                                  │
│                    │capFParent                 │Frame │                       │   │                                                  │
│                    └───────────────────────────┼      │                       │   │                                                  │
│                                                └──────┘                       │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
│                                                                               │   │                                                  │
└───────────────────────────────────────────────────────────────────────────────┘   └──────────────────────────────────────────────────┘
```


Changes to riscv64 `structures.bf`.

Before:

```hs
block frame_cap {
    field       capFMappedASID      16
    field_high  capFBasePtr         39
    padding                         9

    field       capType             5
    field       capFSize            2
    field       capFVMRights        2
    field       capFIsDevice        1
    padding                         15
    field_high  capFMappedAddress   39
}
```

After:

```hs
block frame_cap {
    field_high  capFBasePtr         39
    padding                         25

    field       capType             5
    field       capFSize            2
    field       capFVMRights        2
    field       capFIsDevice        1
    padding                         54
}

block mapped_frame_cap {
    field_high  capFNext            37
    padding                         27
    field       capType             5
    -- these are cheap and easy to store and make some operations easier
    field       capFSize            2
    field       capFVMRights        2
    field       capFIsDevice        1
    -- index into parent PT.
    field       capFIndex           9
    padding                         8
    field_high  capFParent          37
}

block mapped_page_table_cap {
    field_high  capPTNext           37
    padding                         27
    field       capType             5
    -- index into the parent PT.
    field       capPTIndex          9
    padding                         13
    field_high  capPTParent         37
}

-- N-level page table
block page_table_cap {
    field_high  capPTBasePtr        39
    padding                         25

    field       capType             5
    padding                         59
}
```




## Drawbacks

-   Each "mapping capability" requires one whole `cte_t` entry, which is 4 words.
    For densely-packed VSpaces, this means one full page table of 512 entries (words)
    requires 512 CNode entries worth of space, or 2048 words worth of space.
    Depending on usage, this could use more memory than shadow page tables.
    In contrast, ASID pools use only a few bits of information per VSpace assigned.

-   There is not enough information on 32-bit arches for this to work: we have
    about 2 words worth of storage in a capability; paddr of caps are currently
    stored as 29 bit pointers (omitting the lower 3 bits for alignment) which
    leaves us with 6 bits to store the index into the HW page table so it can be
    unmapped. This is not enough; nor does this include the `capType` field which
    is at least another 5+ bits.

-   Because of the use of parent/next pointers in the page table mapping caps,
    and the fact that we don't control the cleanup order when untypeds are revoked,
    we can run into O(n²) time complexity during cleanup.

    This might possibly be mitigated if we derive the cap from the page table
    instead of the frame? And then we get say a "page table entry" cap,
    which has a pointer to a child (frame/page table) as doubly-linked list.
    The issue with this method is then we don't have O(1) access to the "parent" -- there
    would instead be up to 512 entries of "page table entries" which are doubly-linked
    and we have to walk backwards to find the parent. But that's still O(n),
    and if we clean up from an arbitrary frame via the CDT that only has to walk
    the O(number of times we mapped that same frame).

    @tsewell's mention of the XOR trick might be able to mitigate this (store
    the XOR of the parent and back pointers; if you have one you can get the other).


## Rationale and alternatives

When a frame is mapped into a page table, the kernel needs to be able to perform the following:

1.  Deleting a mapped frame capability should remove the mapping from any page tables that it was
    mapped into (from the HW's perspective).

2.  Ideally, immediately adjust information stored in capabilities of mapped frames, or
    (as is currently the case) safely[^1] handle invalid data.

3.  Invalidate any address-specific specific HW caches (e.g. TLBs via HW ASID flushes)

Notably we need to be able to do this even if one of the source Untyped caps are
revoked, and the cleanup may happen out of order.

This previously happened via (SW) ASID pools, which uses a (kernel-internal) global ID table
which gets stored as metadata inside every page table and frame, and can be looked up
in order to unmap pages from frames. (This can also store extra information, such as on
arm32's `asid_map_t`). There is not enough storage within a frame/page table capability
to directly store forward/backwards references to the frames/page tables associated
with the information in the hardware.

---

Implementation note: This function needs to work without any extra context to cap finalisation,
and so we must be able to remove the mappings with just the single page_table/frame cap.

```c
finaliseCap_ret_t Arch_finaliseCap(cap_t cap, bool_t final)
{
    finaliseCap_ret_t fc_ret;

    switch (cap_get_capType(cap)) {
    case cap_frame_cap:

        if (cap_frame_cap_get_capFMappedVSpaceId(cap)) {
            unmapPage(cap_frame_cap_get_capFSize(cap),
                      cap_frame_cap_get_capFMappedVSpaceId(cap),
                      cap_frame_cap_get_capFMappedAddress(cap),
                      cap_frame_cap_get_capFBasePtr(cap));
        }
        break;
    case cap_page_table_cap:
        if (final && cap_page_table_cap_get_capPTIsMapped(cap)) {
            /*
             * This PageTable is either mapped as a vspace_root or otherwise exists
             * as an entry in another PageTable. We check if it is a vspace_root and
             * if it is delete the entry from the ASID pool otherwise we treat it as
             * a mapped PageTable and unmap it from whatever page table it is mapped
             * into.
             */
            vspace_id_t vspaceId = cap_page_table_cap_get_capPTMappedVSpaceId(cap);
            findVSpaceForVSpaceId_ret_t find_ret = findVSpaceForVSpaceId(vspaceId);
            pte_t *pte = PTE_PTR(cap_page_table_cap_get_capPTBasePtr(cap));
            if (find_ret.status == EXCEPTION_NONE && find_ret.vspace_root == pte) {
                deleteVSpaceId(vspaceId, pte);
            } else {
                unmapPageTable(vspaceId, cap_page_table_cap_get_capPTMappedAddress(cap), pte);
            }
        }
        break;
```

---

(weak alternative)

-   Could switch around the CDT/cap-information so that we derive from page tables
    the mapping and store the frame in the data, as opposed to derive from
    frame the mapping and store the page table in the data.

    This gives different performance characteristics.

### alternatives

-   There was (apparently) an implementation developed early on in seL4 history
    that instead used shadow page tables in the kernel, which gave us enough
    storage space to track this information. However this also forces higher memory
    usage.

-   @tsewell had some half-thoughts many years ago regarding a limited form
    of kernel allocator for these paging structures.

    Thoughts, from me:
      - Allocator is somewhat like the (prior art's) slab allocation / vma metadata
      - Having smaller pointers allows us to store enough information in the mappings
      - All of the paging structures could be allocated out of a relatively large region
        (the "kernel allocator" which has enough space to track things efficiently and precisely)

A corollary:

  - What purpose do "frames" serve except as page table entries?
  - Newer linux kernels use "folio" (closer to our untyped, i.e. power-of-2-aligned set of pages)
    along with rmap information (see prior art). But it's much more than just 2 words, so it doesn't
    really help us unless we allocate that sort of "shadow" information structure.
    It's possible this is doable in a clean way.


## Prior art

~~N/A, this is seL4-specific.~~ Actually Linux has the exact same problem for swapping.

This is the rmap API in Linux.

Turns out some of the issues are exactly as described in this 2004 paper:

https://www.kernel.org/doc/ols/2004/ols2004v2-pages-71-74.pdf

> A second cost was space. In its original form
> pte_chains cost two pointers per mapping.
> An optimization eliminated the extra structure
> for singly-mapped pages and another optimiza-
> tion added multiple pointers per list entry, but
> the space taken by the pte_chain structures
> was still significant.

... and then they used a pointer to VSpace + Vaddr implementation (more like ASID pools):

> All anonmm structures are linked together by
> fork. A new anonmm structure is allocated
> on exec. The offset stored in struct page
> is the virtual address of the page, while the ob-
> ject pointer points to an anonmm that the page
> is mapped in.

then a modified version:

> The basic mechanism of anon_vma is the
> addition of an anon_vma structure linked to
> each vma that has anonymous pages. The
> anon_vma structure has a linked list of
> all vmas that map that anonymous range.
> The pointer in struct page points to the
> anon_vma and the index is the offset into the
> current mapping

Also see: https://www.kernel.org/doc/gorman/html/understand/understand006.html
which describes the pte_chain implementation (that relies on the slab allocator)
and also an object-based one.

https://github.com/torvalds/linux/blob/master/mm/rmap.c




## Unresolved questions

-   Is this enough of an improvement on ASID pools that makes it worth the
    verification costs?

-   Is this actually correct & can be implemented efficiently?

-   Can this be done on 32-bit architectures where there does not seem to be enough
    bits to store the necessary information in 2 words?




[^1]: Technically speaking, in the current kernel it is possible to map a frame back into the same VSpace using the
      same (software) ASID as before, and then perform flush/cache operations by virtual address using the
      old, stale frame.

      https://github.com/seL4/seL4/pull/1282






## appendix - conversation with Gerwin

[Gerwin]:   the same names get used in a lot more contexts
            and it doesn't help that people have been confused about ASIDs there too
            at least in the x64 proofs, there is definitely a lot of misnamed stuff

[Julia]:    my understanding was we used the software asids purely as a compressed pointer to a vspace

[Gerwin]:   it depends, not necessarily only to a VSpace
            depending on architecture it can point to an asid_map entry, and that can contain more things
            although you could say those are the things that define a vspace

[Julia]:    I mean I don't exactly know why every page table cap needs to know the asid either

[Gerwin]:   the reason the caps have ASIDs is so that you can check that they still point into the same vspace they are supposed to point to
            because it is not possible to find all page caps when you remove a page table
            (for instance)
            the sad thing is that this doesn't really fully work, at least not nicely
            but it does mean you need at least authority to the full VSpace to mess with cap authority inside the VSpace

            The scenario where it does not fully work is the following: you construct a normal VSpace, and you have let's say a page table cap T, and a page cap P where the page is mapped in T.
            Someone (maybe you) revokes T, and the page table needs to be deleted
            there is no cap relationship between T and P, only a page table mapping
            so we can't find P in the delete operation for T

            but, since P stores the ASID and vaddr where it is mapped, it can check for each invocation if the mapping still exists
            so it knows that it is safe to operate

            where it doesn't fully work is when you delete T and the create a new T, and you map a different P' in the same slot
            P now still confers some authority to the mapping of P' even though that is not really intended

[Julia]:    there's never a case where we can prevent revoking without umpapping first?

[Gerwin]:   we can unmap, but we can't find all of the page caps for P (there can be many copies)

            we can only find caps that are derived from each other, but mapping isn't a cap derivation operation and there is not enough storage inside caps to make it one
            The authority problem with P and P' is not a big deal, the max authority you can get that way is to unmap P', but you must already have VSpace authority to do all the rest, so you can do that anyway

            so you're not gaining anything, but it is weird behaviour and not really nice

            we did have a solution for this many years ago with a different implementation (shadow page tables), but it uses a lot more memory
            and we never verified it, because it wasn't really clear if it is better or not
