Modular Layer 2 (ML2) neutron plug-in là framework cho phép OpenStack Networking đồng thời sử dụng nhiều công nghệ mạng. ML2 framework phân biệt giữa 2 loại drivers có thể được cấu hình:

- Type drivers:
  - Định nghĩa cách mà OpenStack network thực hiện kỹ thuật. VD: VXLAN
  - Mỗi loại mạng được quản lý bởi ML2 type driver. Type drivers duy trì bất kỳ trạng thái mạng cụ thể nào. Chúng xác nhận thông tin cụ thể cho provider network và có trách nhiện phân bổ segment miễn phí trong project networks.
- Mechanism drivers
  - Định nghĩa mechanism truy cập OpenStack network của loại nào đó. VD: Open vSwitch mechanism driver.
  - Mechanism driver chịu trách nhiệm về việc lấy thông tin thiết lập bởi type driver và đảm bảo nó được áp dụng đúng cách với mechanisms mạng cụ thể đã được kích hoạt.
  - Mechanism driver có thể dụng L2 agents (thông qua RPC) và/hoặc trực tiếp tương tác với thiết bị bên ngoài hoặc controllers.
  - Nhiều mechanism và type drivers có thể được sử dụng đồng thời để truy cập cập các ports khác nhau của cùng 1 virtual network.
