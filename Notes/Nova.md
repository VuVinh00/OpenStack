## Filtering

- AllHostsFilter: không filtering. Nó duyệt tất cả các hosts có sẵn.
- ImagePropertiesFilter: filter host dựa vào properties được định nghĩa trên instance’s image. Nó sẽ chọn các host có thể hỗ trợ các thông số cụ thể trên image được sử dụng bởi instance. ImagePropertiesFilter dựa vào kiến trúc, hypervisor type và virtual machine mode được định nghĩa trong instance. Ví dụ, máy ảo yêu cầu host hỗ trợ kiến trúc ARM thì ImagePropertiesFilter sẽ chỉ chọn những host đáp ứng yêu cầu này.
- CoreFilter: filter dựa vào mức độ sử dụng CPU core. Nó sẽ chọn host có đủ số lượng CPU core.
- AggregateCoreFilter: filter bằng số lượng CPU core với giá trị cpu_allocation_ratio .
- RamFilter: filter bằng RAM, các hosts có đủ dung lượng RAM sẽ được chọn.
- AggregateRamFilter: filter bằng số lượng RAM với giá trị ram_allocation_ratio

## Weighting

- Scheduler_weight_classes: Host weighted và sắp xếp lớn nhất được chỉ định làm hosts đặt instances (mặc định dùng tất cả các weigher)
- io_ops_weight_multiplier: Multiplier used for weighing host I/O operations. A negative value means a preference to choose light workload compute hosts.

## Nova-cell

Thông thường khi cài đặt OpenStack, tất cả compute node cần gửi thông tin vào message queue và DB server (sử dụng nova-conductor). Cách làm này khá nặng nề cho messages queue và DB. Khi cloud của bạn mở rộng, có rất nhiều compute server kết nối cùng hạ tầng tài nguyên và gây nghẽn.

Khái niệm Cell ra đời để giúp mở rộng tài nguyên tính toán. Nova Cell là cách để mở rộng compute workload bằng cách phần phối tải lên hạ tầng tài nguyên như DB, message queue, tới nhiều instance.

Kiến trúc Nova cell tạo ra các group compute nodes, sắp xếp theo mô hình cây, được gọi là cell. Mỗi cell có DB và message queue của nó. Chiến thuật này là để ràng buộc kết nối đến DB, message queue theo các cell khác nhau.

Bằng cách nào kiến trúc này hoạt động? hãy nhìn vào các thành phần và cách nó giao tiếp với nhau. Như đã trình bày bên trên, các cell được sắp xếp theo kiểu cây, gốc của cây là API cell và nó chạy Nova API nhưng không chạy Nova compute service, trong khi các node khác được gọi là compute cell, chạy tất cả Nova service.

Cell’s architecture làm việc bằng cách tách Nova API service (thành phần nhận đầu vào từ toàn bộ cách thành phần khác của Nova compute). Tương tác giữa Nova API và các thành phần khác được thay thế bởi Message queue based RPC call, khi Nova API nhận một call để khởi động instance mới, nó sử dụng Cell RPC call để sắp xếp instance này trên 1 compute cell đang sẵn sàng. Compute cell chạy DB, message queue của nó và hoàn thành tập nova service ngoại trừ Nova API. Compute cell sau đó chạy instance bằng cách sắp xếp nó trên một compute node:
