## 1. NUMA

### NUMA: None-Uniform Memory Access

**Vấn đề**: CPU hoạt động nhanh hơn RAM rất nhiều => Khi xử lý lượng dữ liệu lớn trên nhiều CPUs thì các CPUs sẽ phải chờ để lấy dữ liệu từ RAM vào, thời gian truy cập RAM phụ thuộc vào vị trí mà CPU cần truy cập tới.

**NUMA architecture** được thiết kế để sử dụng trong `multilprocessing`: với tư tưởng là cung cấp RAM riêng biệt cho từng CPU hoặc 1 nhóm các CPUs, nhằm: `avoiding the performance hit when several processors attempt to address the same memory.` Mặc định các CPUs nằm trên cùng 1 socket sẽ được chia vào cùng 1 NUMA nodes (kiểm tra bằng lệnh `lscpu | grep NUMA`), và sử dụng chung 1 phần RAM (gọi là local RAM).  Cụ thể là khi ta cắm RAM thì cần cắm đều vào 2 khay RAM ở 2 bên:

![numa](http://www.ksingh.co.in/images/NUMA.png)

**Vấn đề của NUMA**: trong trường hợp dữ liệu cần được xử lý trên nhiều thread, thì khi đó sẽ có các CPU trên các NUMA node khác nhau cần truy cập vào cùng 1 dữ liệu đó. Để xử lý vấn đề này, giữa các NUMA nodes sẽ có thêm `hardware hoặc software` để vẫn chuyển dữ liệu giữa các local RAM của các NUMA node. Hoạt động share dữ liệu này diễn ra chậm => ảnh hưởng tới performance. => Chỉ phù hợp với 1 số ứng dụng.

> For a NUMA system, the best performance is achieved when a process and its memory reside in the same NUMA node.

> **Note**: if the compute nodes have NUMA architecture, then it is a good idea to enable NUMA and leverage the operating system and hypervisor to use CPU pinning. 

Ngoài ra còn 1 số khái niệm:

**SMP - Symmetric Multi-Processing**: tương ứng với số lượng `physical cores` trên host.

**SMT CPUs- Simultaneous Multi-Thread CPUs**: tương ứng với số lượng `threads` mà 1 physical core support (thường là 2).

> **Hiện tại:** có [bug nova 1289064](https://bugs.launchpad.net/nova/+bug/1289064) và [specs](https://specs.openstack.org/openstack/nova-specs/specs/rocky/approved/numa-aware-live-migration.html) liên quan tới việc `live-migrate` khi sử dụng NUMA topology , đã được fix [ở đây](https://review.opendev.org/#/c/625880/) và [ở đây](https://review.opendev.org/#/c/640462/) => Nếu sử dụng NUMA và CPU pinning thì cần cherry pick về.

### Thực hiện
Kiểm tra NUMA nodes trên compute host
```
# lscpu  | grep NUMA
NUMA node(s):          2
NUMA node0 CPU(s):     0-7,16-23
NUMA node1 CPU(s):     8-15,24-31
```

Add thêm `NUMATopologyFilter` vào nova-scheduler:

```
# cat nova-scheduler.conf 
[DEFAULT]
cpu_allocation_ratio = 4.0
ram_allocation_ratio = 1.1

[scheduler]
driver = filter_scheduler

[filter_scheduler]
available_filters = nova.scheduler.filters.all_filters
enabled_filters = RetryFilter, AvailabilityZoneFilter, ComputeFilter, AggregateInstanceExtraSpecsFilter, ImagePropertiesFilter, ServerGroupAntiAffinityFilter, ServerGroupAffinityFilter, AggregateCoreFilter, AggregateRamFilter, AggregateIoOpsFilter, AggregateImagePropertiesIsolation, NUMATopologyFilter
host_subset_size = 6
disk_weight_multiplier = 0
```

Set metadata cho flavor:
- `hw:numa_nodes=1` để đảm bảo các vCPUs của VMs nằm trên `cùng NUMA node`
- `hw:numa_nodes=2` để buộc các vCPUs của VMs nằm trên cả `2 NUMA nodes`
```
# restrict an instance’s vCPUs to a single host NUMA node
openstack flavor set NUMA_8vCPU-16GB_RAM-50_Disk --property hw:numa_nodes=1
```

**Tạo 2 VMs có cùng 8 vCPUs, 16GB RAM, trong đó 1 trên flavor enabled numa và 1 không enabled.** Kiểm tra xml:
```
openstack server create --flavor 9cd0e9ae-6de0-4f16-80f2-525e5836ff08 --image IMAGE_ID --key-name KEY_NAME \
  --user-data USER_DATA_FILE --security-group SEC_GROUP_NAME --property KEY=VALUE \
  INSTANCE_NAME
```

## 2. CPU pinning

### Định nghĩa CPU pinning

Mặc định thì các processes trên vCPUs của các máy ảo không được chỉ định cho host CPUs cụ thể nào, mà sẽ được phân vào các host CPUs khác nhau giống như các process bình thường. Đây chính là tính năng giúp ta có thể config `overcommitting CPUs`. Nhưng đối với 1 số workload cần `real-time` thì ta không thể sử dụng chế độ sharing CPUs này được do `latency`.

> Khi sử dụng CPU pinning thì cần tạo riêng 1 zone (host aggregate) 

Ta có thể config `cpu_policy` trên `flavor` hoặc cho `image`. Chỉ config cpu_policy trên flavor `hoặc` image hoặc phải config trên cả 2 giống nhau. Nếu config `cpu_policy` trên flavor và image khác nhau thì sẽ bị exception.

```
openstack flavor set NUMA_8vCPU-16GB_RAM-50_Disk --property hw:cpu_policy=dedicated
```

Để config các vCPUs nằm trên các `thread sibling` của host CPU có SMT (Simultaneous Multi Thread CPUs - hỗ trợ 2 threads chạy trên 1 physical core) (**force mode**):
```
 openstack flavor set NUMA_8vCPU-16GB_RAM-50_Disk \
  --property hw:cpu_policy=dedicated \
  --property hw:cpu_thread_policy=require
```

Để config các vCPUs nằm trên KHÁC `thread sibling` của host CPU có SMT) (**force mode**):
```
 openstack flavor set NUMA_8vCPU-16GB_RAM-50_Disk \
  --property hw:cpu_policy=dedicated \
  --property hw:cpu_thread_policy=isolate
```

Để config các vCPUs nằm trên cùng `thread sibling` của host CPU có SMT) `nếu có thể ` (default):
```
openstack flavor set NUMA_8vCPU-16GB_RAM-50_Disk \
  --property hw:cpu_policy=dedicated \
  --property hw:cpu_thread_policy=prefer
```

