# Data Layout

## Why Layout Matters

The physical arrangement of data affects:

- cache;
- TLB;
- memory bandwidth;
- GC scanning.

## Struct Size

Larger objects reduce:

- objects per cache line;
- objects per page.

Small changes can cross allocator size classes.

## AoS vs SoA

Array of Structures:

```text
entity entity entity
```

Good when processing complete objects.

Structure of Arrays:

```text
field field field
```

Good when processing one field across many objects.

Choose based on access pattern.

## Hot and Cold Data

Frequently accessed fields should avoid being surrounded by rarely used data.

A hot/cold split can reduce working-set size.

## Pointer Density

Pointer-heavy layouts affect:

- GC scan work;
- cache behavior;
- locality.

Compact representations often improve multiple layers simultaneously.
