---
title: "OR Filter Equality Lookup"
---

# OR Filter Equality Lookup
---------------------------

Overview
--------

Applications often send searches that are one big OR of equality tests on
the same attribute:

    (|(uid=user1)(uid=user2)(uid=user3) ... (uid=user1000))

This is how a sync job fetches a batch of accounts, how an inventory system
checks a list of hostnames, and how an org chart tool finds everyone who
reports to a set of managers.

These searches are usually fully indexed, and the index side is quick. The
slow part hides somewhere else. Before returning an entry the server has to
prove the entry really matches the filter, and that filter test walked the
OR from left to right, every branch, for every candidate. Each branch
scanned the entry's attributes and normalized its values all over again.
With 1000 branches and 1000 candidates, that's a million branch evaluations
for a single search.

The server now builds a sorted lookup table from the OR's assertion values,
once per search. Each candidate entry normalizes its values once and finds
the matching branch with a binary search. On a 100,000 entry test database a
355-branch search over 612 candidates went from 1.7 seconds to 0.18 seconds,
and a 10,000 candidate case went from 23 seconds to 0.4 seconds. Search
results are identical, and anything the table can't handle simply falls back
to the old code.

Use Cases
---------

-   The case from issue 6275: an application keeps its user list in sync
    with the directory and fetches all the accounts it cares about in one
    search, a thousand uids in a single OR:

        (|(uid=jsmith)(uid=mjones)(uid=bwilson)...)

-   An inventory or configuration management system keeps its machines in
    LDAP and checks a batch of hostnames in one query:

        (|(cn=web01.example.com)(cn=web02.example.com)(cn=db03.example.com)...)

-   A backup or quota tool walks a filesystem, collects the numeric owners
    it found, and maps them back to accounts:

        (|(uidNumber=1004)(uidNumber=1027)(uidNumber=1263)...)

Clients that send small ORs, like SSSD with its handful of branches, aren't
affected. The lookup doesn't even wake up below 16 branches; a table
wouldn't pay for itself there.

Design
------

The backend creates its own normalized copy of the search filter at the
start of each search. After that copy is made, the server walks it and looks
at every OR node. When an OR contains at least 16 equality branches on the
same attribute, and the attribute qualifies, the server builds a sorted
table of the branch values and attaches it to that OR node.

During the per-entry filter test, an OR node that has a table does not walk
its branches. The entry's values are normalized once, with the same
normalizer that was applied to the branch values, and looked up in the
table.

The table only decides which branch to test; it's a shortcut to the right
branch, never a substitute for testing it. When a value is found in the
table, the server still runs the normal access check and the normal matching
call for that branch. A branch the client is not allowed to read still never
counts as a match, which has been the rule since ticket 48275. If no table
branch matches, branches outside the table (other attributes, substrings,
and so on) are evaluated the old way.

An attribute qualifies when:

-   its syntax is Directory String, IA5 String, Integer, Numeric String,
    Telephone Number, or DN. For these, two values are equal exactly when
    their normalized forms are the same bytes.
-   its equality matching rule is one of the eight standard rules
    (caseIgnoreMatch, caseExactMatch, caseIgnoreIA5Match, caseExactIA5Match,
    integerMatch, numericStringMatch, telephoneNumberMatch,
    distinguishedNameMatch).
-   the filter uses the plain attribute, without options. `uid` qualifies,
    `uid;lang-en` does not.

When an OR mixes several attributes, the largest qualifying group gets the
table and the remaining branches are evaluated normally. On a tie, the group
that appears first in the filter wins.

The old code runs unchanged when:

-   the OR has fewer than 16 qualifying branches
-   the attribute is not in the schema, or uses generalizedTime, Boolean,
    Bit String, or Name And Optional UID syntax
-   the matching rule is a collation rule or comes from a plugin
-   the search is a tombstone or RUV search
-   the attribute may be supplied by CoS or Roles for the entry being tested
-   a DN-valued entry has more values than the table has keys, where the old
    code is cheaper
-   a stored DN value cannot be parsed
-   the evaluation is an access-only pass, which must visit every branch

One more restriction, and it's the subtle one. A filter branch can evaluate
to true, false, or undefined, where undefined usually means "you're not
allowed to read this attribute". Normal searches treat false and undefined
the same, so when all table lookups miss, the server can mark the entry as a
non-match right away.
NOT and VLV are the two places that can tell false and undefined apart, so
under a NOT, and in VLV searches, this shortcut is disabled and the old walk
decides.

One side effect: the server no longer asks the ACL plugin about branches
whose values the entry does not have. This is only visible with ACL debug
logging enabled, where large OR searches now produce far fewer lines.
Nothing changes about who can see what.

Implementation
--------------

