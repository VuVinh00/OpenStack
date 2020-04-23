#### Core plugin
**Core API** chịu trách nhiệm cung cấp kết nối Layer-2 cho VM. API server gọi tới core plugin khi nó nhận được request create a new virtual network hoặc khi một port mới được tạo trên virtual network

**Core plugin** implement một số Neutron resources:
- **Network**: Neutron network resource đại diện cho một domain Layer-2. Tất cả virtual instances cần network đều phải kết nối tới virtual network
- **Ports**: A Neutron port đại diện cho một thiết bị đầu cuối tham gia vào virtual network
- **Subnet**: Neutron subnet resouces add thêm Layer-3 pool tới Neutron network. The subnet có thể có một DHCP server cung cấp IP cho VM

**Modular Layer 2 (ML2)** neutron plug-in là framework cho phép OpenStack Networking đồng thời sử dụng nhiều công nghệ mạng. ML2 framework phân biệt giữa 2 loại drivers có thể được cấu hình:

- **Type drivers**:
  - Định nghĩa cách mà OpenStack network thực hiện kỹ thuật.
    - Flat: Tất cả instances kết nối vào một network chung. Không được tag VLAN_ID vào packet hoặc phân chia vùng mạng
    - VLAN(12bit = 4096 id) : mạng LAN ảo
    - VXLAN/GRE(24bit = 16 triệu id) : tạo ra các mạng ảo L2 trên nền hạ tầng mạng L3 network
  - Mỗi loại mạng được quản lý bởi ML2 type driver. Type drivers duy trì bất kỳ trạng thái mạng cụ thể nào. Chúng xác nhận thông tin cụ thể cho provider network và có trách nhiện phân bổ segment miễn phí trong project networks.
- **Mechanism drivers**
  - Định nghĩa cơ chế (machanism) truy cập OpenStack network của loại nào đó. VD: Open vSwitch mechanism driver.
  - Mechanism driver chịu trách nhiệm về việc lấy thông tin thiết lập bởi type driver và đảm bảo nó được áp dụng đúng cách với mechanisms mạng cụ thể đã được kích hoạt.
  - Mechanism driver có thể dụng L2 agents (thông qua RPC) và/hoặc trực tiếp tương tác với thiết bị bên ngoài hoặc controllers.
  - Nhiều mechanism và type drivers có thể được sử dụng đồng thời để truy cập cập các ports khác nhau của cùng 1 virtual network.

#### Service plugin 
Được gọi để thực hiện các dịch vụ network cao cấp hơn, ví dụ như routing giữa networks, implementation of firewalls, loab balancers hoặc tạo VPN services
