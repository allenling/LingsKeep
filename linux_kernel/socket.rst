socket
===========

linux内核中的socket源码

内核版本是v4.15




socket结构
=============

socket是BSD socket结构, 包含了file, sock等属性


https://elixir.bootlin.com/linux/v4.15/source/include/linux/net.h#L110

.. code-block:: c

    /**
     *  struct socket - general BSD socket
     *  @state: socket state (%SS_CONNECTED, etc)
     *  @type: socket type (%SOCK_STREAM, etc)
     *  @flags: socket flags (%SOCK_NOSPACE, etc)
     *  @ops: protocol specific socket operations
     *  @file: File back pointer for gc
     *  @sk: internal networking protocol agnostic socket representation
     *  @wq: wait queue for several uses
     */
    struct socket {
    	socket_state		state;
    
    	short			type;
    
    	unsigned long		flags;
    
    	struct socket_wq __rcu	*wq;
    
    	struct file		*file;
    	struct sock		*sk;
    	const struct proto_ops	*ops;
    };


这里只是一些声明, 具体的实现是在https://elixir.bootlin.com/linux/v4.15/source/net/socket.c中, 并且每一种类型的socket, ops也不会一样, 比如tcp和udp

1. socket也有自己的file

2. socket是应用层的表示, 而网络层的表示是sock结构

socket.ops
==============

socket.ops是一个proto_ops的结构体, 定义了socket的操作, 下面也只是声明而已, 具体实现由不同的socket来自己实现, 比如tcp和udp有不同的实现

https://elixir.bootlin.com/linux/v4.15/source/include/linux/net.h#L133


.. code-block:: c

    // 包含了poll, connect等操作
    struct proto_ops {
    	int		family;
    	struct module	*owner;
    	int		(*release)   (struct socket *sock);
    	int		(*bind)	     (struct socket *sock,
    				      struct sockaddr *myaddr,
    				      int sockaddr_len);
    	int		(*connect)   (struct socket *sock,
    				      struct sockaddr *vaddr,
    				      int sockaddr_len, int flags);
    	int		(*socketpair)(struct socket *sock1,
    				      struct socket *sock2);
    	int		(*accept)    (struct socket *sock,
           // 省略了很多代码
    }

sock结构
============

sock是一个网络层的表示, 定义很长了

https://elixir.bootlin.com/linux/v4.15/source/include/net/sock.h#L318

.. code-block:: c

    // 注释也很长
    /**
      *	struct sock - network layer representation of sockets
      * 省略了其他注释
     */
    struct sock {
    
    // 很多定义, 先忽略吧
    
    }


socket系统调用
=================

https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L1317

.. code-block:: c

    SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
    {
    	int retval;
    	struct socket *sock;
    	int flags;
    
    	/* Check the SOCK_* constants for consistency.  */
    	BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
    	BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
    	BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
    	BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);
    
    	flags = type & ~SOCK_TYPE_MASK;
    	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
    		return -EINVAL;
    	type &= SOCK_TYPE_MASK;
    
    	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
    		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;
    
        // 分配socket结构的内存
    	retval = sock_create(family, type, protocol, &sock);
    	if (retval < 0)
    		return retval;
    
        // 创建file已经其他复杂工作
    	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
    }
   
sock_create和sock_map_fd是主要调用

sock_create
================

调用socket来创建一个sockt, 最终会调用__sock_create这个函数

https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L1196