New file `ldap/servers/slapd/filter_or_lookup.c` holds the eligibility
checks, the table build, the probe, and the free function. The per-entry
lookup path is `vattr_test_filter_or_lookup()` in
`ldap/servers/slapd/filterentry.c`. The build is called from the filter
pre-digest block in `ldap/servers/slapd/back-ldbm/ldbm_search.c`. The
configuration attribute is wired in `libglobs.c` like the other `cn=config`
switches.

The table hangs off the OR node of the backend's private filter copy and is
freed together with it in `slapi_filter_free()`. `slapi_filter_dup()` does
not copy tables. Filters parsed for other purposes (plugins, persistent
search, ACL evaluation, `cn=config` searches) never get tables. The table is
read-only after the build, so later pages of a paged search can read it
safely.

Memory cost is about 40 bytes per branch for the lifetime of the search. A
10,000-branch OR allocates around 400 KB. There is no upper limit on the
table size beyond the existing limits on incoming requests; in the
measurements below, even a 1000-branch table built for a search with no
candidates cost nothing visible.

Configuration
-------------

    dn: cn=config
    nsslapd-enable-or-filter-lookup: on

The default is on. The setting is dynamic and needs no restart:

    dsconf <instance> config replace nsslapd-enable-or-filter-lookup=off

The server reads the setting when it prepares a search's filter, so a change
applies to new searches. A paged search that's already running keeps its
tables until the client finishes paging; don't expect a search in flight to
change course.

Troubleshooting
---------------

One line is logged when tables are built, at a debug level that is off by
default. Add 524288 (SLAPI_LOG_BACKLDBM) to `nsslapd-errorlog-level` and
look for:

    DEBUG - ldbm_back_search - OR filter equality lookup engaged: 1 node(s), largest 355 branches

The line means tables were built for the search, nothing more. It doesn't
say how many entries used them. There's no counter in `cn=monitor` and no
`notes=` keyword in the access log.

To confirm the lookup is really deciding, also enable filter logging (add
32) and check that the per-branch `test_ava_filter - => AVA:` lines
disappear for the attribute. Do that on a test system; at these log levels a
busy server spends more time logging than searching.

If the feature does not engage, check that the setting is on, that the OR
has at least 16 equality branches on one attribute, that the attribute's
syntax and equality matching rule are in the supported lists above, and that
the search is not a tombstone or RUV search.

Performance - To Be Updated With More Data
-----------

Measured on Fedora with a 100,000 entry database. Every number is the median
of 15 searches. Both runs used the same installed package and the same
indexes; only `nsslapd-enable-or-filter-lookup` was changed.

| Search | off | on |
|--------|----:|---:|
| 355 DN branches, 612 candidate entries, no matches | 1.73 s | 0.18 s |
| the same filter over 10,000 candidates | 23.2 s | 0.39 s |
| 1000 branches, more than half values present | 2.92 s | 0.29 s |
| the only match on the last of 355 branches | 3.84 s | 0.20 s |
| 128 branches, all values present | 0.26 s | 0.17 s |

The saving grows with the number of candidates times the number of
branches. Below about 64 branches the difference is within measurement
noise. Searches with no candidates measure the same even with a 1000-branch
table, so building a table that goes unused costs nothing visible. Nothing
got slower beyond noise, and that's the part we watched most closely.

For context, packaged OpenLDAP on the same data ran the 10,000 candidate
case in 0.85 seconds; 389 DS with the lookup ran it in 0.39 seconds.
Different servers do different work, so read that as context, not as a
scoreboard.

Testing
-------

-   `dirsrvtests/tests/suites/filter/filter_or_lookup_test.py`: the same
    searches with the feature on and off must return the same results.
    Covers all supported matching rules, non-ASCII values, escaped DNs,
    thresholds 15 and 16, mixed attribute groups, subtypes, CoS, referrals,
    paging, VLV with server-side sorting, and proxy authorization.
-   `filter_or_lookup_feature_test.py`: reads the debug log to prove the
    lookup engaged, fell back, or was disabled where expected.
-   `filter_or_union_test.py` and
    `filter_large_filter_interaction_test.py`: OR result sets across
    `idl_set` consumers, and large DN filters combined with substring and
    approximate branches.
-   `memory_leaks/filter_not_first_lifecycle_test.py`: filter ownership
    under ASan across startup, VLV, paged, and cancelled operations. Needs a
    dedicated ASan build and is not part of the default CI run.

Replication
-----------

No impact. The change is confined to search evaluation.

Updates and Upgrades
--------------------

Nothing to migrate. The attribute has a default, needs no schema or
`dse.ldif` changes, and upgraded servers get the feature enabled. Setting it
to off restores the previous behavior exactly.


Origin
------

https://github.com/389ds/389-ds-base/issues/7664

Author
------

<simon.pichugin@gmail.com>
