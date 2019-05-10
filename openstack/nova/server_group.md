### Giới thiệu về Server group

Khi tạo server group có `4 policy` là:
- **Affinity** [[1]](#abc): các vm trong cùng 1 affinity server group sẽ nằm trên cùng 1 compute host. Không có restrict giữa các affinity server group với nhau, nghĩa là các vm trong 2 affinity server group vẫn có thể nằm trên cùng 1 compute host (E.g: các compute node khác hết tài nguyên, số affinity server group >= compute node, ...)
- **Soft affinity** [[2]](#abc): được áp dụng trong quá trình nova-scheduler `weight`, nhằm mục đích khi compute host hiện tại hết tài nguyên thì các vm tiếp theo trong soft affinity group sẽ được phân bổ nằm trên cùng 1 node compute khác. Nhưng do sử dụng weight nên sẽ bị ảnh hưởng bởi các tham số weigth khác. 
- **Anti-affinity** [[3]](#abc): các vm trong cùng 1 anti-affinity server group sẽ nằm trên các compute host khác nhau. Trong trường hợp không còn compute host nào khác cho vm mới tạo ra thì sẽ báo lỗi ` Exhausted all hosts available for retrying build failures for instance UUID`.
    - Hiện tại trong anti-affinity group đã support thêm rule: `max_server_per_host` trong anti-affinity group.  Sử dụng `novaclient` để tạo và show info (do openstackclient compute chưa support [[4]](#abc))
	    ```
	    nova server-group-create anti-affinity-max-server-2 anti-affinity --rule max_server_per_host=2
	    nova server-group-get UUID
		```
- **Soft anti-affinity** [[5]](#abc): cũng được áp dụng trong quá trình `weight`, tương tự như soft affinity.

Ngoài ra còn có default config `server-group-members` (quy định số VMs trong 1 server group) là `10`

<a name="abc"></a>
- [1] https://github.com/openstack/nova/blob/a86604dc60/nova/scheduler/filters/affinity_filter.py#L137 
- [2] https://github.com/openstack/nova/blob/a86604dc60/nova/scheduler/weights/affinity.py#L55
- [3] https://github.com/openstack/nova/blob/a86604dc60/nova/scheduler/filters/affinity_filter.py#L82
- [4] https://github.com/openstack/python-openstackclient/blob/master/openstackclient/compute/v2/server_group.py#L45
- [5] https://github.com/openstack/nova/blob/a86604dc60/nova/scheduler/weights/affinity.py#L74

**Tham khảo:**
- https://specs.openstack.org/openstack/nova-specs/specs/liberty/approved/soft-affinity-for-server-group.html
- https://specs.openstack.org/openstack/nova-specs/specs/rocky/implemented/complex-anti-affinity-policies.html

