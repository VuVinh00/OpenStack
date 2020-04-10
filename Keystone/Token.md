### 1. Token là gì?

OpenStack token là bearer tokens, được sử dụng để authenticate và validate users và processes trong môi trường Openstack. Dịch vụ Openstack Keystone là một core service chịu trách nhiệm về các issues và validates tokens. Khi sử dụng tokens, users và các software clients thông qua API's authenticate có thể nhận và cuối cùng sử dụng token khi gửi các request. Ví dụ khi tạo compute resources để cấp phát storage, các service như Nova hoặc Ceph sẽ phải validate token với Keystone, nếu được Keystone chấp nhận thì sẽ thực hiện requested, còn không sẽ bị deny


<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/token-work.png">

### 2. Các loại token

Các loại token trong Keystone

#### UUID (Universal Unique Identifier)

Mặc định token provider trong Keystone là UUID. Nó là 32-byte bearer token cần phải lưu trữ trên các node controller cùng với metadata liên quan để được validated

Đặc điểm của UUID trong keystone

- Có độ dài 32 byte, nhỏ, dễ sử dụng, không nén
- Không mang theo đầy đủ thông tin, do đó luôn phải gửi lại keystone để xác thực authorize
- Được lưu vào database
- Sử dụng thời gian dài làm giảm hiệu suất hoạt động, CPU tăng và thời gian đáp ứng lâu
- 

#### PKI & PKIZ (public key infrastructure)

Token format cũng được lưu trữ trên các node controller. PKI tokens chứa thông tin catalog của người dùng, vì vậy kích thước của token có thể khá lớn, tùy thuộc vào quy mô của cloud. PKIZ token chi đơn giản là phiên bản nén của PKI tokens

Đặc điểm của PKI

- Mã hóa bằng private key, kết hợp public key để giải mã, lấy thông tin
- Token chứa nhiều thông tin như User ID, project ID, domain, role, service catalog, create time, exp time,...
- Xác thực ngay tại user, không cần phải gửi yêu cầu xác thực đến Keystone
- Có bộ nhớ cache, sử dụng cho đến khi hết hạn hoặc bị thu hồi --> ít truy vấn đến keystone
- Kích thước lớn, chuyển token qua HTTP
- Kích thước lớn chủ yếu do chứa thôn tin service catalog
- Tuy nhiên header của HTTP chỉ giới hạn 8kb. Web server không thể xử lý nếu không cấu hình lại, khó khăn hơn UUID
- Để khắc phục lỗi trên thì phải tăng kích thước header HTTP của web server
- Lưu token vào database

Đặc điểm của PKIZ

- Tương tự PKI
- Khắc phục nhược điểm của PKI, token sẽ được nén lại để có thể truyền qua HTTP
- TUy nhiên token vẫn có kích thước lớn

#### Fernet

Fernet token là một message packed tokens chứa data về authentication và authorization. Fernet tokens được sinh ra và mã hóa trước khi gửi cho users. Điều quan trọng là fernet không cần lưu trữ trên cluster để validate thành công

Đặc điểm của fernet:

- Sử dụng mã hóa đối xứng (Sử dụng chung key để mã hóa và giải mã)
- Có kích thước khoảng 255 byte, không nén (lớn hơn UUID và nhỏ hơn PKI)
- Chứa các thông tin cần thiết như userid, projectid, domainid, methods,... Không chứa service catalog
- Không lưu token vào database
- Cần phải gửi lại keystone để xác nhận, tương tự UUID
- Cần phải phân phối các khóa cho các khu vực khác nhau trong Openstack <-- chưa rõ
- Sử dụng cơ chế xoay khóa để tăng tính bảo mật
- Nhanh hơn UUID và PKI

###### Các loại key