.. code-block:: c

    int __sock_create(struct net *net, int family, int type, int protocol,
    			 struct socket **res, int kern)
    {
    	int err;
        // 声明一个socket结构
    	struct socket *sock;
        // 这个pf就是每一种类型的socket需要的初始化定义
    	const struct net_proto_family *pf;
    
    	/*
    	 *      Check protocol is in range
    	 */
         // 判断协议簇
    	if (family < 0 || family >= NPROTO)
    		return -EAFNOSUPPORT;
    	if (type < 0 || type >= SOCK_MAX)
    		return -EINVAL;
    
    	/* Compatibility.
    
    	   This uglymoron is moved from INET layer to here to avoid
    	   deadlock in module load.
    	 */
    	if (family == PF_INET && type == SOCK_PACKET) {
    		pr_info_once("%s uses obsolete (PF_INET,SOCK_PACKET)\n",
    			     current->comm);
    		family = PF_PACKET;
    	}
    
    	err = security_socket_create(family, type, protocol, kern);
    	if (err)
    		return err;
    
    	/*
    	 *	Allocate the socket and allow the family to set things up. if
    	 *	the protocol is 0, the family is instructed to select an appropriate
    	 *	default.
    	 */
         // 这里就是实际分配结构体内存的地方
    	sock = sock_alloc();
    	if (!sock) {
    		net_warn_ratelimited("socket: no more sockets\n");
    		return -ENFILE;	/* Not exactly a match, but its the
    				   closest posix thing */
    	}
    
    	sock->type = type;
        // 省略很多代码

        // 拿到family对应的socket配置
        pf = rcu_dereference(net_families[family]);

        // 省略代码

        // 初始化具体的socket类型
        err = pf->create(net, sock, protocol, kern);

        // 省略代码

    
    }

1. sock_alloc作用是分配socket对应的内存结构, 主要是inode

2. pf是socket的配置表

3. pf = rcu_dereference(net_families[family])这一句是: 通过family去查表找到对应的socket类型的配置, 调用create去创建具体类型的socket, 其实就是覆盖一些通过配置

net_families
==============

这是一组配置表, 是一个数组, 没一项都是net_proto_family结构

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L163
    static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;


    // https://elixir.bootlin.com/linux/v4.15/source/include/linux/net.h#L205
    struct net_proto_family {
    	int		family;
    	int		(*create)(struct net *net, struct socket *sock,
    				  int protocol, int kern);
    	struct module	*owner;
    };


inet类型的socket的net_proto_family定义是在

https://elixir.bootlin.com/linux/v4.15/source/net/ipv4/af_inet.c#L1018

.. code-block:: c

    static const struct net_proto_family inet_family_ops = {
    	.family = PF_INET,
    	.create = inet_create,
    	.owner	= THIS_MODULE,
    };


也就是说, 一个inet类型的socket, 会调用inet_create去更新socket中的属性

那inet的配置什么时候设置到net_families这个数组中的呢, 是在init函数中调用socket_register去注册的


.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/net/ipv4/af_inet.c#L1819
    static int __init inet_init(void)
    {
    	struct inet_protosw *q;
    	struct list_head *r;
    	int rc = -EINVAL;
    
    	sock_skb_cb_check_size(sizeof(struct inet_skb_parm));
    
            // 下面的proto_register就是注册上tcp/udp等协议
    	rc = proto_register(&tcp_prot, 1);
    	if (rc)
    		goto out;
    
    	rc = proto_register(&udp_prot, 1);
    	if (rc)
    		goto out_unregister_tcp_proto;
    
    	rc = proto_register(&raw_prot, 1);
    	if (rc)
    		goto out_unregister_udp_proto;
    
    	rc = proto_register(&ping_prot, 1);
    	if (rc)
    		goto out_unregister_raw_proto;
    
    	/*
    	 *	Tell SOCKET that we are alive...
    	 */
            // 把inet_family_ops注册到net_families这个数组中
    	(void)sock_register(&inet_family_ops);
    
            // 省略代码
    
    }

    // https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L2521
    int sock_register(const struct net_proto_family *ops)
    {
    	int err;
    
    	if (ops->family >= NPROTO) {
    		pr_crit("protocol %d >= NPROTO(%d)\n", ops->family, NPROTO);
    		return -ENOBUFS;
    	}
    
    	spin_lock(&net_family_lock);
    	if (rcu_dereference_protected(net_families[ops->family],
    				      lockdep_is_held(&net_family_lock)))
    		err = -EEXIST;
    	else {
                // 设置net_families数组
    		rcu_assign_pointer(net_families[ops->family], ops);
    		err = 0;
    	}
    	spin_unlock(&net_family_lock);
    
    	pr_info("NET: Registered protocol family %d\n", ops->family);
    	return err;
    }
    EXPORT_SYMBOL(sock_register);

