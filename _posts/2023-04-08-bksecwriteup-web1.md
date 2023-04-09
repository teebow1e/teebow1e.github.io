---
title: BKSec CTF tuyển thành viên 2023 - Writeup mảng Web
date: 2023-04-08 23:00:00 +0700
categories: [CTF Write-up]
tags: [web]
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

Khi nhập thử vào input của web, mình thấy input được trả ngay về, thử nghiệm các payload SSTI (do biết backend là Python) như `{{2+2}}` thì đều được trả về nguyên gốc --> SSTI không phải hướng đi của bài này.
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
1. Bắt request trong development console của trình duyệt.
2. Dùng Burp bắt redirect
3. Tự viết code để bắt redirect + xuất thành flag
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
> TODO: cài docker đã =))
{: .prompt-warning}