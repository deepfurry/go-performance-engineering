# TLB and Pages

## Virtual Memory

Applications access virtual addresses.

The CPU translates them through page tables.

The TLB caches these translations.

## TLB Pressure

Random access across many pages increases translation overhead.

Compact layouts improve:

- cache locality;
- page locality;
- TLB efficiency.

## Engineering Impact

Data layout affects more than cache.

A design with fewer scattered allocations may also reduce page and translation overhead.
