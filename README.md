---
layout: page
title: SP16MP3
permalink: /MP3/
---

* TOC
{:toc}

# Machine Problem 3 - Synchronization between Client and Server

- Due Date: 11/22 23:59

- Late submission: We will use the following formula to calcuate your final
    score in this assignment.
$$Score(Raw) = \begin{cases}
RawScore & \text{if you submit before the due date.}\\
RawScore \times \max(0, (1 - 0.05 \times \lceil\frac{DelaySeconds}{86400}\rceil)) & \text{otherwise.}
\end{cases}$$

If you have any problem for this homework, you can [submit an issue](https://github.com/SystemProgrammingatNTU/SP16Issues/issues).

---

## Background
In this homework, you are going to build one of the most important feature of CSIEBOX. 
That is, allowing user to upload files and directories in local repository to remote server,  and later download them into new client device. Also, you will learn how to monitor change of local files and directories in client side and synchronize with server.

Moreover, you will learn how to enable your sever to handle requests from multiple clients with I/O multiplexing.



## Communication
In this homework, you need to wite two programs, csiebox_client and csiebox_server, and they need to communicate with each other through Internet. To achieve this, network socket is needed, which is beyond the scope of this class. But don’t worry, we have built all you need for communication, all you have to do is follow our instructions and protocols. If you are interested in socket, information can be found in Resource section.

### How to Connect Client to Server
First thing to do in this MP is to connect your client application to server and login. We have done the code of this part for you, you only need to modify the configuration files so that it can work on your machine. There are three configuration files you need to modify: server.cfg, client.cfg and account. 


Server.cfg contains information of server, and it has two fields, path and account_path. Path is the path of server directory where we will store files and directories from our users, and account_path is the path of the configuration file ‘account’. 


Client.cfg contains information of client, and it has five fields. Name is the user name of your local device, for example, it will be your student ID if you run it on workstation. Server is the ip address of the machine that runs your server application. It will be ‘localhost’ if both client and server are run on the same machine. Else you can use the command ‘ifconfig’ to get the ip address. User and passwd is the user name and password of the user that run the client. You can register new user by adding pair of ‘username,passwd’ to the configuration file ‘account’. Path is the path of client directory, where stores files and directories we are going to synchronize.


Account contains information of registered user on the server. Pairs of ‘username, passwd’ is stored in this file. You can register new user by adding pair of ‘username,passwd’ to this file. Each user will have its own directory on server, which will be ‘/path/to/server/dir/username’ once it uploads some files to server.


Once you modified those configuration files, you can try to connect client to server. Compile the code provided by TA, and four binary will be generated: csiebox_server, csiebox_client, port_register, and inotify_test. csiebox_server and csiebox_client is binary for server and client respectively. Port_register is used to register port number of server so that client can connect to it through socket so that you have to run port_register before csiebox_server and csiebox_client. TA will run port_register on workstation linux1~15 for you so that you don’t have to run it yourself. 
Once port_register is up, you can run csiebox_server and csiebox_client to see if they can communicate to each other. Csiebox_client will try to longin to the server with account information provided in client.cfg and csiebox_server will verify those information. You can check output of these binary to see if the client login successfully.

### Interface and Protocol
To send message to or receive message from other program, you can use send_message() or recv_message() provided by TA. They function just like read() and write(), but instead of local file, they work on socket. You can fin example of using them in the code provided by TA.


To make communication between client and server simple and efficient, we provide some data structures that define format of message and protocol that defines some actions that client and server can do. You can find the data structures in csiebox_common.h. You can find some examples in the code we provided. 

```
typedef union {
  struct {
    uint8_t magic;
    uint8_t op; 
    uint8_t status;
    uint16_t client_id;
    uint32_t datalen;
  } req;
  struct {
    uint8_t magic;
    uint8_t op; 
    uint8_t status;
    uint16_t client_id;
    uint32_t datalen;
  } res;
  uint8_t bytes[9];
} csiebox_protocol_header;
```
Struct csiebox_protocol_header is a base header for communication protocol between client and server, defined in csiebox_common.h.
It consists of five attributes.
- Attribute Magic, defined for security purpose
- Attribute Op defines operation code.
- Attribute Status defines return status.
- Attribute Client_id defines the identification of client processes, which will be used in multiple clients scenario.
- Attribute Datalen defines the length of optional attributes and will be used to identify the types of optional attributes.
```
typedef enum {
  CSIEBOX_PROTOCOL_MAGIC_REQ = 0x90,
  CSIEBOX_PROTOCOL_MAGIC_RES = 0x91,
} csiebox_protocol_magic;

typedef enum {
  CSIEBOX_PROTOCOL_OP_LOGIN = 0x00,
  CSIEBOX_PROTOCOL_OP_SYNC_META = 0x01,
  CSIEBOX_PROTOCOL_OP_SYNC_FILE = 0x02,
  CSIEBOX_PROTOCOL_OP_SYNC_HARDLINK = 0x03,
  CSIEBOX_PROTOCOL_OP_SYNC_END = 0x04,
  CSIEBOX_PROTOCOL_OP_RM = 0x05,
} csiebox_protocol_op;

typedef enum {
  CSIEBOX_PROTOCOL_STATUS_OK = 0x00,
  CSIEBOX_PROTOCOL_STATUS_FAIL = 0x01,
  CSIEBOX_PROTOCOL_STATUS_MORE = 0x02,
} csiebox_protocol_status;
```
### Actions
The client/server communication protocols consist of five communication sessions: `login`, `sync meta`, `sync file`, `hard link`, and `rm`. Each client/server communication session has its corresponding data structure to represent the transmitted data. The following illustrate the four communication sessions.
- You need to follow the rules of 5 actions and use defined protocol header, but you can modify others if you want. In other words, the only things that you can't modify in HW3 is the rules of 5 actions and protocol header in `csiebox_common.h`.

#### Login session:
```
typedef union {
  struct {
    csiebox_protocol_header header;
    struct {
      uint8_t user[USER_LEN_MAX];
      uint8_t passwd_hash[MD5_DIGEST_LENGTH];
    } body;
  } message;
  uint8_t bytes[sizeof(csiebox_protocol_header) + MD5_DIGEST_LENGTH * 2];
} csiebox_protocol_login;
```
Client sends user name and passwd_hash to server. If the user name and password matches, server returns OK in status field and client_id. Otherwise, FAIL is returned in status field.

#### Sync Meta session:
```
typedef union {
  struct {
    csiebox_protocol_header header;
    struct {
      uint32_t pathlen;
      struct stat stat;
      uint8_t hash[MD5_DIGEST_LENGTH];
    } body;
  } message;
  uint8_t bytes[sizeof(csiebox_protocol_header) +
                4 +
                sizeof(struct stat) +
                MD5_DIGEST_LENGTH];
} csiebox_protocol_meta;
```
When file/dir's permission or time have been changed, client will send sync_meta request include file pathlen and meta stat to server followed by file path. Server returns OK if it sync meta successful, otherwise, FAIL is returned in status field.

#### Sync File session:
```
typedef union {
  struct {
    csiebox_protocol_header header;
    struct {
      uint32_t pathlen;
      struct stat stat;
      uint8_t hash[MD5_DIGEST_LENGTH];
    } body;
  } message;
  uint8_t bytes[sizeof(csiebox_protocol_header) +
                4 +
                sizeof(struct stat) +
                MD5_DIGEST_LENGTH];
} csiebox_protocol_meta;
```
Sync file session consists of two steps in this operation. First, the client sends a sync_meta request which provides struct stat, file path length and file hash. Then, the client sends file path to server followed, and server can use path length in sync_meta header to retrieve the path from a character array. Server has to sync meta, and it checks the file hash, if file hash on client is different from that on server, the server returns MORE and the client will send file data to server. Otherwise, the server returns OK and terminates the session.
```
typedef union {
  struct {
    csiebox_protocol_header header;
    struct {
      uint64_t datalen;
    } body;
  } message;
  uint8_t bytes[sizeof(csiebox_protocol_header) + 8];
} csiebox_protocol_file;
```
When client receives MORE, client sends file header to server, followed by file data.

#### Hard Link session:
```
typedef union {
  struct {
    csiebox_protocol_header header;
    struct {
      uint32_t srclen;
      uint32_t targetlen;
    } body;
  } message;
  uint8_t bytes[sizeof(csiebox_protocol_header) + 8];
} csiebox_protocol_hardlink;
```
The client sends source path length and target path length to server, followed by source and target path. If a hard link is successfully created, server returns OK in status field. Otherwise, it returns FAIL in status field.
- Hint: you can use lstat() to check Inode to determine either it is hard link or not. Learn more about Inode or check out ch4.14 in Advanced programming in the Unix Environment.

#### Rm session:
```
typedef union {
  struct {
    csiebox_protocol_header header;
    struct {
      uint32_t pathlen;
    } body;
  } message;
  uint8_t bytes[sizeof(csiebox_protocol_header) + 4];
} csiebox_protocol_rm;
```
The client sends request path length to server, followed by path data. If the file/directory is successfully removed on the server, the server returns OK in status field. Otherwise, it returns FAIL in status field.
- Hint: because server can receive pathlen in protocol header, so you can use this pathlen to receive the following path by recv_message(conn_fd, buf, rm->message.body.pathlen);



## Submission Guide
- As previous MP, you should push your codes to your repository:
  `https://github.com/SystemProgrammingatNTU/SP16MP2-<YOUR_STUDENT_ID>`.
- Remember to write a `Makefile`, so that we can use `make` to generate an
  executable program `tree_walk`.

## Gradings
You can get points by correctly handling following requirement.
* Handle files correctly: 2 points.
* Handle large (more than several mega bytes) correctly: 1 points.
* Handle directory correctly: 2 points.
* Handle hidden files and directory correctly: 2 points.
* Handle symbolic link correctly: 1 points.

## How we may test your program
If you don't know how to set the time and memory limit, you can refer to
the following script.

```bash
#!/bin/bash

git clone YOUR_MP1_REPOSITORY YOUR_STUDENT_ID
cd YOUR_STUDENT_ID
make
rm -rf server
rm -rf client
mkdir server
mkdir client
echo sometext > client/smallfile
mkdir client/subdirectory
timeout 30s ./tree_walk
```

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>
<script type="text/javascript" async
  src="//cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