inet家族的ops定义
====================

inet中的配置:

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/net/ipv4/af_inet.c#L1027
    static struct inet_protosw inetsw_array[] =
    {
    	{
    		.type =       SOCK_STREAM,
    		.protocol =   IPPROTO_TCP,
    		.prot =       &tcp_prot,
    		.ops =        &inet_stream_ops,
    		.flags =      INET_PROTOSW_PERMANENT |
    			      INET_PROTOSW_ICSK,
    	},
    
    	{
    		.type =       SOCK_DGRAM,
    		.protocol =   IPPROTO_UDP,
    		.prot =       &udp_prot,
    		.ops =        &inet_dgram_ops,
    		.flags =      INET_PROTOSW_PERMANENT,
           },
    
           {
    		.type =       SOCK_DGRAM,
    		.protocol =   IPPROTO_ICMP,
    		.prot =       &ping_prot,
    		.ops =        &inet_sockraw_ops,
    		.flags =      INET_PROTOSW_REUSE,
           },
    
           {
    	       .type =       SOCK_RAW,
    	       .protocol =   IPPROTO_IP,	/* wild card */
    	       .prot =       &raw_prot,
    	       .ops =        &inet_sockraw_ops,
    	       .flags =      INET_PROTOSW_REUSE,
           }
    };


所以tcp类型socket中的ops将会被替换成inet_stream_ops

.. code-block:: c

    // https://elixir.bootlin.com/linux/v4.15/source/net/ipv4/af_inet.c#L928
    const struct proto_ops inet_stream_ops = {
    	.family		   = PF_INET,
    	.owner		   = THIS_MODULE,
    	.release	   = inet_release,
    	.bind		   = inet_bind,
    	.connect	   = inet_stream_connect,
    	.socketpair	   = sock_no_socketpair,
    	.accept		   = inet_accept,
    	.getname	   = inet_getname,
    	.poll		   = tcp_poll,
    	.ioctl		   = inet_ioctl,
    	.listen		   = inet_listen,
    	.shutdown	   = inet_shutdown,
    	.setsockopt	   = sock_common_setsockopt,
    	.getsockopt	   = sock_common_getsockopt,
    	.sendmsg	   = inet_sendmsg,
    	.recvmsg	   = inet_recvmsg,
    	.mmap		   = sock_no_mmap,
    	.sendpage	   = inet_sendpage,
    	.splice_read	   = tcp_splice_read,
    	.read_sock	   = tcp_read_sock,
    	.sendmsg_locked    = tcp_sendmsg_locked,
    	.sendpage_locked   = tcp_sendpage_locked,
    	.peek_len	   = tcp_peek_len,
    #ifdef CONFIG_COMPAT
    	.compat_setsockopt = compat_sock_common_setsockopt,
    	.compat_getsockopt = compat_sock_common_getsockopt,
    	.compat_ioctl	   = inet_compat_ioctl,
    #endif
    };
    EXPORT_SYMBOL(inet_stream_ops);

所以, tcp的poll就是tcp_poll这个函数了


sock_alloc
==============

分配inode和内存给socket结构体

.. code-block:: c

    struct socket *sock_alloc(void)
    {
    	struct inode *inode;
    	struct socket *sock;
    
    	inode = new_inode_pseudo(sock_mnt->mnt_sb);
    	if (!inode)
    		return NULL;
    
    	sock = SOCKET_I(inode);
    
    	inode->i_ino = get_next_ino();
    	inode->i_mode = S_IFSOCK | S_IRWXUGO;
    	inode->i_uid = current_fsuid();
    	inode->i_gid = current_fsgid();
    	inode->i_op = &sockfs_inode_ops;
    
    	this_cpu_add(sockets_in_use, 1);
    	return sock;
    }


sock_map_fd
===============

在socket的系统调用中, 最后return的是sock_map_fd的返回值

sock_map_fd这个函数是分配具体的fd结构, 以及赋值具体的file类型, 具体函数实现到socket结构体

https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L435

