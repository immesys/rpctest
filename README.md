# Goroutine leak reproducer

This is a simple program adapted from https://github.com/zombiezen/go-capnproto2/blob/master/rpc/example_test.go
to demonstrate problems I see with a larger program.

## What I see

Even after the connection is closed, there are goroutines still running. Specifically, one dispatch() goroutine
for every ServerToClient that is called. As this goroutine references any objects passed to the X_ServerToClient
function, they pin a substantial amount of memory.

If you run the program, the output you see is

```
Results: 7 30
2016/08/07 00:28:33 rpc: aborted by remote: rpc: shutdown
panic: look at the running goroutines. I would expect none, but dispatch() and manager still exist

goroutine 1 [running]:
panic(0x638000, 0xc8200f2190)
	/usr/local/go/src/runtime/panic.go:481 +0x3e6
main.main()
	/home/michael/go/src/github.com/immesys/rpctest/main.go:69 +0xfae

goroutine 17 [syscall, locked to thread]:
runtime.goexit()
	/usr/local/go/src/runtime/asm_amd64.s:1998 +0x1

goroutine 5 [select]:
zombiezen.com/go/capnproto2/server.(*server).dispatch(0xc8200104c0)
	/home/michael/go/src/zombiezen.com/go/capnproto2/server/server.go:64 +0x202
created by zombiezen.com/go/capnproto2/server.New
	/home/michael/go/src/zombiezen.com/go/capnproto2/server/server.go:56 +0x22f

goroutine 8 [semacquire]:
sync.runtime_Semacquire(0xc8200aa034)
	/usr/local/go/src/runtime/sema.go:47 +0x26
sync.(*WaitGroup).Wait(0xc8200aa028)
	/usr/local/go/src/sync/waitgroup.go:127 +0xb4
zombiezen.com/go/capnproto2/rpc.(*manager).shutdown(0xc8200aa020, 0x7f254b1c2028, 0xc82000a190, 0x0)
	/home/michael/go/src/zombiezen.com/go/capnproto2/rpc/manager.go:68 +0xba
zombiezen.com/go/capnproto2/rpc.dispatchRecv(0xc8200aa020, 0x7f254b1c6498, 0xc8200a4000, 0xc820074240)
	/home/michael/go/src/zombiezen.com/go/capnproto2/rpc/transport.go:134 +0x2c1
zombiezen.com/go/capnproto2/rpc.NewConn.func1()
	/home/michael/go/src/zombiezen.com/go/capnproto2/rpc/rpc.go:102 +0x54
zombiezen.com/go/capnproto2/rpc.(*manager).do.func1(0xc8200aa020, 0xc8200126c0)
	/home/michael/go/src/zombiezen.com/go/capnproto2/rpc/manager.go:50 +0x52
created by zombiezen.com/go/capnproto2/rpc.(*manager).do
	/home/michael/go/src/zombiezen.com/go/capnproto2/rpc/manager.go:51 +0xb4
exit status 2
```

## What I expect to see

I would think these goroutines should die at some point, but it seems that Close() is never called.

Thanks!
