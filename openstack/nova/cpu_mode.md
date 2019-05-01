### KVM configuration

#### Specify the CPU model of KVM guests

Mục tiêu;
- Maximize performance cho VMs thông qua việc expose các CPU features 
- Đảm bảo `consistent CPU` giữa toàn bộ các VMs => Live migrate.

Trong libvirt, thông tin về CPU bao gồm:
- Base CPU model name (which is a shorthand for a set of feature flags) (check in `/usr/share/libvirt/cpu_map.xml`)
- A set of additional feature flags
- The topology (sockets/cores/threads)

**Có 2 config cần quan tâm:**  `cpu_mode` và `cpu_model`: define which type of CPU model is exposed to the hypervisor when using KVM.
- **cpu_mode**: `none`, `host-passthrough`, `host-model`, and `custom`.
	- **host-model (default for KVM & QEMU)**: khi trong nova.conf để `cpu_mode=host-model` thì libvirt sẽ xác CPU model của host bằng cách tìm kiếm matching các features mà host CPU đang enable so với trong file `/usr/share/libvirt/cpu_map.xml` => Config này giúp VM đạt được performance tốt (nhưng chưa phải nhất) đồng thời giúp có thể live migrate VMs được trong trường hợp `slightly different CPU model`. (Để có thể live migrate được giữa tất cả các compute host khác nhau CPU model thì cần sử dụng `cpu_mod=custom` - xem tiếp bên dưới)
	- **host-passthrough**: khi sử dụng config này, libvirt sẽ báo với KVM sử dụng luôn host CPU mà không chỉnh sửa gì về các features enabled (so với `host-model`). Do đó mà ở mode này thì VMs sẽ đạt được performance tốt nhất, nhưng sẽ không live migrate được trong trường hợp compute host khác CPU model, thậm chí nếu khác cả về kernel versions [[1]](https://docs.openstack.org/nova/latest/admin/configuration/hypervisor-kvm.html#host-pass-through ). => Chỉ sử dụng mode này khi các compute node giống hệt nhau về CPU model cũng như kernel versions, hoặc trong trường hợp xác định là không cần live migrate VMs trên các node compute này.
	- **custom**: khi sử dụng mode này, ta sẽ cần chỉ định CPU model name cho libvirt thông qua config `cpu_model`. Ta có thể tham khảo trong file `/usr/share/libvirt/cpu_map.xml` để tìm ra được CPU model phù hợp nhất (các features enabled) với các loại host CPU model khác nhau.  
	Ví dụ về config:
		```
		[libvirt]
		cpu_mode = custom
		cpu_model = pentium3
		```
	- **none (default for all libvirt-driven hypervisors other than KVM & QEMU)**: ở mode này, libvirt sẽ không chỉ định CPU model nữa mà để cho hypervisor tự chọn default model.

**Set CPU extra feature flags**
Ngoài ra, khi **cpu_mode** là `host-passthrough`, `host-model`, hoặc `custom` thì ta có thể config thêm `cpu_model_extra_flags` để chỉ định thêm các feature flags. Ví dụ khi ta chọn `cpu_model=IvyBridge` thường sẽ không enable flag `pcid`, do đó nếu cần, ta có thể config:
```
[libvirt]
cpu_mode = custom
cpu_model = IvyBridge
cpu_model_extra_flags = pcid
```

[1] https://docs.openstack.org/nova/latest/admin/configuration/hypervisor-kvm.html#host-pass-through 