**Pinning vcpu sử dụng virsh command** đối với các VMs đã tạo trước đó.
```
virsh vcpuinfo instance-0000002e

# pinning vcpu 0 to host CPU 5
virsh vcpuping instance-0000002e 0 5
```

### CPU pinning + restrict core trên compute host
Đổi config của `nova-compute` và `restart nova_compute` trên node `compute_237`

```
[DEFAULT]
vcpu_pin_set=0,1,2,3,8,9,10,11,16,17,18,19,24,25,26,27
```

Config để các host processes không chạy trên các cpu kia:

```
# grubby --update-kernel=ALL --args="isolcpus=0,1,2,3,8,9,10,11,16,17,18,19,24,25,26,27"

# grub2-install /dev/sda
Installing for i386-pc platform.
Installation finished. No error reported.

```

Reboot compute node:

```
# reboot
```

Kiểm tra lại:

```
# cat /sys/devices/system/cpu/isolated
0-3,8-11,16-19,24-27
# cat /sys/devices/system/cpu/present 
0-31
```
Thực hiện tạo flavor để VMs:

```
openstack flavor set NUMA_8vCPU-16GB_RAM-50_Disk --property hw:cpu_policy=dedicated

nova flavor-key  NUMA_8vCPU-16GB_RAM-50_Disk set aggregate_instance_extra_specs:pinned=true
```

Tạo Host Aggregate

