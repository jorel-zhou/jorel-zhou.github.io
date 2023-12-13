---
icon: material/ssh
---

#### SSH ~/.ssh/config file

```bash
Host *
ForwardAgent no
ForwardX11 no
ForwardX11Trusted yes
User dxjzhou
Port 22
Protocol 2
ServerAliveInterval 60
ServerAliveCountMax 30
Compression yes

Host cc-github.bmwgroup.net
  User git
  Port 22
  HostName cc-github.bmwgroup.net
  IdentityFile "C:\Users\jzhou58\.ssh\id_rsa"
  TCPKeepAlive yes
  ProxyCommand "C:\Program Files\Git\mingw64\bin\connect.exe" -S 127.0.0.1:1080 -a none %h %p

Host github.com
  User git
  Port 22
  HostName github.com
  IdentityFile "C:\Users\jzhou58\.ssh\id_rsa"
  TCPKeepAlive yes
  ProxyCommand "C:\Program Files\Git\mingw64\bin\connect.exe" -S 127.0.0.1:7890 -a none %h %p
```