# Refill Pipe

Compared to the YanqiHu architecture, the NanHu architecture adds a dedicated backfill pipeline Refill Pipe. For the miss request of load or store, the replacement path will be selected before entering the Miss Queue. After getting the data returned by L2, it can be sent to the Refill Pipe. Therefore, only one beat is needed to write the backfill data to the DCache, without accessing the DCache again.

## Read-write conflict between Refill Pipe and Main Pipe

Both the Refill Pipe and the Main Pipe will write to the DCache. The Refill Pipe is completed in one beat, while the Main Pipe needs to go through four stages of pipelines such as reading tags and reading data before writing data to the DCache. In order to ensure the consistency of the read and write data in the Main Pipe, that is, the read and write will not be interrupted by the write operation of the Refill Pipe, the Refill Pipe will be blocked and temporarily unable to write to the DCache in the following cases:

* There is a set conflict between the request of the Refill Pipe and the Stage 1 of the Main Pipe;
* The Refill The request of Pipe and Stage 2 / Stage 3 of Main Pipe have set conflicts and have the same way enable signal `way_en`.

The Main Pipe will complete tag comparison in Stage 1 and get `way_en`, but due to the tight timing, the blocking strategy of Main Pipe Stage 1 for Refill Pipe is relaxed here, and only blocking based on set is required.
