---
title: "graceful restart と stop"
---

graceful restart と stop をしたいですか？
いくつかの方法があります。

[fvbock/endless](https://github.com/fvbock/endless) を使って、デフォルトの `ListenAndServe` を置き換えることができます。詳細は Issue [#296](https://github.com/gin-gonic/gin/issues/296) を参照ください。

```go
router := gin.Default()
router.GET("/", handler)
// [...]
endless.ListenAndServe(":4242", router)
```

endless の代わりは以下があります。

* [manners](https://github.com/braintree/manners): A polite Go HTTP server that shuts down gracefully.
* [graceful](https://github.com/tylerb/graceful): Graceful is a Go package enabling graceful shutdown of an http.Handler server.
* [grace](https://github.com/facebookgo/grace): Graceful restart & zero downtime deploy for Go servers.

もし Go 1.8 を使っているなら、これらのライブラリを使う必要はないかもしれません！http.Server 組み込みの [Shutdown()](https://golang.org/pkg/net/http/#Server.Shutdown) メソッドを、graceful shutdowns に利用することを検討してみてください。詳細は Gin の [graceful-shutdown](https://github.com/gin-gonic/examples/tree/master/graceful-shutdown) サンプルコードを見てみてください。

```go
//go:build go1.8
// +build go1.8

package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router.Handler(),
	}

	go func() {
		// service connections
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()

	// Wait for interrupt signal to gracefully shutdown the server with
	// a timeout of 5 seconds.
	quit := make(chan os.Signal, 1)
	// kill (no params) by default sends syscall.SIGTERM
	// kill -2 is syscall.SIGINT
	// kill -9 is syscall.SIGKILL but can't be caught, so don't need add it
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("Shutdown Server ...")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Println("Server Shutdown:", err)
	}
	log.Println("Server exiting")
}
```
