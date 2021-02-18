# lets-make-raspy-dns-20210218

# Raspberry Pi で DNS サーバーを作ろう !

## GIT のインストール

```shell=
$ sudo apt-get install -y git
$ git clone https://github.com/JunyaKaneko/lets-make-raspy-dns-20210218.git
```

## DNS Resolver の構築

BIND9 の設定ファイルのあるディレクトリに移動する.

```shell=
$ cd lets-make-raspy-dns-20210218/conf
$ ls
```

```ls``` した結果でてくる Clone したプロジェクトの中の BIND9 の設定ファイルは以下の通り.

```
db.192.168.1  db.mydomain  named.conf.local  named.conf.options
```

BIND9 の設定ファイルは, ```/etc/bind``` にあるので, その中に入っている設定ファイルのリストもみておく.

```shell=
$ ls /etc/bind
```

```ls``` の結果は以下の通り.

```
bind.keys  
db.127  
db.empty  
named.conf 
named.conf.local    
rndc.key
db.0       
db.255  
db.local  
named.conf.default-zones  
named.conf.options  
zones.rfc1918
```

```named.conf.options``` を開いて編集する.

```shell=
$ sudo vi named.conf.options
```

```
options {
	directory "/var/cache/bind";

	// If there is a firewall between you and nameservers you want
	// to talk to, you may need to fix the firewall to allow multiple
	// ports to talk.  See http://www.kb.cert.org/vuls/id/800113

	// If your ISP provided one or more IP addresses for stable 
	// nameservers, you probably want to use them as forwarders.  
	// Uncomment the following block, and insert the addresses replacing 
	// the all-0's placeholder.

	// forwarders {
	// 	0.0.0.0;
	// };

	//========================================================================
	// If BIND logs error messages about the root key being expired,
	// you will need to update your keys.  See https://www.isc.org/bind-keys
	//========================================================================
	dnssec-validation auto;

	listen-on-v6 { any; };
    listen-on { any; };

	allow-query { trusted; }; // クエリを許可する IP# 
	allow-query-cache { trusted; }; // キャッシュへのクエリを許可する IP#

	recursion yes; // 外部の DNS への問い合わせを行う
	allow-recursion { trusted; }; // 外部の DNS への問い合わせを許可する IP #
};

// ① DNS サーバーの利用を許可する IP# のリスト
acl trusted {
	192.168.1.0/24; // 自分の LAN の設定に合わせる.
    127.0.0.1;
};
```

```/etc/resolv.conf``` を編集し, DNS Resolver を Raspberry Pi に設定する.

```shell=
$ sudo vi /etc/resolv.conf
```

```/etc/resolv.conf``` を下記一行だけにする.

```
nameserver 127.0.0.1
```

```nslookup``` コマンドを実行してインターネット上に存在するドメイン名と IP# の変換ができないことを確認する.

```shell=
$ nslookup google.com
```

```
;; connection timed out; no servers could be reached
```

先ほど Resolver として設定した BIND9 を起動する.

```shell=
$ sudo service bind9 start
```

しばらく待ってから ```nslookup``` コマンドを実行する.

```shell=
$ nslookup google.com
```

下記のような応答が得られたら成功!

```
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	google.com
Address: 172.217.161.46
Name:	google.com
Address: 2404:6800:4004:80a::200e
```

これで DNS Resolver が完成した.

## Authoritative Server の構築

```/etc/bind9/named.conf.local``` を開いて編集する.

```shell=
$ sudo vi named.conf.local
```

```named.conf.local``` には, ドメイン名から IP# を導くために利用する設定ファイルと, IP# からドメイン名を導くための設定ファイルの場所を記載する.

```
// ドメイン名から IP# を導く設定ファイルの場所.
// ドメイン名は自分の好きなものを利用.
// ただし, インターネット上に存在しないものにすること.
zone "mydomain" {
	type master;
	file "/etc/bind/db.mydomain";
};

// IP# からドメイン名を導く設定ファイルの場所.
// IP# は自身の LAN に合わせて適宜変更.
zone "1.168.192.in-addr.arpa" {
	type master;
	file "/etc/bind/db.192.168.1";
};
```

```/etc/bind/db.mydomain``` を作成.

```shell=
$ sudo vi db.mydomain
```

```
$TTL	604800
@	IN	SOA	m2.mydomain. hostmaster.mydomain. (
			2021021801      ; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	m2.mydomain.
m1	IN	A	192.168.1.1
m2	IN	A	192.168.1.2
m3	IN	A	192.168.1.3
m4	IN	A	192.168.1.4
m5	IN	A	192.168.1.5
m6	IN	A	192.168.1.6
m7	IN	A	192.168.1.7
m8	IN	A	192.168.1.8
m9	IN	A	192.168.1.9
m10	IN	A	192.168.1.10
dns	IN	A	192.168.1.50
```

エラーがないか下記コマンドで検証する.

```shell=
$ sudo named-checkzone m1 db.mydomain
```

次に, ```/etc/bind/db.192.168.1``` を作成.

```shell=
$ sudo vi db.192.168.1
```

```
$TTL	604800
@	IN	SOA	dns.mydomain. hostmaster.mydomain. (
			2021021801	; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	dns.mydomain.
1	PTR	m1.mydomain.
2	PTR	m2.mydomain.
3	PTR	m3.mydomain.
4	PTR	m4.mydomain.
5	PTR	m5.mydomain.
6	PTR	m6.mydomain.
7	PTR	m7.mydomain.
8	PTR	m8.mydomain.
9	PTR	m9.mydomain.
10	PTR	m10.mydomain.
50	PTR	dns.mydomain.
```

エラーがないか下記コマンドで検証する.

```shell=
$ sudo named-checkzone 1 db.192.168.1
```

エラーがなければ, ```nslookup``` コマンドを実行して, まだ ```m1.mydomain``` が発見できないことを確認する.

```shell=
$ sudo nslookup m1.mydomain
```

```
Server:		127.0.0.1
Address:	127.0.0.1#53

** server can't find m1.mydomain: SERVFAIL
```

```bind9``` を再起動する.

```shell=
$ sudo service bind9 restart
```

```m1.mydomain``` の IP# が ```192.168.1.1``` であることを確認する.

```shell=
$ nslookup m1.mydomain
```

```
Server:		127.0.0.1
Address:	127.0.0.1#53

Name:	m1.mydomain
Address: 192.168.1.1
```

IP# ```192.168.1.1``` が ```m1.mydomain``` であることを確認する.

```shell=
$ nslookup 192.168.1.1
```

```
1.1.168.192.in-addr.arpa	name = m1.mydomain.
```

## 自身のコンピュータの DNS を Raspberry Pi に設定してみる.

### Windows用の設定

https://pc-karuma.net/windows-10-change-dns-settings/

### MAC 用の設定

![](https://i.imgur.com/qo0qgTb.png)
![](https://i.imgur.com/uAk72gr.png)
![](https://i.imgur.com/9AbZ7Zm.jpg)



Raspberry Pi にドメイン名を用いて ping を発信する.

```shell=
$ ping dns.mydomain
```

```
PING dns.mydomain (192.168.1.50): 56 data bytes
64 bytes from 192.168.1.50: icmp_seq=0 ttl=64 time=636.986 ms
64 bytes from 192.168.1.50: icmp_seq=1 ttl=64 time=1.212 ms
```