.. code-block:: c

    static int sock_map_fd(struct socket *sock, int flags)
    {
    	struct file *newfile;
    	int fd = get_unused_fd_flags(flags);
    	if (unlikely(fd < 0)) {
    		sock_release(sock);
    		return fd;
    	}
    
        // 这里!!!!
    	newfile = sock_alloc_file(sock, flags, NULL);
    	if (likely(!IS_ERR(newfile))) {
    		fd_install(fd, newfile);
    		return fd;
    	}
    
    	put_unused_fd(fd);
    	return PTR_ERR(newfile);
    }


sock_alloc_file
=====================

分配具体file, 和file的operations

https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L395

.. code-block:: c

    struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
    {
    	struct qstr name = { .name = "" };
    	struct path path;
    	struct file *file;
    
    	if (dname) {
    		name.name = dname;
    		name.len = strlen(name.name);
    	} else if (sock->sk) {
    		name.name = sock->sk->sk_prot_creator->name;
    		name.len = strlen(name.name);
    	}
    	path.dentry = d_alloc_pseudo(sock_mnt->mnt_sb, &name);
    	if (unlikely(!path.dentry)) {
    		sock_release(sock);
    		return ERR_PTR(-ENOMEM);
    	}
    	path.mnt = mntget(sock_mnt);
    
    	d_instantiate(path.dentry, SOCK_INODE(sock));
    
        // file结构
        // 注意下socket_file_ops
    	file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
    		  &socket_file_ops);
    	if (IS_ERR(file)) {
    		/* drop dentry, keep inode for a bit */
    		ihold(d_inode(path.dentry));
    		path_put(&path);
    		/* ... and now kill it properly */
    		sock_release(sock);
    		return file;
    	}
    
    	sock->file = file;
    	file->f_flags = O_RDWR | (flags & O_NONBLOCK);
        // 这里的private_data就是file对应的具体结构
        // epoll也是这样
    	file->private_data = sock;
    	return file;
    }

注意到生成file结构的时候, 使用socket_file_ops覆盖了底层的file结构中的f_ops属性, 也就是说, socket_file_ops这个包含了socket特定的具体的file操作实现

socket_file_ops
====================

这里是socket的file的具体实现

https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L140

.. code-block:: c

    static const struct file_operations socket_file_ops = {
    	.owner =	THIS_MODULE,
    	.llseek =	no_llseek,
    	.read_iter =	sock_read_iter,
    	.write_iter =	sock_write_iter,
    	.poll =		sock_poll,
    	.unlocked_ioctl = sock_ioctl,
    #ifdef CONFIG_COMPAT
    	.compat_ioctl = compat_sock_ioctl,
    #endif
    	.mmap =		sock_mmap,
    	.release =	sock_close,
    	.fasync =	sock_fasync,
    	.sendpage =	sock_sendpage,
    	.splice_write = generic_splice_sendpage,
    	.splice_read =	sock_splice_read,
    };


注意到, poll操作是sock_poll, 也就是说, 在epoll中调用fd对应的file对应的poll实现的时候, 如果fd是一个socket, 那么就会调用到sock_poll


sock_poll
==========

socket的poll实现

https://elixir.bootlin.com/linux/v4.15/source/net/socket.c#L1100

.. code-block:: c

    static unsigned int sock_poll(struct file *file, poll_table *wait)
    {
    	unsigned int busy_flag = 0;
    	struct socket *sock;
    
    	/*
    	 *      We can't return errors to poll, so it's either yes or no.
    	 */
        // 这里file的private_data就是socket结构了
    	sock = file->private_data;
    
    	if (sk_can_busy_loop(sock->sk)) {
    		/* this socket can poll_ll so tell the system call */
    		busy_flag = POLL_BUSY_LOOP;
    
    		/* once, only if requested by syscall */
    		if (wait && (wait->_key & POLL_BUSY_LOOP))
    			sk_busy_loop(sock->sk, 1);
    	}
    
        // 注意这里, 调用的是socket结构中的ops->poll函数
    	return busy_flag | sock->ops->poll(file, sock, wait);
    }


所以, 最终poll的实现是在socket结构中的ops的poll函数


 

