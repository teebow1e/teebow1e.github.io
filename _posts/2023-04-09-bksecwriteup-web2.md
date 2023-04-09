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
Vào trang web chúng ta có 1 form cơ bản:
![front](/assets/img/2023-04-09-bksecweb2/feedback-front.png)
Khi submit form, server đã generate ra 1 trang web với những thông tin đã được truyền vào:
![submitted](/assets/img/2023-04-09-bksecweb2/feedback-sample.png)

Ở thanh navigation bar, chúng ta có 1 endpoint là `Admin Panel`, tuy nhiên bấm vào thì bị lỗi Forbidden.

Bài này có cho chúng ta một file zip chứa mã nguồn, giải nén file ra, và ta có các file:
```bash
teebow1e@teebow1e:~/feedback$ tree
.
└── Feedback_Form
    ├── Dockerfile
    ├── app.py
    ├── build-docker.sh
    ├── docker-compose.yml
    ├── requirements.txt
    ├── static
    │   └── feedbacks
    └── templates
        ├── errors
        │   ├── content_error.html
        │   └── subject_error.html
        └── index.html

5 directories, 8 files
```
Mình có file `app.py`:
```python
from werkzeug.urls import url_fix
from secrets import token_urlsafe
from flask import Flask, request, abort, render_template, redirect, url_for
from unidecode import unidecode
import os, re

app = Flask(__name__)
app.config['FLAG'] = os.environ['FLAG']

def findWholeWord(w):
    return re.compile(r'\b({0})\b'.format(w), flags=re.IGNORECASE).search

bad_chars = "'_#&;+"
bad_cmds = ['import', 'os', 'popen', 'subprocess', 'env', 'print env']

@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))

@app.route('/admin', methods=['GET'])
def admin():
    return abort(403)

@app.route("/feedback", methods=["POST"])
def create():
    username = request.form.get("username", "")
    subject = request.form.get("subject", "")
    content = request.form.get("content", "")
    if len(subject) > 128 or "_" in subject or "/" in subject:
        return redirect(url_for("index", error="subject_error"))
    if "_" in content or "." in content or len(content) > 512:
        return redirect(url_for("index", error="content_error"))
    if any(char in bad_chars for char in content):
        return redirect(url_for("index", error="content_error"))
    for cmd in bad_cmds:
        if findWholeWord(cmd)(content):
            return redirect(url_for("index", error="content_error"))
    name = "static/feedbacks/" + url_fix(unidecode(subject).replace(" ", ""))  + token_urlsafe(16) + ".html"
    with open(name, "w") as f:
        feedback = f'''<!DOCTYPE html>
<html lang="en">
  <head>
    << STRIPPED >>
  </head>
  <body>
    << STRIPPED >>

    <div class="container">
      <div class="text-center mt-5">
        <h1>From {username}</h1>
        <p class="lead">{subject}</p>
      </div>
      <div class="container">
        <div class="alert alert-error" role="alert">
          <p>{content}</p>
        
    << STRIPPED >>
  </body>
</html>'''
        f.write(feedback)
    return redirect(name)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=os.environ['PORT'])
```

Qua phân tích code, mình thấy được:
- Backend của web là Flask, FLAG được lưu ở trong biến môi trường và config của webapp. <br>
    → Khả năng cao đây là 1 web-app bị dính lỗi SSTI.
- Các ký tự bị filter: '_#&;+ <br>
    → làm cho quá trình tìm payload khó khăn hơn
- Các word bị filter: 'import', 'os', 'popen', 'subprocess', 'env', 'print env' <br>
    → không cho phép người dùng import các thư viện để tương tác với server → Không có RCE.

- Subject không được chứa dấu _ hoặc dấu /, không được dài quá 128 ký tự.
- Content không được dài quá 512 ký tự, không được chứa các word/ký tự đặc biệt bị filter, dấu _, dấu chấm.

- Sau đó, username, subject, content được truyền vào template trong biến `feedback` và được lưu vào thư mục `/static/feedbacks`, tên gồm subject (user có thể control được), 16 ký tự ngẫu nhiên và đuôi `.html`
- Sau khi viết vào file, server sẽ redirect đến trang feedback với thông tin như đã nhập vào.

### Hướng giải
Như đã đề cập trong phần phân tích, sau khi nhập vào thông tin, user bị *redirect* đến trang đích, nên code sẽ không thể được thực thi. (ở đây mình nhập thử payload vào phần content):
![ssti1](/assets/img/2023-04-09-bksecweb2/feedback-sstiattempt.png)

Chính vì thế, mình cần phải tìm cách truyền payload vào hàm `render_template`, và trong code có đúng 1 phần có sử dụng hàm này.
```python
@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))
```

Bên cạnh đó, trong file HTML cũng có 1 đoạn code khá thú vị:
{% raw %}
```html
    <!-- Page content-->
    <div class="container">
      <div class="text-center mt-5">
        <h1>Feedback form for Watermelon clan</h1>
        <p class="lead">We hope you have a good time with us!</p>
      </div>
    {% if error is not none %}
    <div class="container">
      <div class="alert alert-error" role="alert">
        <p>ERROR: {{ error }}</p>
        {% include "errors/" + error + ".html" ignore missing %}
      </div>
    </div>          
```
{% endraw %}

Trong đoạn code này, nếu trang web phát sinh error, web sẽ redirect tới một trang nêu tên lỗi và **render** cả description của lỗi. File description này được lưu ở `/templates/errors`. Vậy mục tiêu sẽ là ghi payload vào trong thư mục này và render qua parameter `error`.

Trong app.py, có một lỗ hổng khi khai báo biến `name`:
```python
name = "static/feedbacks/" + url_fix(unidecode(subject).replace(" ", ""))  + token_urlsafe(16) + ".html"
```
Trong biến `name` đã bao gồm cả đường dẫn, và user có thể điều khiến được biến subject. Và chúng ta có... Path Traversal.

Payload mình sử dụng cho phần **subject** sẽ là:
```
..\..\templates\errors
```

Phần payload SSTI mình sẽ sử dụng 1 payload bypass được tất cả các điều kiện ở trên:
{% raw %}
```
{{ self|attr("\x5f\x5fdict\x5f\x5f") }}
```
{% endraw %}

Và đem đi submit form thôi:
![submit](/assets/img/2023-04-09-bksecweb2/feedback-submit.png)

Sau khi submit, chúng ta được 1 lỗi `Not Found`, lỗi này là do mình đã Path Traversal thành công.
![err](/assets/img/2023-04-09-bksecweb2/feedback-error.png)

Lúc này chỉ cần lấy phần tên trước `.html`, và chèn vào `?error=` là ta sẽ có flag.
![obtain-flag](/assets/img/2023-04-09-bksecweb2/feedback-res.png)

```
BKSEC{un4bl3_t0_imp0rt_1s_4_pa1n_in_th3_@ss_46dba925}
```