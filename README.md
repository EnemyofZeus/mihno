# mihno
A simple TCP server honeypot written in Rust compatible with raw TCP and HTTP clients.

Example build steps with docker:

```
git clone https://github.com/jpegleg/mihno && cd mihno
cargo build --release
docker build -t "myprivateregistryplace:5000/mihno:1" .
```

Example build steps, reconfiguring from default port 3975 to port 389, with systemd and a log file in /var/log/ on linux (using sudo):

```
git clone https://github.com/jpegleg/mihno && cd mihno
sed -i 's/3975/389/g' src/main.rs
cargo build --release
sudo cp target/release/mihno /usr/sbin/mihno.bin
sudo echo "#!/usr/bin/env bash" > /usr/sbin/mihno
sudo echo "/usr/sbin/mihno.bin >> /var/log/mihno.log 2> >> /var/tmp/mihno.err" > /usr/sbin/mihno
sudo chmod +x /usr/sbin/mihno
sudo cp mihno.service /usr/lib/systemd/system/ && sudo systemctl enable mihno
sudo systemctl start mihno
```

There is a fun optional thing I have in this template that is a SHA256 of a blob from a PRNG that is an additional
version tag. 

As usual for rust, `cargo build --release` to compile it. Then the binary in target/release/mihno can be packaged or otherwise deployed etc.

Raw TCP clients like telnet and netcat can send in data, as well as HTTP clients
like cURL etc.

```
$ telnet mihno-host 3975
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
funky data and stuff
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

<html><body><h1>H A R V E S T E D </h1></br></br><p>performance +</br></br>a2c727ee8f206913df426b6bd29d7727bf19a10229466edfc349812388c911bd</p></body></html>
Connection closed by foreign host.
```


The output goes to STDOUT, so redirect to where you need etc.

In this example output, we can see the telnet data in the middle there. The first transaction is a GET / from a web browser,
the second a raw TCP (from our telnet above), and the third a curl doing a POST.

```
2021-12-17 17:13:13.744257569 UTC Listening for connections on port 3975
2021-12-17 17:13:19.679084778 UTC 72a6c16d-75ad-4c6a-bed1-c9c6ce143845  _--->_ start transaction
2021-12-17 17:13:19.679346530 UTC 192.168.1.133:59677 GET / HTTP/1.1
Host: mihno-host:3975
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:95.0) Gecko/20100101 Firefox/95.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: keep-alive
Cookie: jenkins-timestamper-offset=21600000
Upgrade-Insecure-Requests: 1
Sec-GPC: 1
Cache-Control: max-age=0


2021-12-17 17:13:19.679460707 UTC Response sent _<---_ end transaction
2021-12-17 17:13:39.620908274 UTC de379029-0fd9-4e72-9874-9a3ba277b3a9  _--->_ start transaction
2021-12-17 17:13:39.620966387 UTC 127.0.0.1:49364 some funky tcp data sent in

2021-12-17 17:13:46.974986805 UTC Response sent _<---_ end transaction
2021-12-17 19:09:17.934566878 UTC 60bd9d55-9784-4846-a10c-0af23cd6c396  _--->_ start transaction
2021-12-17 19:09:17.934707002 UTC 127.0.0.1:59622 POST / HTTP/1.1
Host: localhost:3975
User-Agent: curl/7.74.0
Accept: */*
Content-Length: 27
Content-Type: application/x-www-form-urlencoded

honk honk message body post
2021-12-17 19:09:17.934791245 UTC Response sent _<---_ end transaction

```


M I H N O can be configured with different ports. It has proven very useful with port 389 and 53, it can detect log4shell for example. An LDAP bind request comes through as capital o letter and then a back tick, as such when viewing invisible characters: 

```
0^L^B^A^A`^G^B^A^C^D^@�
```
That blob (any data the client sends in shorter than 4096 unless a new line is reached) is then followed by NULLs filling the rest of the buffer as well `^@` in M I H N O that default terminals will read as blank spaces unless you view "invisible" characters. The buffer set is 4096, that can be adjusted as needed in main.rs. Any data longer or after the 4096 is not read, and a new line or return sequence will also terminate the stream unless the client starts a new TCP connection.

An example of log4shell LDAP bind vs a regular bind, here is the log4shell bind:

```
2021-12-18 00:09:23.417878259 UTC bc36e55d-3cdb-44ca-8a64-b7616569e75b  _--->_ start transaction
2021-12-18 00:09:23.418116772 UTC 127.0.0.1:40446 0
                                                       `�
2021-12-18 00:09:23.418259503 UTC Response sent _<---_ end transaction
```

And here is an example from Apache Directory Studio, a valid LDAP authenticated BIND request for user admin password admin (that would exit in error on the client side when the client receives the mihno response to its LDAP bind request, unless otherwise adjusted):

```
2021-12-18 03:51:12.493294279 UTC b8b275e0-0637-4778-a48b-73dad564ca2c  _--->_ start transaction
2021-12-18 03:51:12.493480487 UTC 127.0.0.1:57685 0`admin�admin
2021-12-18 03:51:12.545367898 UTC Response sent _<---_ end transaction
```

The IP, ephemeral port, headers, and raw data from the client connection (vulnerable host to log4shell for example as the client) are captured in STDOUT of M I H N O. This means raw binary data types are preserved, via threads of simple Rust. The output is to be used as a stream of honeypot data, likely processed by other programs.

