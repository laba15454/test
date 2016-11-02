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


## Pack Provided by TA

You can find it [here]()

In this MP, we provide you basic code for client and server that take cares of building connection between them. So you only need to focus on how to synchronize client and server through network communication. We also provide a program called `port_register` that help to connect client to server, and some config files that you need to modify to adapt our program to your device.
There's also a simple example of `inotify`. You will use `inotify` to monitor filesystem events on client side so that you can push changes of files and directories on client side to server at runtime.

```
SP16MP3
├── bin( this is where compiled binaries go to, including csiebox_client, csiebox_server and port_register)
├── cdir( this is the directory containing client's files and directories)
├── config( Here you can find config files, including server.cfg, client.cfg and account)
├── include( Here you can find some header files)
├── sdir( this is the directory of server)
└── src( Here is where you put your source code and Makefile)
```


## Communication
In this homework, you need to wite two programs, `csiebox_client` and `csiebox_server`, and they need to communicate with each other through Internet. To achieve this, network socket is needed, which is beyond the scope of this class. But don’t worry, we have built all you need for communication, all you have to do is follow our instructions and protocols. If you are interested in socket, information can be found in Resource section.

### How to Connect Client to Server
First thing to do in this MP is to connect your client application to server and login. We have done the code of this part for you, you only need to modify the configuration files so that it can work on your machine. There are three configuration files you need to modify: server.cfg, client.cfg and account. 


Server.cfg contains information of server, and it has two fields, path and account_path. Path is the path of server directory where we will store files and directories from our users, and account_path is the path of the configuration file ‘account’. 


Client.cfg contains information of client, and it has five fields. Name is the user name of your local device, for example, it will be your student ID if you run it on workstation. Server is the ip address of the machine that runs your server application. It will be ‘localhost’ if both client and server are run on the same machine. Else you can use the command ‘ifconfig’ to get the ip address. User and passwd is the user name and password of the user that run the client. You can register new user by adding pair of ‘username,passwd’ to the configuration file ‘account’. Path is the path of client directory, where stores files and directories we are going to synchronize.


Account contains information of registered user on the server. Pairs of ‘username, passwd’ is stored in this file. You can register new user by adding pair of ‘username,passwd’ to this file. Each user will have its own directory on server, which will be ‘/path/to/server/dir/username’ once it uploads some files to server.


Once you modified those configuration files, you can try to connect client to server. Compile the code provided by TA, and four binary will be generated: csiebox_server, csiebox_client, port_register, and inotify_test. csiebox_server and csiebox_client is binary for server and client respectively. Port_register is used to register port number of server so that client can connect to it through socket so that you have to run port_register before csiebox_server and csiebox_client. TA will run port_register on workstation linux1~15 for you so that you don’t have to run it yourself. 
Once port_register is up, you can run csiebox_server and csiebox_client to see if they can communicate to each other. Csiebox_client will try to longin to the server with account information provided in client.cfg and csiebox_server will verify those information. You can check output of these binary to see if the client login successfully.

### Interface and Protocol
To send message to or receive message from other program, you can use `send_message()` and `recv_message()` provided by TA. They function just like `write()` and` read()`, but instead of local file, they work on socket. You can fin example of using them in the code provided by TA.


To make communication between client and server simple and efficient, we provide some data structures that define format of message and protocol that defines some actions that client and server can do. You can find the data structures in `csiebox_common.h`. You can find some examples in the code we provided. 

```c
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
Struct csiebox_protocol_header is a base header for communication protocol between client and server, defined in `csiebox_common.h`.
It consists of five attributes.
- Attribute Magic, defined for security purpose
- Attribute Op defines operation code.
- Attribute Status defines return status.
- Attribute Client_id defines the identification of client processes, which will be used in multiple clients scenario.
- Attribute Datalen defines the length of optional attributes and will be used to identify the types of optional attributes.
```c
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
```c
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
```c
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
```c
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
```c
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
```c
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
- Hint: you can use lstat() to check Inode to determine either it is hard link or not. Learn more about [Inode](https://en.wikipedia.org/wiki/Inode) or check out ch4.14 in Advanced programming in the Unix Environment.

#### Rm session:
```c
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

