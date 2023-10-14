> Hiện nay, python đang ngày càng có nhiều sức ảnh hưởng trong thế giới lập trình, không chỉ vì tính đươn giản, dễ nắm bắt và ứng dụng trong nhiều lĩnh vực khác nhau (Web, AI, Desktop App, Security, ...). Mà còn vì cộng đồng hỗ trợ cho python và các framework của python. Vì vậy ngày cang nhiều các thư viện, tiện ích được đi kèm với python nhằm mục đích hô trợ cho lập trình viên trong quá trình phát triển các dự án với ngôn ngữ này. Tuy nhiên cùng với đó là các mối nguy hại gia tăng từ chính những thư viện, tiện ích này. Những lỗ hổng từ những tiện ích đó có thể giúp hacker dễ dàng tấn công, khai thác những ứng dụng được xây dựng dựa trên chúng. Werkzeug cũng không ngoại lệ! Trong bài viết này tôi sẽ giới thiệu về Werkzeug cũng như môt trong nhưng mối nguy hiểm tiềm tàng có thể bị khai thác trong quá trình sử dụng Werkzeug mang tên `KeyGen Werkzeug Debugger Console Pin`
# 1. Giới thiệu tổng quan
- Werkzeug là một thư viện cho ứng dụng web theo chuẩn WSGI. Được framework Flask sử dụng cùng với template engine Jinja2 để phát triển website.
- Flask là một micro-framework được viết bằng ngôn ngữ lập trình python dùng để phát triển website. Bên cạnh Flask còn có nhiều framework mạnh và cũng được viết bằng Python khác như Django, Pyramid,...
- Vì là một micro-framework, vì vậy nó sẽ chỉ gồm nhưng thành phần cơ bản nhất để xây dựng một trang web, các thành phân khác sẽ được thêm qua các extension tùy thuộc nhu cầu của lập trình viên. Về cách cài đặt, sử dụng và triển khai, ta có thể tham khảo trong trang chủ của Flask tại [đây](https://flask.palletsprojects.com/en/2.3.x/).
> Vậy WSGI là gì ? Tại sao Werkzeug lại được sử dụng làm base cho các dự án Flask ?
# 2. WSGI là gì ? Tại sao lại phải sử dụng WSGI
- Như ta đã biết, trong vận hành trang web, quá trình nhận gói tin http request và gửi http response là những quá trình tất yếu. Tuy nhiên việc xử lý tất cả những vấn đề xung quanh chúng là một công việc tương đối phức tạp.
- Thông thường, để giải quyết vấn đề này ta có thẻ sử dụng các dịch vụ web server như Apache, Nginx. Với những trang web viết bằng python framework như Flask, Django, .. nếu không có những quy tắc hoặc những chuẩn chung về việc tương tác này; việc vận hành, web python sẽ trở nên lộn xộn và với mỗi dịch vụ web server khi kết hợp với python framework lại gặp phải cùng 1 vấn đề:
	- Làm sao để ứng dụng với framework A có thể giao tiếp với dịch vụ web server B.
- Vì vậy WSGI đã được sinh ra với vai trò như một chuẩn chung để giải quyết vấn đề trên.
![|600](./images/Pasted%20image%2020230828085757.png)
- Từ sơ đồ trên ta thấy rõ vai trò của WSGI server trong Backend: 
	- Xử lý, phân tích các gói tin http request thành dạng đối tượng để Application có thể hiểu được.
	- Chuyển các đối tượng phản hồi từ Application thành http response để web server có thể chuyển tới người dùng.
- Trong đó, Werkzeug WSGI là một thư viện được xây dựng để hỗ trợ Flask theo chuẩn WSGI. Chi tiết về chức năng cũng như vai trò của WSGI, có thể tham khảo thêm tại [đây](https://wsgi.readthedocs.io/en/latest/index.html)

# 3. Tìm hiểu về Werkzeug và chức năng debugger
## 3.1. Werkzeug là gì
- Là một thư viện xây dựng ứng dụng web theo WSGI.
- Werkzeug cung ấp một loạt các tiện ích cho việc phát triển các ứng dụng tuân thủ WSGI. Những tiện ích này thực hiện những công việc đặc thù trong quá trình xử lý gói tin thành đối tượng như: parsing header, gửi và nhận cookie, cho phép truy cập từ form dữ liệu, tạo điều hướng, tạo các error page (trang lỗi) khi xuất hiện ngoại lệ, cung cấp trình gỡ lỗi (debugger) chạy trên browser.
## 3.2. Chức năng debugger trong werkzeug
- Trong report này, chúng ta tập trung vào việc làm sao để gen console pin của trình gỡ lỗi (debugger) trên Werkzeug vì vậy tôi sẽ chỉ giới thiệu và đi sâu vào chức năng debugger của Werkzeug.
### Khi nào ta sử dụng debugger và sử dụng trong môi trường nào
- Thông thường, khi ứng dụng web xuất hiện lỗi trong quá trình triển khai và phát triển, hầu hết các exception sẽ được đưa đến stderr hoặc được lưu vào log lỗi. Về phía giao diện sẽ chỉ xuất hiện cảnh báo `500 Internal Server Error`
![|600](./images/Pasted%20image%2020230828092102.png)
- Trong môi trường phát triển, rõ ràng việc nhận các exception kiểu này không phải giải pháp tốt để có thể giúp dev gỡ lỗi, vì vậy Werkzeug đã cung cấp một chức năng giúp ứng dụng có thể hiện thị rõ ràng các traceback cũng như cung cấp một trình console debug có thể thực thi python code ở bất cứ frame nào.
### Cách kích hoạt và sử dụng
- Để có thể bật chế độ debugger trong ứng dụng, ta cần đóng gói/bọc ứng dụng đó trong 1 middleware là **DebuggedApplication**
```python
_class_ werkzeug.debug.DebuggedApplication(_app_, _evalex=False_, _request_key='werkzeug.request'_, _console_path='e/console'_, _console_init_func=None_, _show_hidden_frames=False_, _pin_security=True_, _pin_logging=True_)
```
- Sử dụng **DebuggedApplication** với 1 ứng dụng web như sau:
```python
from werkzeug.debug import DebuggedApplication
from myapp import app
app = DebuggedApplication(app, evalex=True)
```
- Có rất nhiều tham số trong lớp `DebuggedApplication` với những chức năng khác nhau hỗ trợ cho lập trình viên, các bạn có thể tham khảo kỹ hơn tại [đây](https://werkzeug.palletsprojects.com/en/2.3.x/debug/)
- Để kích hoạt được debugger này thì người dùng cần launch server ở Debug mode. Tuy nhiên với việc cho phép thực thi câu lệnh Python từ xa có khả năng bị khai thác như một RCE, Werkzeug debugger console được bảo vệ bởi một mã PIN dưới dạng XXX-XXX-XXX với X là một chữ số 0-9. Khi launch server, mã PIN cho console được hiển thị ngay trên terminal để bảo đảm chỉ có developer mới có quyền truy cập vào console:
![|500](./images/Pasted%20image%2020230828093113.png)
- Khi xảy ra lỗi, giao diện sẽ xuất hiện traceback như sau:
![|600](./images/Pasted%20image%2020230828093227.png)
- Để sử dụng chức năng debug console, ta truy cập vào đừng dẫn `/console`, lúc này debugger sẽ được bảo vệ bởi mã PIN thì khi một người dùng truy cập vào debugger sẽ bị yêu cầu PIN như hình:
	![|600](./images/Pasted%20image%2020230828093403.png)
	- Khi sử dụng trình debug console, chúng ta sẽ gặp 1 yêu cầu mã PIN bảo vệ chúng. Đây là chức năng bảo mật để hạn chế việc bị khai thác, tấn công nếu chúng ta quên không vô hiệu hóa console khi triển khai lên sản phẩm.
	- Mặc định của Debugger PIN là enable
	- Giá trị của mã PIN được khai báo thông qua biến môi trường `WERKZEUG_DEBUG_PIN`. Giá trị này là số nếu muốn sử dụng mã PIN và `off` nếu muốn vô hiệu hóa nó.
	- Nếu như mã PIN bị nhập sai quá nhiều lần, server sẽ cần phải khởi động lại

> [!RISK]
> - Như đã nói ở trên, nếu ứng dụng web python được bọc bởi lớp `DebuggedApplication`, ta có thể truy cập vào debug console thông qua đường dẫn `/console` để thực hiện debug, lúc này chương trình sẽ cho phép chúng ta thực hiện hiện các câu lệnh python ngay trong ứng dụng.
> - Vậy điều gì sẽ xảy ra nếu như lập trình viên đưa ứng dụng lên product nhưng lại quên không disable chức năng debug console. Liệu mã PIN có đủ sức bảo vệ trước những cuộc tấn công bên ngoài. Và điều gì sẽ xảy ra nếu như hacker có thể vượt qua được mã PIN và chiếm được quyền điều khiển console. Tất cả sẽ có trong phần tiếp theo của article. Các bạn hãy cùng đón đọc nhé !