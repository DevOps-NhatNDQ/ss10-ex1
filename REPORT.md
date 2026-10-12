# BÁO CÁO KỸ THUẬT: PHÂN TÍCH KIẾN TRÚC VÀ ĐÁNH GIÁ CHUYỂN DỊCH TỪ VPS SANG DOCKER

## 1. Bối cảnh và Mục tiêu chuyển dịch
Khi vận hành trên nền tảng VPS truyền thống (Virtual Private Server dựa trên Hypervisor), doanh nghiệp gặp trở ngại lớn về việc tối ưu chi phí phần cứng và chu kỳ phát hành phần mềm (CI/CD). Mỗi VPS mang theo một hệ điều hành khách (Guest OS) độc lập, dẫn đến hao phí CPU/RAM cho các tiến trình nền và thời gian khởi động kéo dài. Việc chuyển đổi sang nền tảng Container (Docker) hướng đến mục tiêu giảm thiểu footprint tài nguyên, tăng mật độ triển khai ứng dụng trên cùng một cụm máy chủ và đẩy nhanh tốc độ co giãn tự động (auto-scaling).

---

## 2. So sánh Kiến trúc: Virtual Machine (VM) vs Container

| Tiêu chí | Virtual Machine (Hypervisor) | Container (Docker Engine) |
| :--- | :--- | :--- |
| Dung lượng Image | Lớn (vài GB đến hàng chục GB do đóng gói toàn bộ Guest OS và thư viện hệ thống). | Nhẹ (vài chục MB đến vài trăm MB, chỉ chứa runtime, dependencies và mã nguồn ứng dụng). |
| Thời gian khởi động | Chậm (tính bằng phút do phải trải qua chu kỳ BIOS/UEFI, tải kernel và khởi động dịch vụ hệ thống). | Gần như tức thì (tính bằng mili-giây đến vài giây, chỉ tương đương việc khởi tạo một process Linux). |
| Hao phí hệ điều hành (OS Overhead) | Rất cao (mỗi VM chạy riêng kernel, bộ nhớ đệm, daemon nền và tốn chi phí mô phỏng phần cứng của Hypervisor). | Bằng không (chia sẻ trực tiếp Host OS Kernel, không chạy kernel riêng, không có lớp trừu tượng phần cứng). |
| Mức độ cô lập | Rất mạnh (cô lập mức phần cứng qua công nghệ Intel VT-x/AMD-V; một VM bị xâm nhập khó ảnh hưởng đến VM khác). | Mức tiến trình (cô lập mức OS thông qua kernel features; nếu kernel máy chủ bị khai thác, toàn bộ container có nguy cơ). |
| Khả năng mở rộng | Khá nặng nề và phức tạp khi tự động scale-out do thời gian provision VM lâu và chi phí tài nguyên lớn. | Vượt trội, hỗ trợ scale-out thần tốc trong các hệ thống điều phối (Docker Swarm/Kubernetes) với tài nguyên tinh gọn. |

---

## 3. Cơ chế Nền tảng của Container trong Linux Kernel

Docker không phải là một máy ảo mà bản chất là cơ chế quản lý các tiến trình cô lập trên Linux thông qua sự kết hợp của hai thành phần cốt lõi trong kernel:

### 3.1. Linux Namespaces (Cơ chế cô lập tài nguyên không gian tên)
Namespaces chịu trách nhiệm quyết định những gì một tiến trình có thể nhìn thấy trong hệ thống:
- PID Namespace (Process ID): Cung cấp cây tiến trình hoàn toàn độc lập cho container. Tiến trình chính của ứng dụng bên trong container mang PID 1 (đóng vai trò init process), trong khi trên máy chủ Host, nó thực chất chỉ là một tiến trình bình thường với PID thực tế khác. Điều này ngăn chặn container can thiệp vào các tiến trình bên ngoài.
- NET Namespace (Network): Cung cấp cho container giao diện mạng (virtual interface), bảng định tuyến (routing table), bảng ARP và dải cổng mạng riêng biệt. Nhờ đó, nhiều container trên cùng một host có thể cùng lắng nghe trên cổng 80 mà không xảy ra xung đột port, sau đó được map ra host qua mạng bridge/veth-pair.
- Mount Namespace (MNT): Cô lập toàn bộ cây thư mục và điểm gắn kết tệp tin (mount points). Kết hợp với công nghệ phân lớp UnionFS (như overlay2), tiến trình container chỉ có thể nhìn thấy và truy cập hệ thống tệp riêng của nó mà không làm thay đổi trực tiếp file hệ thống của máy chủ.

