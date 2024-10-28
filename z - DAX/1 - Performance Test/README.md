# About

This DAX code Slowly calculates PI and can be use to check the performance of your machine.

The method selected to use was the Gregory-Leibniz series, it if far fom optimal, however is the simplest to implement in DAX (however slow, with in this case is good). In the implementation bellow it iterates 9.999.999 times (do not change if you whant yo compare with the results I got bellow).

```
Calc_Pi_Slow = 
SUMX(
    ADDCOLUMNS(
        GENERATESERIES(1, 9999999, 2),
        "Val", 4 / [Value] * IF(ISODD(( [Value] + 1 ) / 2), 1, -1)
    ),
    [Val]
)
```

# Results I got

**TIP**: Take at least 5 measures from Performance Analyzer
- A Great PC ~   2400 ms - AMD Ryzen 7 5700X 8-Core Processor | 24 GB | SSD
- A Good  PC ~   3700 ms - Intel(R) Xeon(R) Platinum 8370C CPU @ 2.80GHz | 64 GB | Probably SSD
- A Bad   PC ~ +16000 ms - Inter(R) Core(TM) i3-2370M CPU @ 2.40GHz | 8 GB | SSD