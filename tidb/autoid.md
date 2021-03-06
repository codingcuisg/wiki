* For auto increment column, TiDB does not guarantee continuity, it only offers
  incremental and unique constraint;
* each TiDB instance would cache 30000 ids locally, if used up, then allocate another
  30000; inside each instance, or each statement, the continuity is not guaranteed either,
  because rowID is allocated using autoid facility, so we may see output like:
  ```
  create table bar2 (a int auto_increment, key (a));
  insert into bar2 () values ();
  insert into bar2 () values ();
  insert into bar2 () values ();
  insert into bar2 () values (), (), ();
  select * from bar2;
  +------+
  | a    |
  +------+
  |    1 |
  |    3 |
  |    5 |
  |    7 |
  |    8 |
  |    9 |
  +------+
  ```
  @sa tableCommon::AddRecord -> tableCommon::AllocAutoID -> allocator::Alloc
  the change of auto id value would be written to storage

* there is another struct `memoryAllocator`, it involves no storage write, hence
  not crash-safe;
