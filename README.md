# Golang Websocket 搭配 protobuf

下載並安裝

```shell
go get -u -v github.com/gorilla/websocket
go get -u -v github.com/golang/protobuf/proto
go get -u -v github.com/golang/protobuf/protoc-gen-go
```

![image](./images/20200831150849.png)

下載proto編譯工具
- 參考這篇
https://github.com/IvesShe/Golang_Protobuf

# 將proto轉化成Golang程式

```bash
protoc --go_out=. *.proto
```

# 執行結果

## 客戶端

發送ping，並在接收到pong時打印"心跳"


## 服務器

服務器接收到ping時，會回傳pong

# 執行畫面

![image](./images/20201129172049.png)

# Server

server.go
```go
package main

import (
	"bytes"
	"encoding/binary"
	"flag"
	"fmt"
	"html/template"
	"log"
	"net/http"

	"github.com/golang/protobuf/proto"
	"github.com/gorilla/websocket"
	"github.com/wonderivan/logger"

	myMsg "../proto"
)

// 定義flag參數，這邊會返回一個相應的指針
var addr = flag.String("addr", "localhost:1351", "http service address")

var upgrader = websocket.Upgrader{
	// 解決跨域問題
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

func echo(w http.ResponseWriter, r *http.Request) {
	// 開啟websocket服務
	ws, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		logger.Debug("upgrade:", err)
		return
	}
	logger.Debug("服務器啟動，監聽端口1351中......")
	buf := make([]byte, 100)

	// 預先關閉，此行在離開echo時會執行
	defer ws.Close()

	// 一直待命收資料
	for {
		// 讀取資料
		mt, message, err := ws.ReadMessage()
		if err != nil {
			logger.Debug("read:", err)
			break
		}

		logger.Debug("recv: %v", message)

		//var pkgId uint16
		//pkgId = uint16(message[0:2])
		//binary.BigEndian.PutUint16(buf[0:2], message[0:2])
		var rpkgId = binary.BigEndian.Uint16(message[0:2])

		// 如果是ping
		if int16(rpkgId) == int16(myMsg.Command_Ping) {
			// 就回pong
			mes := "pong"
			logger.Debug("心跳")

			// 傳輸的資料前面，要加上2bytes的id碼，id碼參考proto檔的command enum
			// 處理消息ID
			var pkgId uint16
			pkgId = uint16(myMsg.Command_Pong)
			binary.BigEndian.PutUint16(buf[0:2], pkgId)
			data := myMsg.StoCHeartBeat{
				//Balance: 123456,
				//Code:    200,
			}

			// 將資料編碼成 Protocol Buffer 格式（請注意是傳入 Pointer）。
			dataBuffer, _ := proto.Marshal(&data)

			// 將消息ID與DATA整合，一起送出
			pkgData := [][]byte{buf[:2], dataBuffer}
			pkgDatas := bytes.Join(pkgData, []byte{})
			err = ws.WriteMessage(mt, pkgDatas)
			logger.Debug("write:", string(mes))
		} else {
			// 將讀到的資料送出
			err = ws.WriteMessage(mt, message)
			logger.Debug("write: %v", message)
		}
		// 將讀到的資料送出
		if err != nil {
			logger.Debug("write:", err)
			break
		}
	}
}

// 這邊會產生一個實體網頁
func home(w http.ResponseWriter, r *http.Request) {
	homeTemplate.Execute(w, "ws://"+r.Host+"/echo")
}

func main() {
	logger.Debug("程序啟動")

	// 建立一個 User 格式，並在其中放入資料。
	data := myMsg.StoCLogin{
		Balance: 168,
	}

	// user := myMsg.User{
	// 	UserId:  888,
	// 	UserPwd: "123456",
	// }
	// data02 := myMsg.UserInfo{
	// 	UserId:  888,
	// 	UserPwd: "123456",
	// }
	//fmt.Println(user)

	// 將資料編碼成 Protocol Buffer 格式（請注意是傳入 Pointer）。
	dataBuffer, _ := proto.Marshal(&data)

	// 將已經編碼的資料解碼成 protobuf.User 格式。
	var login myMsg.StoCLogin
	proto.Unmarshal(dataBuffer, &login)

	// 輸出解碼結果。
	fmt.Println(login.Balance, " ")

	// 調用flag.Parse()解析命令行參數到定義的flag
	//flag.Parse()

	// SetFlags(flag int)可以用來自定義log的輸出格式
	//log.SetFlags(0)

	// 處理/echo的路由
	http.HandleFunc("/echo", echo)

	// 處理/的路由
	http.HandleFunc("/", home)

	// 啟動服務，addr是指針，所以加*取值，若有錯會回傳錯誤log
	log.Fatal(http.ListenAndServe(*addr, nil))

}

// 網頁html
var homeTemplate = template.Must(template.New("").Parse(`
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script>  
window.addEventListener("load", function(evt) {
    var output = document.getElementById("output");
    var input = document.getElementById("input");
    var ws;
    var print = function(message) {
        var d = document.createElement("div");
        d.textContent = message;
        output.appendChild(d);
    };
    document.getElementById("open").onclick = function(evt) {
        if (ws) {
            return false;
        }
        ws = new WebSocket("{{.}}");
        ws.onopen = function(evt) {
            print("OPEN");
        }
        ws.onclose = function(evt) {
            print("CLOSE");
            ws = null;
        }
        ws.onmessage = function(evt) {
            print("RESPONSE: " + evt.data);
        }
        ws.onerror = function(evt) {
            print("ERROR: " + evt.data);
        }
        return false;
    };
    document.getElementById("send").onclick = function(evt) {
        if (!ws) {
            return false;
        }
        print("SEND: " + input.value);
        ws.send(input.value);
        return false;
    };
    document.getElementById("close").onclick = function(evt) {
        if (!ws) {
            return false;
        }
        ws.close();
        return false;
    };
});
</script>
</head>
<body>
<table>
<tr><td valign="top" width="50%">
<p>Click "Open" to create a connection to the server, 
"Send" to send a message to the server and "Close" to close the connection. 
You can change the message and send multiple times.
<p>
<form>
<button id="open">Open</button>
<button id="close">Close</button>
<p><input id="input" type="text" value="Hello world!">
<button id="send">Send</button>
</form>
</td><td valign="top" width="50%">
<div id="output"></div>
</td></tr></table>
</body>
</html>
`))
```