- **Primary key**: Sử dụng cho mã hóa và giải mã token fernet (Chỉ số khóa cao nhất)
- **Secondary key**: Sử dụng cho giải mã token (Chỉ số khóa nằm giữa primary key và staged key)
- **Staged key**: Sử dụng cho giải mã token tương tự secondary key. Khác ở chỗ là Staged key sẽ trở thành primary key ở lần xoay khóa tiếp theo (Chỉ số khóa thấp nhất)

###### Token format

``Version | Timestamp | IV | Ciphertext | HMAC``

- **Version**: 8 bits, chỉ phiên bản token được sử dụng
- **Timestamp**: 64 bits kiểu nguyên, là khoảng thời gian từ ngày 1/1/1970 đến ngày mà token được sinh ra
- **IV (Initialization Vector)**: 128 bits, là một số ngẫu nhiên được sinh ra, mỗi token sẽ có một giá trị IV
- **Ciphertext**: Có kích thước khác nhau, chứa các thông điệp nhận vào
- **HMAC**: Có độ dài 256 bits chứa các trường:

``Version | Timestamp | IV | Ciphertext``

Fernet sử dụng Base64 URL safe để encoded các thành phần trên

###### Fernet Token Generation Workflow

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/generationtoken.png">

- 1. Kết hợp các thông tin: Version, User ID, Methods, Project ID, Expiry Time, Audit ID với Padding
- 2. Các thông tin trên được mã hóa bằng Encryption key (Cipher Text)
- 3. Sử dụng Siging key để kết hợp các trường Fernet Token version, Current Timestamp, IV, Cipher Text
- 4. Thông tin vừa được signing là HMAC

###### Fernet Token Validation Workflow

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/validationtoken.png">

- 1. Restore Padding
- 2. Giải mã token bằng cách sử dụng Fernet Keys để nhận payload
- 3. Xác định version token payload
- 4. Xác định các trường thông tin: User ID, Project ID, Methods, Token Expiry, Audit ID
- 5. So sánh thời gian hiện tại với thời gian hết hạn của token --> Nếu hết hạn: Token Not Found
- 6. Kiểm tra token có bị thu hồi hay không --> Nếu bị thu hồi: Token Not Found
- 7. Trả về token

#### 2.1 Lợi ích của Fernet tokens

- Bởi vì Fernet tokens là ephemeral, ta có ngay các lợi ích sau:
    - Tokens không cần replicated tới các Keystone khác trên controller cluster
    - Storage không bị ảnh hưởng bởi token không được lưu trữ
- Lợi ích quan trọng của Fernet tokens là security. Khi fernet được tạo và mã hóa sẽ secure hơn UUID token là plain text. Ta cũng có thể vô hiệu hóa số lượng lớn token bằng cách đơn giản là thay đổi key được sử dụng để validate chúng (this requires a key rotation strategy)
- Keystone tạo token với các key sử dụng advanced tachnologies như SElinux

### 3. Scoped token

Scoped hay còn gọi là phạm vi ủy quyền:

- Token sẽ thể hiện quyền khác nhau với các scope khác nhau. Ví dụ như quyền trên các project, domain, chứng thực trong một thời điểm khác nhau
- Bản chất mỗi token được cấp sẽ có quyền khác nhau khi làm việc với các project khác nhau

Unscoped token

- Token không có quyền sử dụng bất kỳ catalog, role, project hoặc domain. Mục đích chính sử dụng của unscoped token chỉ đơn giản là xác minh danh tính với Keystone tại thời điểm sau (thường được dùng để generate scoped token) mà không cần xác minh lại thông tin ban đầu

Project-scoped token:

- Project chứa resources như volumes hoặc instances. Project-scoped token thể hiện sự ủy quyền để hoạt động trong project
- Nó bao gồm service catalog, tập hợp các roles, thông tin về project

Domain-scoped token:

- Token cho quyền thực hiện hành động trên domain chỉ định
- Mỗi domain thường bao gồm nhiều project và user

### 4. The overall workflow