```
# nova aggregate-create performance
+----+-------------+-------------------+-------+----------+--------------------------------------+
| Id | Name        | Availability Zone | Hosts | Metadata | UUID                                 |
+----+-------------+-------------------+-------+----------+--------------------------------------+
| 2  | performance | -                 |       |          | 6cdd933c-5943-4fd7-aa8c-544b6ee840b0 |
+----+-------------+-------------------+-------+----------+--------------------------------------+

# nova aggregate-set-metadata performance pinned=true                                                                                                  
Metadata has been successfully updated for aggregate 2.
+----+-------------+-------------------+-------+---------------+--------------------------------------+
| Id | Name        | Availability Zone | Hosts | Metadata      | UUID                                 |
+----+-------------+-------------------+-------+---------------+--------------------------------------+
| 2  | performance | -                 |       | 'pinned=true' | 6cdd933c-5943-4fd7-aa8c-544b6ee840b0 |
+----+-------------+-------------------+-------+---------------+--------------------------------------+

]# nova aggregate-add-host performance compute-1
Host compute-1 has been successfully added for aggregate 2 
+----+-------------+-------------------+-----------------+---------------+--------------------------------------+
| Id | Name        | Availability Zone | Hosts           | Metadata      | UUID                                 |
+----+-------------+-------------------+-----------------+---------------+--------------------------------------+
| 2  | performance | -                 | 'compute-1' | 'pinned=true' | 6cdd933c-5943-4fd7-aa8c-544b6ee840b0 |
+----+-------------+-------------------+-----------------+---------------+--------------------------------------+

```

Cuối cùng là tạo máy ảo và dumpxml để xem.


**Steps to enable CPU pinning in OpenStack**

-   Check if the OpenStack compute nodes (x86 server) have NUMA architecture
-   Make sure that the operating system on the compute nodes supports NUMA
-   Make sure that the hypervisor on the compute nodes support NUMA
-   Change the OpenStack Nova Scheduler configuration to support NUMA and CPU pinning



**Sources:**
- https://www.stratoscale.com/blog/openstack/cpu-pinning-and-numa-awareness/
- https://docs.openstack.org/nova/rocky/admin/cpu-topologies.html

### Test live migrate, migrate / resize VM có CPU pinned

Về `live migrate` thì OK

Về `evacuate / migrate` vm thì khi vào các compute có `vcpu_pin_set` khác với các CPU mà vm đã được pin thì sẽ bị báo lỗi `CPUPinningUnknown`. Để có thể `evacuate / migrate` các VMs có cpu pinning thì ta cần config `vcpu_pin_set` trên các compute host trong Host aggregate giống nhau.

## 3. Huge page

Kiểm tra huge page:
```
# grep Huge /proc/meminfo
AnonHugePages:   2603008 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB

# grep Huge /proc/meminfo
AnonHugePages:   3692544 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```
Trong đó:
- **HugePages_Total**: persistent huge page
- **AnonHugePages**: Transparent Huge Pages (THP) - huge pages được xác định tự động dựa trên `process requests`. Kích thước Transparent huge pages được xác định dựa trên tình trạng fragment của RAM, mặc định kích thước `2 MB huge pages` sẽ được tạo, nếu không được thì kích thước default là `4KB` sẽ được tạo. `Mục tiêu` là để tận dụng lợi ích của Huge page mà không cần phải config ở lớp ứng dụng. Kiểm tra THP đang được enabled hay disabled tại `cat /sys/kernel/mm/transparent_hugepage/enabled`. Default được enable trên centos 7.

**Nhận xét**: với kết quả của câu lệnh `grep Huge /proc/meminfo` thì các compute node đều đã đang support THP, và đang có page size là 2MB.

**Huge page** có thể được cấu hình tại thời điểm boot VM hoặc ngay trong khi VM đang chạy. Nhưng chỉ định `huge page` tại thời điểm `boot time`  thì ta có thể đảm bảo được số lượng huge page availabe, còn tại thời điểm `run time` có thể sẽ failed nếu huge page lớn và RAM đang có nhiều fragment.

> Note: For performance reasons, configuring huge pages for an instance will implicitly result in a NUMA topology being configured for the instance.

Khi sử dụng persistent huge cần lưu ý cấu hình cả ở mức ứng dụng.

**Tham khảo**

-   [https://access.redhat.com/solutions/46111](https://access.redhat.com/solutions/46111)

