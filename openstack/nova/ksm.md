
### KSM - Giới thiệu

**KSM - Kernel Samepage merging**: là một tính năng của kernel cho phép hypervisor có thể chia sẻ các pages trong bộ nhớ giống hệt nhau cho các tiến trình khác nhau hoặc các VM khác nhau.

KVM có thể sử dụng KSM để hợp nhất các pages giống hệt nhau trong memory.  

**Quá trình**
KSM thực hiện việc scan trong bộ nhớ chính -> Tìm kiếm các pages giống nhau. -> gộp các pages giống hệt nhau thành một page. -> Page này được đánh dấu là "Copy On Write" có nghĩa là nếu có một tiến trình muốn modify page này thì nó sẽ tạo ra một page mới để thực hiện các thay đổi trên page đó.

**Tác dụng**: giúp tiết kiệm và tối ưu RAM

**Tradeoff**: 
- CPU phải hoạt động nhiều hơn: để KSM daemon tìm các page giống nhau, gộp các page lại.
- Hiệu năng của VM sẽ bị giảm đi: ngoài vấn đề về CPU, còn 1 vấn đề khác là khi VM cần modify các `shared page` thì phải chờ để tạo ra 1 page mới rồi mới bắt đầu thực hiện các modify.

**Note**: KSM có support đối với các server hiện nay sử dụng `NUMA topology`. Cấu hình tại 	`/sys/kernel/mm/ksm/merge_across_nodes` với giá trị:
- `0`:  chỉ merge các page giống nhau trên cùng NUMA node
- `1`: gộp cả các page giống nhau trên khác NUMA node -> **`Không nên dùng mode này vì ảnh hưởng lớn tới hiệu năng.`**

**Các cấu hình:**

`Source:` https://www.kernel.org/doc/Documentation/vm/ksm.txt

```
# grep . /sys/kernel/mm/ksm/*                                                                                                                             
/sys/kernel/mm/ksm/full_scans:0
/sys/kernel/mm/ksm/max_page_sharing:256
/sys/kernel/mm/ksm/merge_across_nodes:1
/sys/kernel/mm/ksm/pages_shared:0
/sys/kernel/mm/ksm/pages_sharing:0
/sys/kernel/mm/ksm/pages_to_scan:100
/sys/kernel/mm/ksm/pages_unshared:0
/sys/kernel/mm/ksm/pages_volatile:0
/sys/kernel/mm/ksm/run:0
/sys/kernel/mm/ksm/sleep_millisecs:20
/sys/kernel/mm/ksm/stable_node_chains:0
/sys/kernel/mm/ksm/stable_node_chains_prune_millisecs:2000
/sys/kernel/mm/ksm/stable_node_dups:0
```

Trong đó:  (những tham số cần lưu ý có **`màu như này`**)
- **`full_scans`**: 0 -> Thời gian (s)
- **max_page_sharing**: 256 -> Maximum sharing allowed for each KSM page (>= 2)
- **merge_across_nodes:** 1 ->
	 - 0: chỉ merge các page giống nhau trên cùng NUMA node
	 - 1: gộp cả các page giống nhau trên khác NUMA node
- **`pages_shared`:** 0 -> Số pages đã shared (đã merged) 
- **`pages_sharing`:** 0 -> Số pages đang sharing
- **pages_to_scan:** 100 -> Số pages sẽ scan
- **`pages_unshared`:** 0 -> Số pages khác nhau nhưng sẽ liên tục kiểm tra các pages này để merge 
- **`pages_volatile`:** 0  -> Số lượng page thay đổi quá nhanh (how many pages changing too fast to be placed in a tree)
- **run:** 0 ->
	 - 0: stop ksmd daemon nhưng vẫn giữ lại merged pages
	 - 1: run ksmd
	 - 2: stop ksmd daemon đồng thời unmerged các page, but leave mergeable areas registered for next run 
- **sleep_millisecs:** 20 -> thời gian sleep cho tới lần scan tiếp theo
- **stable_node_chains:** 0 -> number of stable node chains allocated
- **stable_node_chains_prune_millisecs:** 2000
- **stable_node_dups:** 0 -> number of stable node dups queued into the stable_node chains

**Khuyến nghị**
- Tỉ lệ `pages_sharing / pages_shared` càng cao càng tốt
- Tỉ lệ `pages_unshared / pages_shared` càng thấp càng tốt, nếu tỉ lệ này rất cao thì nên xem xét xem có nên sử dụng KSM hay không
(Tỉ lệ bao nhiêu? - chưa tìm được)


### KSM - Sử dụng

Tạo `5 VM` với flavor `2 vCPUs - 4GB RAM - 40GB Disk` trên `compute237`:

