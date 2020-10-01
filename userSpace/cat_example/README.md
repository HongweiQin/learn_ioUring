# Cat With Libiouring

Let's write a userspace program that displays user selected
file contents.

Syntax:

	cat [filename1] <[filename2] ...>

We only create one submission queue with only one queue
depth. We read the whole file using one vector read request.

Reference:
[https://unixism.net/loti/tutorial/cat_liburing.html]
(https://unixism.net/loti/tutorial/cat_liburing.html)

## Methodology

1. Create a SQ by `io_uring_queue_init()`.
1. For each file
	1. Open the file.
	1. Get a submition queue entry (sqe) by calling `io_uring_get_sqe()`.
	1. Compose a vector read request by using `io_uring_prep_readv()`.
The request tries to read the whole file.
	1. Link the sqe to the request by calling `io_uring_sqe_set_data()`.
	1. Submit the sqe by calling `io_uring_submit()`.
	1. read the whole file using a vector read request. 
	1. Wait for completion by calling `io_uring_wait_cqe()`. After this
step, we'll get a completion queue entry (cqe) that shows the request
completion information.
	1. Print out the file.
	1. Call `io_uring_cqe_seen()` to tell the kernel that we've
consumed the cqe.
1. Destroy the SQ by `io_uring_queue_exit()`.


## Limitations

1. The file can not be largar than 1MiB. Otherwise, the number of vectors
will overflow the permitted maximum and the async readv will fail.

	```
	$ dd if=/dev/zero of=./test2M bs=1K count=2048
	$ ./cat_simple_example test2M
			Async readv failed. err=-22
	```
1. We have only one submittion queue (SQ) and the SQ has only one entry.
This prevents us from taking advantage of parallelism.

	```
	Note that currently I don't know whether the vectors in the readv can be
	executed in parallel. I think the answer should be filesystem specific.
	My guess is that if the reads hit the filesystem page cache,
	they are copied to the user buffers sequentially. But if they
	all miss the page cache, the bios for disk I/O reads can be parallelized.
	I'll figure it out later.
	```
2. The file read and print process cannot overlap.
3. Multiple files can not be read simultaneously.