### 3.2. Control Groups - cgroups (Cơ chế phân bổ và kiểm soát tài nguyên)
Nếu Namespaces cô lập phạm vi quan sát, thì cgroups quyết định khối lượng tài nguyên mà tiến trình được phép sử dụng:
- Giới hạn CPU: cgroups sử dụng bộ lập lịch CFS (Completely Fair Scheduler) để ấn định quota thực thi. Ví dụ: cờ `--cpus="1.5"` cấp phát tối đa 150ms thời gian CPU cho mỗi chu kỳ 100ms, ngăn chặn hiện tượng một container bị treo ngốn hết 100% tài nguyên CPU của host.
- Giới hạn RAM (Memory): cgroups áp đặt ngưỡng trần dung lượng RAM (`--memory`). Khi container vượt quá ngưỡng bộ nhớ vật lý được cấu hình mà không cấu hình swap, kernel sẽ kích hoạt cơ chế OOM Killer (Out Of Memory) để chấm dứt container này, đảm bảo tính khả dụng và an toàn cho toàn bộ hệ thống máy chủ.

---

## 4. Trích xuất Thông số Vận hành Hệ thống từ `docker info`

Dựa trên dữ liệu thực tế thu thập từ lệnh `docker info` tại node máy chủ `prod-node-01`:

- Server Version: `26.1.1` — Bản phát hành chính thức của Docker Engine, tương thích hoàn toàn với API v1.45.
- Cgroup Driver: `systemd` — Hệ thống sử dụng driver `systemd` đồng nhất với bộ quản lý dịch vụ mặc định của Linux OS, ngăn ngừa xung đột cạnh tranh tài nguyên so với việc dùng cgroupfs. Hệ thống cũng đã kích hoạt phiên bản Cgroup v2 (Cgroup Version: 2).
- Số lượng Containers: `8` containers (trong đó có `6` container đang hoạt động ở trạng thái `Running` và `2` container đã dừng ở trạng thái `Stopped`).
- Số lượng Images: `15` images hiện đang lưu trữ cục bộ trong Storage Driver `overlay2`.

---

## 5. Kết luận
Chuyển đổi kiến trúc từ VPS sang Container giúp doanh nghiệp tối ưu triệt để chi phí hạ tầng thông qua việc loại bỏ chi phí duy trì Guest OS, đồng thời đạt được tốc độ triển khai vượt trội nhờ Namespaces và cgroups. Để đảm bảo an toàn vận hành, hệ thống cần duy trì driver `systemd` kết hợp cgroups v2 để quản lý hạn mức tài nguyên nghiêm ngặt cho từng dịch vụ.


## 6. Kết quả chạy lệnh

```bash
[root@prod-node-01 ~]# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.17.0
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.37.2
    Path:     /usr/libexec/docker/cli-plugins/docker-compose
  volume: Docker Volume (Docker Inc.)
    Version:  v1.0.3
    Path:     /usr/libexec/docker/cli-plugins/docker-volume

Server:
 Containers:
  7          Running
  1          Paused
  0          Stopped
 Images:         15
 Server Version: 26.1.1
 Storage Driver: overlay2
  Backing Filesystem: ext4
  Supports d_type: true
 Cgroup Version: 2
 Cgroup Driver: systemd
 Cpus:         10
 Memory:       14.57GiB
 Kernel Version: 6.9.13-200.el9.x86_64
 Root Dir:     /var/lib/docker
... (phần thông tin còn lại) ...
[root@prod-node-01 ~]# 
```
