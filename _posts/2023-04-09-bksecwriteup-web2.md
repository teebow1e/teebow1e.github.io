---
title: BKSec CTF tuyển thành viên 2023 - Writeup mảng Web (part2)
date: 2023-04-09 00:00:00 +0700
categories: [CTF Write-up]
tags: [web]
---

## Poem

### Thông tin Challenge
> Cô A đã mở 1 trang web để việc dạy học của mình trở nên hiệu quả hơn. Tuy vậy trang web có 1 bí mật đằng sau. Bạn có thể tìm ra nó?
{: .prompt-info}

### Phân tích
![frontpage](/assets/img/2023-04-09-bksecweb2/poem-frontpage.png)
Khi vào trang web, hầu như k có tính năng nào có thể sử dụng được.

Check qua robots.txt, có thể thấy file này không bị ẩn và show ra cây thư mục của server:
```bash
teebow1e@teebow1e:~$ curl http://20.25.141.74:9000/robots.txt
.
├── Dockerfile
├── README.md
├── app.py
├── build-docker.sh
├── docker-compose.yml
├── phlag.txt
├── poems
│   ├── bacoi.txt
│   ├── danguitarcualorca.txt
│   ├── datnuocndt.txt
│   ├── datnuocnkd.txt
│   ├── song.txt
│   ├── taytien.txt
│   └── vietbac.txt
├── requirements.txt
├── static
│   └── robots.txt
└── templates
    └── index.html
```
Qua cây thư mục này, có thể tạm thời xác định mục tiêu là đọc file `phlag.txt`. Tác giả còn cho chúng ta file code mã nguồn của server:
```python
from flask import Flask, request, Response, render_template
import random
import re
import os

app = Flask(__name__, static_folder='static', static_url_path='')

@app.route('/poem')
def getPoem():
    poem = request.args.get('baitho')
    taytien = './poems/taytien.txt'
    vietbac = './poems/vietbac.txt'
    datnuocnkd = './poems/datnuocnkd.txt'
    datnuocndt = './poems/datnuocndt.txt'
    danguitarcualorca = './poems/danguitarcualorca.txt'
    bacoi = './poems/bacoi.txt'
    song = './poems/song.txt'

    if (not poem or re.search(r'[^A-Za-z\.]', poem)):
        return 'Em hãy nhập đúng tên của bài thơ trong chương trình Ngữ văn lớp 12. Nếu em tiếp tục có những hành vi tấn công Website, cô sẽ ghi em vào sổ đầu bài!'

    with open(eval(poem), 'r', encoding='utf-8') as f:
        return render_template('index.html', poem=f.read())

@app.route("/")
def index():
    return render_template('index.html', poem='Em hãy chọn 1 bài thơ em thích.')

if (__name__ == '__main__'):
    app.run(host='0.0.0.0', port=os.environ['PORT'])
```

Có thể thấy, trang web này có duy nhất 1 route `/poem`, nhận input ở parameter `baitho`, sau đó input này sẽ được lưu ở trong biến `poem`. Ngoài ra, mình còn thấy có 1 hàm `eval` ở line 22, vậy thì mục tiêu sẽ là tìm cách chèn command vào biến `poem` để inject vào hàm `eval`.

Tuy nhiên biến `poem` lại bị filter bởi đoạn regex `[^A-Za-z\.]`. Đoạn regex này sẽ chỉ cho phép chữ cái từ A->Z (hoa và thường) và dấu chấm. 

Trong Python, để gọi 1 chức năng nào đó, chúng ta có thể gọi qua function, method, attribute. Tuy nhiên function và method đều yêu cầu cặp dấu `()`. Do vậy chúng ta chỉ có thể gọi hàm qua các attribute được tích hợp trong các Class. Cụ thể ở đây là class `Request` của `Flask`.

Ở trong [trang web này](https://tedboy.github.io/flask/generated/generated/flask.Request.html#attributes) có chỉ ra rất nhiều attribute của Flask Request. Tuy nhiên trong số đó chỉ có `request.data` là có thể fetch dữ liệu từ nơi khác được.

### Hướng giải
Để có thể sử dụng `request.data`, mình cần phải chỉ ra dữ liệu mình muốn fetch ở trong phần request. Mình thực hiện fetch file `phlag.txt` qua Burp:
![fetch](/assets/img/2023-04-09-bksecweb2/poem-requestdata.png)

```
phlag ở khắp mọi nơi.. điều đó nghĩa là gì nhỉ?
```

Flag ở khắp mọi nơi ám chỉ việc flag được lưu trữ ở trong các biến môi trường. Do mình chỉ có LFI chứ không có RCE nên để leak các biến môi trường, mình sẽ fetch file `/proc/self/environ`:
![flag](/assets/img/2023-04-09-bksecweb2/poem-procself.png)

```
BKSEC{p0em_1s_4_g00d_w4y_t0_s4y_ily}
```

## Feedback Form

### Thông tin Challenge
> Câu lạc bộ Anh em Dưa Hấu đang muốn mở rộng địa bàn, và rất cần những góp ý của các huynh đệ khác. Tuy nhiên, họ không sử dụng các dịch vụ khảo sát online mà lại tự build. Họ sợ bị theo dõi bởi cục F giấu tên. Liệu đây có phải một quyết định đúng?
{: .prompt-info}

### Phân tích
