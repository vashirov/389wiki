---
title: "Entry Cache Improvements"
---

{% include toc.md %}

Entry Cache Improvements
-----------------------------

## Why Entry Cache?

We have a different disk storage of entries compared to memory reperesentation. Effecient and reliable
disk storage is different to our an effective in memory format. This also allows us to change the
in memory format without affecting on disk content.

The trade however is that we must perform CPU work to convert between the disk and memory formats
on loads and writes. During writes, which is a smaller part of our work load, this is not a major
concern, but with high concurrent read numbers, this can be very intensive.

This is why we cache the deserialised entries into ram.

More than than, by having an entry cache, we are able to control where and how our data is cached,
where as relying on DB cache, we don't have control of this, and worse, relying on the VFS when a
system comes under memory pressure, VFS pages will be evicted earlier than application memory.
Additionally, the VFS is an LRU which has a number of issues with invalidation patterns. Systems
like LMDB which rely on the VFS as a cache are design for effective disk to VFS transfer, but not
effective CPU cache operations and alignments (which is more important in a concurrent system).

To this end, having an entry cache allows us to control and improve this in our application.

## Cache challenges

As our system is concurrently readable (many parallel readers and a single seralised writer), there
has been no previous work on caches for this kind of system, meaning that some consistency guarantees
may not have been met by previous designs.

Cache strategies like LRU/LFU have many weaknesses related to invalidation patterns that can occur.

The current Entry cache is a custom HashMap with LRU and entries have watermarks to define if they
have been invalidated due to a write. It is not transactional, and requires a single mutex to protect
it.

## To Do

Add probes, find the true bottle necks

## Possible Improvements:

- Change entry cache algorithm ( https://github.com/Firstyear/concread/blob/master/CACHE.md )
- Change Slapi_Entry structure to be more efficient
- Change id2entry entry format (Ldif to Cbor)
- Cache IDLs to avoid BDB hits.
- IDL Compression Library ( https://github.com/Firstyear/idlset )
- Would be great to reduce the entry cache entry's size so more entries can fit into the same amount of memory
- Make duplicating an entry less expensive/reduce the number of duplications

### Slapi_Entry

```
struct slapi_entry
{
    struct slapi_dn e_sdn;        /* DN of this entry */
    struct slapi_rdn e_srdn;      /* RDN of this entry */
    char *e_uniqueid;             /* uniqueID of this entry */
    CSNSet *e_dncsnset;           /* The set of DN CSNs for this entry */
    CSN *e_maxcsn;                /* maximum CSN of the entry */
    Slapi_Attr *e_attrs;          /* list of attributes and values   */
    Slapi_Attr *e_deleted_attrs;  /* deleted list of attributes and values */
    Slapi_Vattr *e_virtual_attrs; /* cache of virtual attributes */
    uint32_t e_virtual_watermark; /* indicates cache consistency when compared
                                     to global watermark */
    Slapi_RWLock *e_virtual_lock; /* for access to cached vattrs */
    void *e_extension;            /* A list of entry object extensions */
    unsigned char e_flags;
    Slapi_Attr *e_aux_attrs;      /* Attr list used for upgrade */
};
```

#### Child Structures

```
struct slapi_dn
{
    unsigned char flag;
    const char *udn; /* DN [original] */
    const char *dn;  /* Normalised DN */
    const char *ndn; /* Case Normalised DN */
    int ndn_len;     /* normalized dn length */
};

struct slapi_rdn
{
    unsigned char flag;
    char *rdn;
    char **rdns;       /* Valid when FLAG_RDNS is set. */
    int butcheredupto; /* How far through rdns we've gone converting '=' to '\0' */
    char *nrdn;        /* normalized rdn */
    char **all_rdns;   /* Valid when FLAG_ALL_RDNS is set. */
    char **all_nrdns;  /* Valid when FLAG_ALL_NRDNS is set. */
};

struct csnset_node
{
    CSNType type;
    CSN csn;
    CSNSet *next;
};

struct slapi_attr
{
    char *a_type;
    struct slapi_value_set a_present_values;
    unsigned long a_flags;        /* SLAPI_ATTR_FLAG_... */
    struct slapdplugin *a_plugin; /* for the attribute syntax */
    struct slapi_value_set a_deleted_values;
    struct bervals2free *a_listtofree; /* JCM: EVIL... For DS4 Slapi compatibility. */
    struct slapi_attr *a_next;
    CSN *a_deletioncsn;                  /* The point in time at which this attribute was last deleted */
    struct slapdplugin *a_mr_eq_plugin;  /* for the attribute EQUALITY matching rule, if any */
    struct slapdplugin *a_mr_ord_plugin; /* for the attribute ORDERING matching rule, if any */
    struct slapdplugin *a_mr_sub_plugin; /* for the attribute SUBSTRING matching rule, if any */
};

struct _entry_vattr
{
    char *attrname;   /* if NULL, the attribute name is the one in attr->a_type */
    Slapi_Attr *attr; /* attribute computed by a SP */
    struct _entry_vattr *next;
};
```

#### slapi_entry_dup()



    1.  ec = slapi_entry_alloc();
        - 4 mallocs

    2.  slapi_entry_init(ec, NULL, NULL);
        - 1 malloc

    3.  slapi_sdn_copy(slapi_entry_get_sdn_const(e), &ec->e_sdn);
        - 3 mallocs

    4.  slapi_srdn_copy(slapi_entry_get_srdn_const(e), &ec->e_srdn);
        - 5 frees
        - 1 malloc
        - 3 array mallocs (malloc * rdn's)

    5.  ec->e_dncsnset = csnset_dup(e->e_dncsnset);
        - 1 malloc per CSN in set

    6.  ec->e_maxcsn = csn_dup(e->e_maxcsn);
        - 1 malloc

    7.  if (e->e_uniqueid != NULL) { ec->e_uniqueid = slapi_ch_strdup(e->e_uniqueid); }
        - 1 malloc

    8.  for (a = e->e_attrs; a != NULL; a = a->a_next) { Slapi_Attr *newattr = slapi_attr_dup(a); }
        - per attribute: 
            - 2 mallocs
            - 4 mallocs per value
            - 1 free (potentially in value sorting)


    9.  for (a = e->e_deleted_attrs; a != NULL; a = a->a_next) { Slapi_Attr *newattr = slapi_attr_dup(a); }
        - same as above
        - per attribute: 
            - 2 mallocs
            - 4 mallocs per value
            - 1 free (potentially in value sorting)

    10. for (aiep = attrs_in_extension; aiep && aiep->ext_type; aiep++) { aiep->ext_copy(e, ec); }
        - 1 malloc
    


20 mallocs + 150-200 mallocs for attribute dupping





























