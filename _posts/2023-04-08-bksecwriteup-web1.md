---
title: BKSec CTF tuyển thành viên 2023 - Writeup mảng Web
date: 2023-04-08 23:00:00 +0700
categories: [CTF Write-up]
tags: [web]
image:
    path: /assets/banner.png
---

## Echo echo ech000

### Thông tin Challenge
> I can repeat all day :)))))))))
{: .prompt-info}

### Phân tích và Hướng giải
Kết nối tới đường dẫn được cung cấp, mình có được giao diện trang web:
![web front page](/assets/img/2023-04-08-bksecwrup/web1-frontpage.png)

Response header của trang cho mình biết back-end của trang web này là Python:
![web resp header](/assets/img/2023-04-08-bksecwrup/web2-resp.png)

Khi nhập thử vào input của web, mình thấy input được trả ngay về, thử nghiệm các payload SSTI (do biết backend là Python) như `{{config}}` thì đều được trả về nguyên gốc --> SSTI không phải hướng đi của bài này.
![ssti attempt](/assets/img/2023-04-08-bksecwrup/web1-attemptssti.png)

Khi nhập thử các kí tự đặc biệt như `;`, mình thấy web không trả lại bất kì input gì, có khả năng đây là 1 bug command injection.
Thử nghiệm với command `;ls`, ta có được command injection:
![injection attempt](/assets/img/2023-04-08-bksecwrup/web1-inject1.png)

Sau đó mình lấy flag thui:
![get flag](/assets/img/2023-04-08-bksecwrup/web1-getflag.png)

```
BKSEC{c0mmAnd_InjEctiOn_is_3z_r1ght?}
```

## Hyperspeed

### Thông tin Challenge
> This website is a piece of code I wrote when I was high, and I put a secret on it. However, I forgot everything I had done. Can you help me find my secret?
{: .prompt-info}

> Hint 1: redirection
{: .prompt-info}

Truy cập vào link, mình có trang web:
![frontpage](/assets/img/2023-04-08-bksecwrup/web2-frontpage.png)

Bấm vào nút <kbd>DROP ME THERE</kbd> chỉ dẫn mình đến endpoint `/api/viewflag` với các fake flag nên không có gì để bận tâm. Dựa vào hint `redirect` thì phần nào cũng đoán được ra hướng giải là flag sẽ nằm trên url và được redirect liên tục. Đây là một số cách giải mà mình gợi ý.
- Bắt request trong development console của trình duyệt.
- Dùng Burp bắt redirect
- Tự viết code để bắt redirect + xuất thành flag
```bash
curl -v -L http://20.25.141.74:9002/3cst4sy 2>&1 | grep -i "^< location:"
```
Và mình có flag:
```bash
teebow1e@teebow1e:~$ curl -v -L http://20.25.141.74:9002/3cst4sy 2>&1 | grep -i "^< location:"
< Location: /B
< Location: /K
< Location: /S
< Location: /E
< Location: /C
< Location: /{
< Location: /9
< Location: /3
< Location: /t
< Location: /_
< Location: /h
< Location: /1
< Location: /g
< Location: /H
< Location: /-
< Location: /y
< Location: /e
< Location: /T
< Location: /}
< Location: /api/viewflag
```

## Maze Maze Mazeee

### Thông tin Challenge
> Warming up by escaping the maze is very common in ctf competitions.
{: .prompt-info}

> Hint 1: Có page-1 thì sẽ có page-2 v.v....
{: .prompt-info}

### Hướng giải
Vào web mình thấy có dòng `ARE YOU ROBOT`, check robots.txt thấy có 1 entry trong `Disallow`:
```bash
teebow1e@teebow1e:~$ curl http://20.25.141.74:9001/robots.txt
User-agent: *
Disallow: L3BhZ2VzL3BhZ2UtMS1Ub21ic3RvbmUuaHRtbA==
```

Giải mã Base64 dòng Disallow mình được 1 URL:
```bash
teebow1e@teebow1e:~$ echo L3BhZ2VzL3BhZ2UtMS1Ub21ic3RvbmUuaHRtbA== | base64 -d
/pages/page-1-Tombstone.html
```

Truy cập vào đường dẫn đã cho, ta có:
![front](/assets/img/2023-04-08-bksecwrup/web3-frontpage.png)

Và mã nguồn:
![src](/assets/img/2023-04-08-bksecwrup/web3-src.png)

Có thể thấy đây là 1 motif rất phổ biến trong CTF, tìm nhanh từ khoá `pages` ta được url thứ 2:
![find](/assets/img/2023-04-08-bksecwrup/web3-find.png)

Do đó thì task này trở thành 1 bài toán programming, tìm thấy url tiếp theo, kết nối đến, tìm và lặp lại..
Mình đã code mẫu 1 bài, các bạn có thể tham khảo:
```python
import requests
from bs4 import BeautifulSoup

link = input("enter the link: ") # enter the start link here 

while True:
    links = []
    html = requests.get(link).text
    if len(html) > 5000:
        soup = BeautifulSoup(html, 'html.parser')
    else:
        print('you probably had hit the destination, requesting to it now..')
        print(f"the full url is: {link}")
        print(requests.get(link).text)
        break

    for urls in soup.find_all('div'):
        links.append(urls.get('href'))

    for el in links:
        if el.startswith("page"):
            print(el)
            link = f"http://192.168.39.36:9001//pages/{el}" #replace this IP address with the IP provided
```

Và mình có được URL cuối cùng:
```
http://20.25.141.74:9001/pages/page-101-qncdl1248dbsl.html
```

Lúc này mình được dẫn đến round 2, một trang web có bật tính năng Directory Listing:
![dir](/assets/img/2023-04-08-bksecwrup/web3-dirlist.png)
Do số lượng file con khá là nhiều và mình tin là flag nằm trong 1 trong số những file này, mình sẽ dùng `wget` để tải hết file về, và dùng `grep` để tìm flag.
```bash
teebow1e@teebow1e:~/a$ wget -r http://20.25.141.74:9001/round2_qncdl1248dbsl/index.html
--2023-04-09 21:15:08--  http://20.25.141.74:9001/round2_qncdl1248dbsl/index.html
Connecting to 20.25.141.74:9001... connected.
HTTP request sent, awaiting response... 200 OK

...

FINISHED --2023-04-09 21:17:39--
Total wall clock time: 43s
Downloaded: 187 files, 24K in 0.03s (685 KB/s)
```

```bash
teebow1e@teebow1e:~/a/20.25.141.74:9001/round2_qncdl1248dbsl$ grep -r BKSEC *
vxbxqulpft/ngwmgtipii/opcchgnohr.html:BKSEC{fake fake fake}
wpknccrgyj.html:BKSEC{1+1=3}
wyzzrsdxhn/yoeipptofm/ezafnnisfk.html:BKSEC{D0_u_l1ke_the_m4ze_RUnn3r?_31x585ha9u}
```