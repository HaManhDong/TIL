

### NUMA
NUMA: None-Uniform Memory Access

**Vấn đề**: CPU hoạt động nhanh hơn RAM rất nhiều => Khi xử lý lượng dữ liệu lớn trên nhiều CPUs thì các CPUs sẽ phải chờ để lấy dữ liệu từ RAM vào, thời gian truy cập RAM phụ thuộc vào vị trí mà CPU cần truy cập tới.

**NUMA architecture** được thiết kế để sử dụng trong `multilprocessing`: với tư tưởng là cung cấp RAM riêng biệt cho từng CPU hoặc 1 nhóm các CPUs, nhằm: `avoiding the performance hit when several processors attempt to address the same memory.` Mặc định các CPUs nằm trên cùng 1 socket sẽ được chia vào cùng 1 NUMA nodes (kiểm tra bằng lệnh `lscpu | grep NUMA`), và sử dụng chung 1 phần RAM (gọi là local RAM).  Cụ thể là khi ta cắm RAM thì cần cắm đều vào 2 khay RAM ở 2 bên:

![numa](http://www.ksingh.co.in/images/NUMA.png)

**Vấn đề của NUMA**: trong trường hợp dữ liệu cần được xử lý trên nhiều thread, thì khi đó sẽ có các CPU trên các NUMA node khác nhau cần truy cập vào cùng 1 dữ liệu đó. Để xử lý vấn đề này, giữa các NUMA nodes sẽ có thêm `hardware hoặc software` để vẫn chuyển dữ liệu giữa các local RAM của các NUMA node. Hoạt động share dữ liệu này diễn ra chậm => ảnh hưởng tới performance. => Chỉ phù hợp với 1 số ứng dụng.

> For a NUMA system, the best performance is achieved when a process and its memory reside in the same NUMA node.

> **Note**: if the compute nodes have NUMA architecture, then it is a good idea to enable NUMA and leverage the operating system and hypervisor to use CPU pinning. 

Ngoài ra còn 1 số khái niệm:

**SMP - Symmetric Multi-Processing**: tương ứng với số lượng `physical cores` trên host.

**SMT CPUs- Simultaneous Multi-Thread CPUs**: tương ứng với số lượng `threads` mà 1 physical core support (thường là 2).

**Hiện tại:** có [bug nova 1289064](https://bugs.launchpad.net/nova/+bug/1289064) và [specs](https://specs.openstack.org/openstack/nova-specs/specs/rocky/approved/numa-aware-live-migration.html) liên quan tới việc `live-migrate` khi sử dụng NUMA topology , đã được fix [ở đây](https://review.opendev.org/#/c/625880/) và [ở đây](https://review.opendev.org/#/c/640462/) => Nếu sử dụng NUMA và CPU pinning thì cần cherry pick về.

### CPU pinning

CPU pinning: processor affinity

Enables the binding and unbinding of a [process](https://en.wikipedia.org/wiki/Process_(computing) "Process (computing)") or a [thread](https://en.wikipedia.org/wiki/Thread_(computing) "Thread (computing)") to a [central processing unit](https://en.wikipedia.org/wiki/Central_processing_unit "Central processing unit") (CPU) or a range of CPUs, so that the process or thread will execute only on the designated CPU or CPUs rather than any CPU 

**Steps to enable CPU pinning in OpenStack**

-   Check if the OpenStack compute nodes (x86 server) have NUMA architecture
-   Make sure that the operating system on the compute nodes supports NUMA
-   Make sure that the hypervisor on the compute nodes support NUMA
-   Change the OpenStack Nova Scheduler configuration to support NUMA and CPU pinning

**Sources:**
- https://www.stratoscale.com/blog/openstack/cpu-pinning-and-numa-awareness/
- https://docs.openstack.org/nova/rocky/admin/cpu-topologies.html



