通过proxy调用curl下载文件, 在共享屏幕的时候```-u```后只输入用户名

```bash
curl -fail -v -x "https://{your_proxy_ip}:{your_proxy_port}" \
{your_url} -o wendows.zip \
-u {your_username}:{your_password}
```