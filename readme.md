# Report For GSoC 2022

## Basic information

Project : MariaDB ColumnStore MCOL-4995

This project goal is to implement basic vectorized filtering at ARM platforms for Mariadb Column Store Engine.

## Introduction

During my participation in the Mariadb as a GSoC2022 student.I have made serveral tasks.

* Implement vectorized filtering for MCS. This is the main task.I use serveral ways to implement it.
  
  I first used the arm neon to implement it.Because MCS already has vectorized filtering which uses SSE.I need to unify arm neon and SSE to provide the same interface for upper layer.Due to that ,it has a little performance loss.If we want to unify the behavior of simdProcessor, we have to make concessions.It is a trade-off.
  
  We get some excellent result from micro benchmark.
  
  ```cpp
  Before my work.
  -----------------------------------------------------------------------------------------------------
  Benchmark                                                           Time             CPU   Iterations
  -----------------------------------------------------------------------------------------------------
  FilterBenchFixture/BM_ColumnScan1ByteVectorizedCode             82923 ns        82925 ns         8351
  FilterBenchFixture/BM_ColumnScan1Byte1FilterVectorizedCode     117027 ns       117044 ns         5949
  FilterBenchFixture/BM_ColumnScan2ByteTemplatedCode              41096 ns        41098 ns        17019
  FilterBenchFixture/BM_ColumnScan2Byte1FilterTemplatedCode       58552 ns        58570 ns        11999
  FilterBenchFixture/BM_ColumnScan2Byte1FilterVectorizedCode      58474 ns        58490 ns        11934
  FilterBenchFixture/BM_ColumnScan4ByteTemplatedCode              21984 ns        21986 ns        31855
  FilterBenchFixture/BM_ColumnScan4ByteVectorizedCode             29395 ns        29413 ns        23626
  FilterBenchFixture/BM_ColumnScan8ByteTemplatedCode              10895 ns        10896 ns        64333
  FilterBenchFixture/BM_ColumnScan8Byte1FilterTemplatedCode       14439 ns        14453 ns        47839
  FilterBenchFixture/BM_ColumnScan8ByteVectorizedCode             10841 ns        10842 ns        63192
  FilterBenchFixture/BM_ColumnScan8Byte1FilterVectorizedCode      14414 ns        14428 ns        48412
  
  After the vectorized filtering has been implemented.
  -----------------------------------------------------------------------------------------------------
  Benchmark                                                           Time             CPU   Iterations
  -----------------------------------------------------------------------------------------------------
  FilterBenchFixture/BM_ColumnScan1ByteVectorizedCode              8172 ns         8173 ns        85692
  FilterBenchFixture/BM_ColumnScan1Byte1FilterVectorizedCode       8975 ns         8992 ns        77879
  FilterBenchFixture/BM_ColumnScan2ByteTemplatedCode               5041 ns         5049 ns       138703
  FilterBenchFixture/BM_ColumnScan2Byte1FilterTemplatedCode        5775 ns         5791 ns       120794
  FilterBenchFixture/BM_ColumnScan2Byte1FilterVectorizedCode       5770 ns         5783 ns       121063
  FilterBenchFixture/BM_ColumnScan4ByteTemplatedCode               4084 ns         4086 ns       171615
  FilterBenchFixture/BM_ColumnScan4ByteVectorizedCode              4434 ns         4448 ns       157333
  FilterBenchFixture/BM_ColumnScan8ByteTemplatedCode               2779 ns         2781 ns       252110
  FilterBenchFixture/BM_ColumnScan8Byte1FilterTemplatedCode        3643 ns         3656 ns       191413
  FilterBenchFixture/BM_ColumnScan8ByteVectorizedCode              2781 ns         2782 ns       251334
  FilterBenchFixture/BM_ColumnScan8Byte1FilterVectorizedCode       3640 ns         3656 ns       191375
  ```

       As we can see,the performance has been improved a lot.And for a such query        " select * from tablename where colname='number' " at  dataset cardinality is 1        000 000 000.Its performance is improved by approximately 40%.

       The related Prs are the follows.

        [MCOL-4995 Research/implement basic vectorized filtering for ARM platforms](https://github.com/mariadb-corporation/mariadb-columnstore-engine/pull/2427)

        [Support_max_min](https://github.com/mariadb-corporation/mariadb-columnstore-engine/pull/2458)

        Beyond those,I also use a different way that is suitable for arm neon. And I  get         more excellent result from that.It is improved about 2x than before according to         micro benchmark.



* ```cpp
  FilterBenchFixture/BM_ColumnScan1ByteTemplatedCode             129939 ns       129946 ns         5390
  FilterBenchFixture/BM_ColumnScan1Byte1FilterTemplatedCode      131282 ns       131301 ns         5328
  FilterBenchFixture/BM_ColumnScan1ByteVectorizedCode              8172 ns         8173 ns        85692
  FilterBenchFixture/BM_ColumnScan1Byte1FilterVectorizedCode       8975 ns         8992 ns        77879
  FilterBenchFixture/BM_ColumnScan2ByteTemplatedCode               5041 ns         5049 ns       138703
  FilterBenchFixture/BM_ColumnScan2Byte1FilterTemplatedCode        5775 ns         5791 ns       120794
  FilterBenchFixture/BM_ColumnScan2Byte1FilterVectorizedCode       5770 ns         5783 ns       121063
  FilterBenchFixture/BM_ColumnScan4ByteTemplatedCode               4084 ns         4086 ns       171615
  FilterBenchFixture/BM_ColumnScan4ByteVectorizedCode              4434 ns         4448 ns       157333
  FilterBenchFixture/BM_ColumnScan8ByteTemplatedCode               2779 ns         2781 ns       252110
  FilterBenchFixture/BM_ColumnScan8Byte1FilterTemplatedCode        3643 ns         3656 ns       191413
  FilterBenchFixture/BM_ColumnScan8ByteVectorizedCode              2781 ns         2782 ns       251334
  FilterBenchFixture/BM_ColumnScan8Byte1FilterVectorizedCode       3640 ns         3656 ns       191375
  ```



        The related code is [here](https://github.com/mariadb-corporation/mariadb-columnstore-engine/compare/develop...NTH19:mariadb-columnstore-engine:change_mask_arm_neon).

        And I also use SVE to implement vectorized filtering.But the performance is worse         than arm neon.        

* optimize string comparasions for primproc.

    For Eq comparasions in p_Dictionary function of pp,the way uses hashmap is faster     than for loop, but when the filtercount is less than a magic  number N, the for loop     method is faster.And I have simply found that N is 6,but it is just an estimate.

<img title="" src="file:///C:/Users/Administrator/Desktop/shixi/Report/images/186296661-91d10d3d-7154-426b-afa4-54077aa0a211.png" alt="s" width="682" data-align="right">

    Accroding to this micro benchmark.The time complexity of for loop is O(n).We can      find that when filtercount is 3,the time for loop method is half of eqFilter. So when     filtercount is less than 6,the way uses for loop benefits a lop to performance.

    The related Pr is [here](https://github.com/mariadb-corporation/mariadb-columnstore-engine/pull/2525)



* Bug fixes
  
  [Fix micro benchmark](https://github.com/mariadb-corporation/mariadb-columnstore-engine/pull/2543)

## Further Work

* Try  MCOL-4783( Propagate LIMIT into TupleBPS to reduce the number of simultaneous Primitive Jobs issued by TBPS)

## Appreciation to mentor

I want to express my thanks to [drrtuy (Roman Nozdrin) ](https://github.com/drrtuy)for mentoring and helping me.