## Synchronization
In this MP, there are three different synchronization scenario: client uploads existing local files or directories to server, client downloads files or directories that was previously uploaded from server, and push changes of local files or directories to server at runtime.
The first scenario is like MP2, where you need to traverse client directory and copy its content to server directory. Yet this time client and server are two seperate programs, and possibly run on different machines. The rule for handling different type of file is the same as MP2, for example, you have to copy soft link itself instead of the file it point to. You need to make sure that files and directories in client side and server side have identical content and status. You can put your client-side code like directory traversal in `csiebox_client_run()` in `csiebox_client.c` and server-side code like creation of copied files and directories in `handle_request()` in `csiebox_server.c`.
The second scenario like the reverse process of previous scenario. A user, after uploading his or her files and directories in one client device to server, he or she might go to another device and login to CSIEBOX. This time, server has users files and directories and the client is empty, so the client needs to download these content to its local directory. Note that the code TAs provide doesn’t support this kind of synchronization, so you have to do it yourself. Modify `csiebox_client_run()` in `csiebox_client.c`, and `csiebox_server_run()` and `handle_request()` in `csiebox_server.c` to implement this feature.
The third scenario is when the first or second one is done, and content in client and server is identical. At this stage, the client process needs to monitor its local directory to catch event like creation or modification of files, and push these changes to server. You can write code of this part in `csiebox_client_run()` in `csiebox_client.c`.

### Monitor Change of Directories and File usig Inotify