Đầu tiên ta lấy một ví dụ, sử dụng openstack CLI để list các project đang tồn tại. Để nhìn thấy rõ hơn những gì câu lệnh này thực hiện, ta chạy câu lệnh với option ``--debug``:

``openstack --debug project list``

Nó sẽ cho một kết quả khá dài và khó đọc, đầu tiên ta nhìn vào dòng tín hiệu rằng một yêu cầu đến API được thực hiện. API đầu tiên là một GET request đến URL ``http://controller:5000/v3``

Yêu cầu này sẽ trả lại một list của version API sẵn có. Ở case này, kết quả trả ra stable version là v3. Tiếp theo client sẽ gửi POST request tới URL ``http://controller:5000/v3/auth/tokens``

Ta sẽ nhìn thấy API endpoint trong [Keystone Identity API reference](https://docs.openstack.org/api-ref/identity/v3/?expanded=token-authentication-with-scoped-authorization-detail#token-authentication-with-scoped-authorization), phương pháp này được sử dụng để create và return token. Khi đưa ra request này, client sẽ sử dụng các dữ liệu được cung cấp trong biến môi trường được set trong file ``rc`` để authenticate với Keystone, Keystone sẽ assemble và return lại token. Ta có thể format token sang json để dễ đọc hơn bằng các copy token vào file có định dạng .json sau đó dùng câu lệnh ``jq . file.json``

Cuối cùng, ở dưới output, ta sẽ thấy lời gọi API để lấy list project được tạo ra ``http://controller:5000/v3/projects``

--> Overall flow sẽ như thế này:

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/authorizationworkflowgetprojects.png">

Tương tự như các service khác

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/openstackauthorization.png">

### 5. Lấy token, thao tác với OpenStack API

Để thao tác với OpenStack API có một số phương thức sau:

- **Curl**:
    - Công cụ dòng lệnh cho phép tương tác với các giao thức FPT, HTTP, IMAP,...
    - Không có giao diện, chạy trên dòng lệnh
- **OpenStack command-line client**:
    - Mỗi project trong OpenStack cho phép người dùng tương tác với chính nó thông qua CLI. Bản chất các CLI sử dụng API OPS
- **REST client**:
    - Giao diện đồ họa, cho phép người sử dụng làm việc nhanh chóng với API
- **OpenStack Python Software Development Kit (SDK)**:
    - Viết trên Pyhon cho phép user là việc với tài nguyên Openstack
    - SDK sử dụng python để làm việc với các API Openstack
    - OpenStack CLI được xây dựng trên Python SDK

##### Chuẩn bị môi trường

Bài viết sẽ sử dụng Postman để làm việc với OpenStack API:

Postman là một công cụ cho phép chúng ta làm việc với API, nhất là REST. POSTMAN hỗ trợ tất cả các phương thức HTTP (GET, POST, PUT,...). Postman cho phép lưu lại lịch sử các lần request, rất tiện cho việc sử dụng lại khi cần

Để bắt đầu với postman ta vào trang chủ https://www.getpostman.com/ và download phiên bản phù hợp cho hệ điều hành đang sử dụng

##### Lấy token dạng unscoped token

Bước 1: Sử dụng giao thức POST, URL gồm địa chỉ của controller, port Keytone, ví dụ: ``http://10.5.9.4:5000/v3/auth/tokens``

Bước 2: Chèn data raw dạng json vào body để xác thực

```
{ "auth": {
	"identity": {
		"methods": ["password"],
		"password": {
			"user": {
				"name": "admin",
				"domain": { "id": "default" },
				"password": "123@123"
			}
		}
	}
}
}
```

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/body.png">

Bước 3: Thiết lập header "Content-Type"

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/contextheader.png">

Bước 4: Gửi request

Bước 5: Kết quả trả về token và giá trị X-Subject-Token tại header response

<img src="https://github.com/VuVinh00/OpenStack/blob/master/Image/tokenunscoped.png">
