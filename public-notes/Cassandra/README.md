# Cassandra issues

## ```Maximum memory usage reached, cannot allocate chunk```, shortly followed by ```java.lang.OutOfMemoryError: Map failed```

### Solution

The Linux kernel parameter vm.max_map_count was set to the default value of 65535. This was considerably smaller than Datastax's recommended value of 1048575. Set the kernel parameter vm.max_map_count to the recommended value.

On CentOS 7:
1. Add the following line to /etc/sysctf.conf: ```vm.max_map_count = 1048575```
2. Run the following command as a privileged user: ```sysctl -p```
3. Restart the Cassandra process on the affected box.

### Background:

My C* cluster was affected by this issue for a number of weeks, where it caused a node to crash every couple of hours during regular operation (compaction etc.), and some other nodes to crash too when a nodetool repair was run on the cluster (which I did not investigate fully - I supposed this was due adjacent nodes becoming stressed by the repair operation, but I had figured this on the assumption that the cluster was having issues with JVM heap memory and physical memory).

Although I increased physical memory apportioned to the nodes and also doubled and quadrupled heap memory, this did not solve the issue. I also attempted to up the C\* parameter *file_cache_size_in_mb*, but this did not work either as the *file_cache_size_in_mb* would always be exceeded (the partitions causing this issue are extremely large), and the node would crash shortly afterwards with an mmap error.

After investigation, I found that the cluster had most of the [Datastax recommended production settings](https://docs.datastax.com/en/dse/6.0/dse-admin/datastax_enterprise/config/configRecommendedSettings.html), except for the kernel parameter vm.max_map_count being set to a value much higher than the default. This value determines "the maximum number of memory map areas a process may have" (according to [Linux doco](http://kernel.org/doc/Documentation/sysctl/vm.txt)). Since changing this to the recommended setting, the cluster has been operating normally, and currently has a repair running (whereas previously it would have crashed out fairly quickly).

It goes without saying that this problem could have been avoided by having a schema that didn't allow for such long partitions - this however does bring the cluster into line with recommended productions settings, and also buys a little time to work about fixing up the schema.
