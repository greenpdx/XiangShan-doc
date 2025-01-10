# DataStorage

The `DataStorage` module is responsible for the storage and reading and writing of cache data.
According to the characteristics of each Tilelink channel, the `DataStorage` module has 2 read ports (`sourceC_r`, `sourceD_r`),
3 write ports (`sinkD_w`, `sourceD_w`, `sinkC_w`).
In order to improve the concurrency of reading and writing, the module can parameterize the number of internal banks, and different banks can read and write in parallel.
In addition, the module also supports parameterized overclocking of the data read from SRAM to achieve higher frequency requirements.