# Client

client.go
```go
package main

import (
	"bytes"
	"encoding/binary"
	"flag"
	"log"
	"net/url"
	"os"
	"os/signal"
	"time"

	"github.com/gorilla/websocket"
	"github.com/wonderivan/logger"
	"google.golang.org/protobuf/proto"

	myMsg "../proto"
)

// 定義flag參數，這邊會返回一個相應的指針
var addr = flag.String("addr", "localhost:1351", "http service address")

func main() {
	// 調用flag.Parse()解析命令行參數到定義的flag
	//flag.Parse()

	// SetFlags(flag int)可以用來自定義log的輸出格式
	//log.SetFlags(0)

	buf := make([]byte, 100)

	// 定義一個os.Signal的通道
	interrupt := make(chan os.Signal, 1)

	// Notify函數讓signal包將輸入信號轉到interrupt
	signal.Notify(interrupt, os.Interrupt)

	// 處理連接的網址
	u := url.URL{Scheme: "ws", Host: *addr, Path: "/echo"}
	logger.Debug("connecting to %s", u.String())

	// 連接服務器
	ws, _, err := websocket.DefaultDialer.Dial(u.String(), nil)
	if err != nil {
		log.Fatal("dial:", err)
	}

	// 預先關閉，此行在離開main時會執行
	defer ws.Close()

	// 定義通道
	done := make(chan struct{})

	go func() {
		// 預先關閉，此行在離開本協程時執行
		defer close(done)
		for {
			// 一直待命讀資料
			_, message, err := ws.ReadMessage()
			if err != nil {
				logger.Debug("read:", err)
				return
			}
			var pkgId = binary.BigEndian.Uint16(message[0:2])
			//var bodyId = binary.BigEndian.Uint16(string(myMsg.Command_Pong))
			if int16(pkgId) == int16(myMsg.Command_Pong) {
				// 將已經編碼的資料解碼成 protobuf.User 格式。
				var bodyClass myMsg.CtoSHeartBeat
				proto.Unmarshal(message[2:], &bodyClass)
				logger.Debug("recv: %v %v %v", pkgId, int16(myMsg.Command_Pong), message)
				//logger.Debug("bodyClass: %v %v", bodyClass.Balance, bodyClass.Code)
				logger.Debug("心跳")
			} else {
				logger.Debug("recv: %v ", message)
			}
		}
	}()

	// NewTicker 返回一個新的Ticker
	// 該Ticker包含一個通道字段，並會每隔時間段d就向該通道發送當時的時間
	// 它會調整時間間隔或者丟棄tick信自以適應反應慢的接收者
	// 如果d <= 0 會觸發panic，關閉該Ticker可以釋放相關資源
	ticker := time.NewTicker(1000000 * time.Second)
	heartbeat := time.NewTicker(2 * time.Second)

	// 預先停止，此行在離開main時執行
	defer ticker.Stop()

	for {
		select {
		case <-done:
			// 返回
			return
		case t := <-ticker.C:
			// ticker定義的時間到了會執行這邊
			//log.Println("<-ticker.C")
			message := t.String()
			err := ws.WriteMessage(websocket.TextMessage, []byte(message))
			if err != nil {
				logger.Debug("write:", err)
				return
			}
			logger.Debug("write:", message)
		case <-heartbeat.C:
			// ticker定義的時間到了會執行這邊
			//log.Println("<-heartbeat.C")

			var pkgId uint16
			pkgId = uint16(myMsg.Command_Ping)
			binary.BigEndian.PutUint16(buf[0:2], pkgId)
			data := myMsg.CtoSHeartBeat{}

			// 將資料編碼成 Protocol Buffer 格式（請注意是傳入 Pointer）。
			dataBuffer, _ := proto.Marshal(&data)

			// 將消息ID與DATA整合，一起送出
			pkgData := [][]byte{buf[:2], dataBuffer}
			pkgDatas := bytes.Join(pkgData, []byte{})
			err = ws.WriteMessage(websocket.BinaryMessage, pkgDatas)

			//message := "ping"
			//err := ws.WriteMessage(websocket.TextMessage, []byte(message))
			if err != nil {
				logger.Debug("write:", err)
				return
			}
			logger.Debug("write:", pkgDatas)
		case <-interrupt:
			// 強制執行程序時，會進入這邊
			log.Println("interrupt")

			// Cleanly close the connection by sending a close message and then
			// waiting (with timeout) for the server to close the connection.
			// 關閉連結並寄出close的的id
			err := ws.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
			if err != nil {
				logger.Debug("write close::", err)
				return
			}
			select {
			case <-done:
				// 結束完成會執行這邊
				logger.Debug("<-done")
			case <-time.After(10 * time.Second):
				// 超時處理，防止select阻塞著
				logger.Debug("<-time")
			}
			return
		}
	}
}
```

# 小結

使用之前練習的項目，將websocket與protobuf一起練習使用。