In this MP, client program needs to continusly monitoring changes of file and directory in local repository, and synchronize those changes with server. Change includes creation and removement of file and directory, and modification of file content and status( you are guaranteed that there will be *no* renaming ). To do this, you need to use [inotify API](http://man7.org/linux/man-pages/man7/inotify.7.html). The inotify API provides a mechanism for monitoring filesystem events. Following is brief introduction of how to use inotify. More information can be found in Resource section. You can also find example of using inotify in `inotify_test.c`.


To start monitoring, you need to first create an inotify instance with [inotify_init(2)](http://man7.org/linux/man-pages/man2/inotify_init.2.html). It will return a file descriptor representing the inotify instance. Then, you add the file or directory you want to monitor to the instance with [inotify_add_watch(2)](http://man7.org/linux/man-pages/man2/inotify_add_watch.2.html) . It will add the file or directory into the ‘watch list’ of the instance and return a watch descriptor that can later be used to identify which file or directory has trigger an inotify event. Note that when a directory is monitored, inotify will return events for the directory itself, and for files inside the directory. You can remove a file or descriptor form watch list by calling [inotify_rm_watch(2)](http://man7.org/linux/man-pages/man2/inotify_rm_watch.2.html) .


Once the files or directories have been monitored, you can call `read(2)` on the inotify descriptor to read events that happened on those items. Event is represented as following data structure:
```c
          struct inotify_event {
               int      wd;       /* Watch descriptor */
               uint32_t mask;     /* Mask describing event */
               uint32_t cookie;   /* Unique cookie associating related
                                     events (for rename(2)) */
               uint32_t len;      /* Size of name field */
               char     name[];   /* Optional null-terminated name */
           };
```
You can find example of how to read events from inotify instance and how to handle those events in `inotify_test.c`.


Note that information in inotify event doesn’t include the pathname of the item triggering the event, only the watch discritpor. Thus you have to keep the mapping between watch descriptor and pathname by yourself. We provide a simple hash table data structure that can help you do this, which can be found in `hash.h` and `hash.c`. By inotify API, you can use `wd = inotify_add_watch(inotify_fd, "/PathOfDir_YouWantToMonitor", IN_CREATE | IN_DELETE | IN_ATTRIB | IN_MODIFY)` to add a monitor on a directory , so wd is the inotify id and /PathOfDir_YouWantToMonitor is the directory path. You can use `put_into_hash(&(client->inotify_hash), (void*)inotify_path, wd)` to store this relation and use `get_from_hash(&(client->inotify_hash), (void**)&wd_path, event->wd)` to retrive the monitored directory path by event->wd.


Once the client program catchs events, it needs to synchronize server according to those event.

## I/O Multiplexing
In this MP, server program needs to handle requests from multiple clients using I/O multiplexing. However, the code provided by TA doesn’t support this, so you have to modify the code. You need to modify `csiebox_server_run()` and `handle_request()` in `csiebox_server.c`, and use [select()](http://man7.org/linux/man-pages/man2/select.2.html) to handle concurrent client requests without blocking the server process. You are allowed to use [poll()](http://man7.org/linux/man-pages/man2/poll.2.html) or [epoll](http://man7.org/linux/man-pages/man7/epoll.7.html), which are enhanced versions of `select()`, to support I/O multiplexing.

## Assignment
In this MP, you need to accomplish following assignments:

1. With one client and one server, the client directory contains some files and directories while the server directory is empty. You need to synchronize client and server, that is, upload existing items in client to server. And after that, client needs to continusly monitor its local directory and push changes to server.
2. Following assignment 1, the user logout and then login again in another client device with empty client directory. You need to synchronize client and server, that is, download existing items in server to client. And after that, client needs to continusly monitor its local directory and push changes to server.
3. Do assignment 1, but this time, there will be many clients( at most 5) of different accounts.
4. This time, there will be at most 10 clients, all of different accounts. All users starts with empty server directory. Some of them will logout and login from another client device with empty client directory. And all of these clients need to continusly monitor their local directory and push changes to server. It is like the combination of assignment1 and assignment2 in multi-clients scenario. 

Note:
- There will be directory, files, symbolic link and hard link in client directory.
- During client monitoring, there will be no creation of hard link.
- The size of regular file is at most 100 MB.

## Gradings

2 points for each assignment

**!!IMPORTANT!!**
Note that MP4 is based on MP3, so please *DO ACCOMPLISH THIS MP*, else you will have trouble doing MP4. If you have problem doing this MP, feel free to issue it on github, or turn to TAs directly.

## Explanation of Important Files
+ csiebox_client.c: this's the process to simulate client, you need to implement your code about client action in this file.
+ csiebox_server.c: this's the process to simulate server, you need to implement your code about server action in this file.
+ hash.c: provide a build data structure, which can help you to manage the relation of inotify id and monitored dir path. By inotify API, you can use `wd = inotify_add_watch(inotify_fd, "/PathOfDir_YouWantToMonitor", IN_CREATE | IN_DELETE | IN_ATTRIB | IN_MODIFY)` to add a monitor on a directory , so wd is the inotify id and /PathOfDir_YouWantToMonitor is the directory path. You can use `put_into_hash(&(client->inotify_hash), (void*)inotify_path, wd)` to store this relation and use `get_from_hash(&(client->inotify_hash), (void**)&wd_path, event->wd)` to retrive the monitored directory path by event->wd.
 - Note: You can also use another way to record the relation between inotify id and directory path, for example, you can use char array to maintain this.
+ connect.c, port_register.c, csiebox_common.c: connect between client process and server process.
+ inotify_test.c: a test program about inotify, you need to learn how to use it, and write an inotify system in csiebox_client.c.
+ Server.cfg: it contains information of server, and it has two fields, path and account_path. Path is the path of server directory where we will store files and directories from our users, and account_path is the path of the configuration file ‘account’. 
+ Client.cfg: it contains information of client, and it has five fields. Name is the user name of your local device, for example, it will be your student ID if you run it on workstation. Server is the ip address of the machine that runs your server application. It will be ‘localhost’ if both client and server are run on the same machine. Else you can use the command ‘ifconfig’ to get the ip address. User and passwd is the user name and password of the user that run the client. You can register new user by adding pair of ‘username,passwd’ to the configuration file ‘account’. Path is the path of client directory, where stores files and directories we are going to synchronize.
+ Account: it contains information of registered user on the server. Pairs of ‘username, passwd’ is stored in this file. You can register new user by adding pair of ‘username,passwd’ to this file. Each user will have its own directory on server, which will be ‘/path/to/server/dir/username’ once it uploads some files to server.


## Submission Guide
- As previous MP, you should push your codes to your repository:
  `https://github.com/SystemProgrammingatNTU/SP16MP3-<YOUR_STUDENT_ID>`.
- Remember to maintain `Makefile` in source directory, so that we can use `make` to generate
  executable programs `csiebox_client` and `csiebox_server`.
  
## Resource
[fts(3) - Linux man page](http://man7.org/linux/man-pages/man3/fts.3.html)(it’s an API for traversing file hierarchies, in case you didn’t finish your MP2)
[Introduction of socket](http://wmnlab.ee.ntu.edu.tw/nmlab/exp1_socket.html)
[inotify(7) - Linux man page](http://man7.org/linux/man-pages/man7/inotify.7.html)
[select(3) - Linux man page](https://linux.die.net/man/3/select)
[poll(3) - Linux man page](https://linux.die.net/man/3/poll)
[epoll(4) - Linux man page](https://linux.die.net/man/4/epoll)
[select() - Beej's Guide to Network Programming](http://beej.us/guide/bgnet/output/html/multipage/selectman.html)
[select() - Beej's Guide to Network Programming 正體中文版](http://beej-zhtw.netdpi.net/07-advanced-technology/7-2-select)

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>
<script type="text/javascript" async
  src="//cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
