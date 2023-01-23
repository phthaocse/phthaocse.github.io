---
date: 2022-04-02 17:35
layout: post
description: Concurrency Problems - Race condition 
tags: Vietnamese, golang, race-condition, concurrency, thao-posts
---
# Race condition là gì ?

## Định nghĩa
Race condition là tình huống một hay nhiều thread cùng cố gắng thay đổi giá trị của một biến trong vùng nhớ chung (shared memory) tại cùng một thời điểm, kết quả của việc thực thi phụ thuộc vào thứ tự cụ thể mà việc truy cập diễn ra, và thứ tự truy cập đó ta không thể kiểm soát. Điều này có thể dẫn đến kết quả không mong muốn của toàn bộ quá trình.

## Ví dụ
Để hiểu rõ về vấn đề này mình đã viết một chương trình minh họa race condition bằng Go, mình sẽ dùng một khái niệm gần tương tự thread ở level OS để minh họa đó là [Gorountine](https://go.dev/tour/concurrency/1) - lightweight thread được cung cấp bởi Go. Trong ví dụ này chúng ta sẽ mô phỏng việc "Thêm vào giả hàng" ở các trang thương mại điện tử, giả sử có một mặt hàng X hiện đang còn 1000 sản phẩm trong kho, tại thời điểm mở bán, 1000 khách hàng đồng thời thêm sản phẩm đó vào giỏ hàng. Sau đó ta sẽ kiểm tra xem liệu số lượng tồn kho sau khi 1000 khách hàng đã thêm vào giỏ có đúng bằng 0 theo đúng lý thuyết. Chúng ta sẽ mô phỏng 1000 request thêm vào giỏ hàng bằng việc tạo ra 1000 Gorountines và nhiệm vụ của mỗi Goroutine là giảm số lượng `stock` đi 1 đơn vị, `stock` ở đây sẽ global variable và được sử dụng chung bởi tất cả Gorountine như điều kiện đã đặt ra.

### Tiến hành
```go
var stock int = 1000

func add_to_cart(w *sync.WaitGroup) {
	defer w.Done()
	stock--
}

func main() {
	var w sync.WaitGroup
    w.Add(1000)
	for i := 0; i < 1000; i++ {
		go add_to_cart(&w)
	}
	w.Wait()
	fmt.Println("Current stock: ", stock)
}

```
Trong đoạn code trên mình có sử dụng một số utilities của `sync` package để giúp cho việc mô phỏng diễn ra chính xác. Vì hàm main (entry point của bất kỳ application nào trong Go) cũng là một Gorountine và một khi nó kết thúc thì tất cả các Gorountine khác cũng bị drop nên `w.Wait()` xuất hiện ở đây đề giúp cho `main` chờ các Gorountine khác kết thúc trước khi nó kết thúc. `w.Add()` được gọi trong main Goroutine để thêm vào số lượng Goroutine mà nó cần chờ, ở đây là 1000 Goroutines. Sau khi hoàn thành công việc thì Goroutine sẽ signal cho main Gorountine bằng việc gọi hàm `w.Done()` mà ta có thể thấy trong `add_to_cart` qua câu lệnh `defer w.Done()`, [defer](https://go.dev/tour/flowcontrol/12) là một keyword trong Go dùng để nói với hàm sở hữu nó thực hiện câu lệnh ngay sau keyword `defer` tại thời điểm kết thúc.

### Kết quả và giải thích
```shell
Current stock:  93
Current stock:  32
Current stock:  114
Current stock:  22
Current stock:  6
```

Sau khi tiến hành thực thi chương trình ở trên 5 lần chúng ta nhận được 5 kết quả khác nhau và không có kết quả nào đúng như kết quả mà ta mong muốn là số lượng stock còn lại là 0. Vậy tại sao kết quả lại như vậy ?

Trong phần định nghĩa có nói kết quả của quá trình thực thi sẽ phụ thuộc vào thứ tự mà việc truy cập vào vùng nhớ chung diễn ra. Ta cần biết rằng các Gorountine sẽ được thực thi theo một thứ tự mà ta không thể kiểm soát, mà nó được quyết định bởi runtime scheduler của Go cũng tương tư thread ở level OS  sẽ được quyết định bởi OS scheduler. Việc này dẫn tới tại một thời điểm các Goruntine được thực thi cùng lúc với nhau dẫn đến việc biến `stock` được truy cập cùng lúc. Đây cũng là nguyên nhân dẫn tới việc biến `stock` bị tính toán sai mà ở đây một cách chi tiết là qua câu lệnh `stock--`. Vậy ta cùng nhìn sâu hơn ở low-level cuả câu lệnh `stock--`, ở mức độ ngôn ngữ máy, nó được thực hiện như sau
```shell
register = stock
register = register - 1
stock = register
```
Giá trị từ biến `stock` sẽ được đọc và ghi xuống thanh ghi, sau đó tiến hành tính toán bởi CPU và được ghi lại vào biến `stock`. Điều này dẫn đến việc khi hai hay nhiều truy cập đồng thời vào biến stock thì giá trị tương ứng của `stock` sẽ được ghi vào nhiều thanh ghi khác nhau với cùng một giá trị, vì thế làm cho giá trị của `stock` bị tính toán sai. Ví dụ ta có 3 Gorountines cùng truy cập vùng nhớ của `stock` và thực thi phép toán, thì lúc này giá trị của `stock` thay vì cần giảm 3 đơn vị thì nó chỉ giảm 1.

```shell
Goroutine1: stock-- =>  register1 = stock (stock=1)
                        register1 = register1 - 1
                        stock = register1 (stock = 0)
Goroutine2: stock-- =>  register2 = stock (stock=1)
                        register2 = register2 - 1
                        stock = register2 (stock = 0)
Goroutine3: stock-- =>  register3 = stock (stock=1)
                        register3 = register3 - 1
                        stock = register3 (stock = 0)
```

Đó là những gì bên dưới đã xảy ra và những điều này chúng ta sẽ không thể thấy được ở high level tức là trên những dòng code mà ta viết nên một khi gặp phải thì việc debug sẽ trở nên rất khó khăn. Và qua ví dụ ta cũng thấy vấn đề xảy ra do việc có nhiều process cùng truy cập vào một vùng một cách đồng thời đã được đề cập trong phần định nghĩa. Từ đây ta cũng biết cách để phòng tránh race condition bằng việc can thiệp để các process không được cùng truy cập vào cùng một vùng nhớ chung tại một thời điểm nhất định.

### Giải pháp

Mời mọi người đón xem ở trong [phần tiếp theo](https://hung-m-dao.github.io/thao/race-condition-solution/)


### Read more

* [Race condition](https://en.wikipedia.org/wiki/Race_condition)
* [Chapter 6 - Operating System Concepts by Abraham Silberschatz, Peter B. Galvin, Greg Gagne](https://codex.cs.yale.edu/avi/os-book/OS10/practice-exercises/index-solu.html)