![Imgur](https://i.imgur.com/ZdHLlWJ.png)

**Thông tin RAM và CPU sử dụng trước khi enabe KSM**
```
# free -g
              total        used        free      shared  buff/cache   available
Mem:             62          15          19           1          27          39
Swap:            27           0          27
```

**Thực hiện**
```
# date
Thu Jun 20 14:54:00 +07 2019

# echo 0 > /sys/kernel/mm/ksm/merge_across_nodes

# echo 1 > /sys/kernel/mm/ksm/run

# top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                               
  189 root      25   5       0      0      0 S   3.0  0.0   0:03.25 ksmd 

# echo "100000" > /sys/kernel/mm/ksm/pages_to_scan
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                      
  189 root      25   5       0      0      0 R  90.4  0.0   0:30.10 ksmd

# grep . /sys/kernel/mm/ksm/*
/sys/kernel/mm/ksm/full_scans:8
/sys/kernel/mm/ksm/max_page_sharing:256
/sys/kernel/mm/ksm/merge_across_nodes:0
/sys/kernel/mm/ksm/pages_shared:88213
/sys/kernel/mm/ksm/pages_sharing:548163
/sys/kernel/mm/ksm/pages_to_scan:100000
/sys/kernel/mm/ksm/pages_unshared:1139200
/sys/kernel/mm/ksm/pages_volatile:2526
/sys/kernel/mm/ksm/run:1
/sys/kernel/mm/ksm/sleep_millisecs:20
/sys/kernel/mm/ksm/stable_node_chains:9
/sys/kernel/mm/ksm/stable_node_chains_prune_millisecs:2000
/sys/kernel/mm/ksm/stable_node_dups:1033

# free -g
              total        used        free      shared  buff/cache   available
Mem:             62          13          21           1          27          41
Swap:            27           0          27
```

![Imgur](https://i.imgur.com/gxvbu3d.png)

![Imgur](https://i.imgur.com/djquGPS.png)

**Nhận xét**: khi tăng số lượng `pages_to_scan=100000`, chờ KSM chạy 1 lúc (8s) và check lại thì thấy RAM đã giảm được 2GB, xem trên `top` thì `ksmd` sử dụng tới `90.4% CPU 0`, xem trên grafana theo biểu đồ `cpu busy - theo metrics node_cpu_seconds_total` thì tăng từ `1% lên 4%`. 

**Thử giảm pages_to_scan từ 100.000 -> 50.000 để xem CPU có giảm hay không**

=> %CPU có giảm xuống nhưng không đáng kể: từ `90.4% -> 83.8%`

```
[root@controller237 ~]# echo 20000 > /sys/kernel/mm/ksm/pages_to_scan

[root@controller237 ~]# top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                      
  189 root      25   5       0      0      0 R  83.8  0.0  26:57.09 ksmd

[root@controller237 ~]# grep . /sys/kernel/mm/ksm/*
/sys/kernel/mm/ksm/full_scans:143
/sys/kernel/mm/ksm/max_page_sharing:256
/sys/kernel/mm/ksm/merge_across_nodes:0
/sys/kernel/mm/ksm/pages_shared:90859
/sys/kernel/mm/ksm/pages_sharing:548071
/sys/kernel/mm/ksm/pages_to_scan:20000
/sys/kernel/mm/ksm/pages_unshared:1136285
/sys/kernel/mm/ksm/pages_volatile:2890
/sys/kernel/mm/ksm/run:1
/sys/kernel/mm/ksm/sleep_millisecs:20
/sys/kernel/mm/ksm/stable_node_chains:9
/sys/kernel/mm/ksm/stable_node_chains_prune_millisecs:2000
/sys/kernel/mm/ksm/stable_node_dups:1032
```

### Stop KSM nhưng vẫn giữ các page đã merged
```
[root@controller237 ~]# grep . /sys/kernel/mm/ksm/*
/sys/kernel/mm/ksm/full_scans:182
/sys/kernel/mm/ksm/max_page_sharing:256
/sys/kernel/mm/ksm/merge_across_nodes:0
/sys/kernel/mm/ksm/pages_shared:90876
/sys/kernel/mm/ksm/pages_sharing:548060
/sys/kernel/mm/ksm/pages_to_scan:20000
/sys/kernel/mm/ksm/pages_unshared:1137454
/sys/kernel/mm/ksm/pages_volatile:1716
/sys/kernel/mm/ksm/run:1
/sys/kernel/mm/ksm/sleep_millisecs:20
/sys/kernel/mm/ksm/stable_node_chains:9
/sys/kernel/mm/ksm/stable_node_chains_prune_millisecs:2000
/sys/kernel/mm/ksm/stable_node_dups:1032

[root@controller237 ~]# echo 0 > /sys/kernel/mm/ksm/run

[root@controller237 ~]# free -g
              total        used        free      shared  buff/cache   available
Mem:             62          13          21           1          27          41
Swap:            27           0          27

```
