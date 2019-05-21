

5000/5000
Giới hạn ký tự: 5000
# Bạn không biết JS: Async & Performance
# Chương 4: Generators

Trong Chương 2, chúng ta đã xác định hai nhược điểm chính để thể hiện kiểm soát luồng không đồng bộ với Callbacks:

* Async dựa trên cuộc gọi lại không phù hợp với cách bộ não của chúng ta lên kế hoạch cho các bước của một nhiệm vụ.
* Callbacks không đáng tin cậy hoặc có thể kết hợp được do * Inversion of Control *.

Trong Chương 3, chúng ta đã trình bày chi tiết về cách Promises đảo ngược lại * Inversion of Control * của Callbacks, khôi phục trustability/composability.

Bây giờ chúng ta chuyển sự chú ý của mình để thể hiện kiểm soát dòng không đồng bộ theo kiểu tuần tự và đồng bộ. "Phép thuật" làm cho nó có thể là ES6 ** Generators **.

## Breaking Run-to-Completion

Trong Chương 1, chúng ta đã giải thích một kỳ vọng của hầu hết các nhà phát triển JS đối với code của họ: một khi một hàm bắt đầu thực thi, nó sẽ chạy cho đến khi hoàn thành và không có code nào khác có thể làm gián đoạn và chạy ở giữa.

ES6 giới thiệu một loại chức năng mới không phải hoạt động giống với hành vi chạy đến hoàn thành. Loại chức năng mới này được gọi là một "Generator."

Để hiểu các hàm ý, hãy xem xét ví dụ này:

```js
var x = 1;

function foo() {
	x++;
	bar();				// <-- what about this line?
	console.log( "x:", x );
}

function bar() {
	x++;
}

foo();					// x: 3
```

Trong ví dụ này, chúng ta biết chắc chắn rằng `bar ()` chạy ở giữa` x ++ `và` console.log (x) `. Nhưng nếu `bar ()` không ở đó thì sao? Rõ ràng, kết quả sẽ là `2` thay vì` 3`.

Điều gì xảy ra nếu `bar ()` không có mặt, nhưng nó vẫn có thể chạy bằng cách nào đó giữa các câu lệnh `x ++` và` console.log (x) `? Làm thế nào để thực hiện điều đó?

Trong các ngôn ngữ đa luồng ** preemptive **, về cơ bản có thể cho `bar ()` "ngắt" và chạy chính xác vào đúng thời điểm giữa hai câu lệnh đó. Nhưng JS không phải là preemptive, cũng không phải là (hiện tại) đa luồng. Tuy nhiên, một dạng ** cooperative ** của "sự gián đoạn" này (concurrency) là có thể, nếu chính `foo ()` có thể chỉ ra "pause" ở phần đó trong code.

** Lưu ý: ** ta sử dụng từ "cooperative" không chỉ vì liên quan đến thuật ngữ concurrency cổ điển (xem Chương 1), mà vì như bạn sẽ thấy trong đoạn trích tiếp theo, cú pháp ES6 để chỉ ra điểm tạm dừng trong code là `yield`.

Đây là code ES6 để thực hiện đồng thời hợp tác như vậy:

```js
var x = 1;

function *foo() {
	x++;
	yield; // pause!
	console.log( "x:", x );
}

function bar() {
	x++;
}
```

** Lưu ý: ** Bạn có thể sẽ thấy hầu hết các tài liệu / code JS khác sẽ định dạng khai báo trình tạo là `function * foo () {..}` thay vì như ta đã làm ở đây với` function * foo () { ..} `- sự khác biệt duy nhất là chỗ đặt dấu ` * `. Hai kiểu giống nhau về mặt chức năng / cú pháp, như là một dạng `hàm * foo () {..}` (không có khoảng trắng) thứ ba. Dùng kiểu nào cũng được, nhưng về cơ bản ta thích `function * foo..` vì nó phù hợp khi ta tham chiếu một trình tạo bằng văn bản với` * foo ()`. Nếu ta chỉ nói `foo ()`, bạn sẽ không biết rõ nếu ta đang nói về một trình tạo hay một hàm thông thường. Đó hoàn toàn là một lựa chọn mang tính phong cách.

Bây giờ, làm thế nào chúng ta có thể chạy đoạn code ở trên sao cho `bar ()` thực thi tại điểm của `ield` bên trong `* foo ()`?

```js
// construct an iterator `it` to control the generator
var it = foo();

// start `foo()` here!
it.next();
x;						// 2
bar();
x;						// 3
it.next();				// x: 3
```

Trước khi chúng ta giải thích các cơ chế / cú pháp khác nhau với các trình tạo ES6, hãy xem qua luồng hành vi:

1. Hoạt động `it = foo ()` *không * thực thi trình tạo `* foo ()`, nhưng nó chỉ xây dựng một * iterator * sẽ điều khiển việc thực thi của nó.
2. Đoạn `it.next ()` đầu tiên khởi động generator `* foo ()` và chạy `x ++` trên dòng đầu tiên của `* foo ()`.
3. `* foo ()` tạm dừng tại câu lệnh `yield`, tại điểm đầu tiên mà call đến ` it.next () `kết thúc. Hiện tại, `* foo ()` vẫn đang chạy và hoạt động, nhưng nó ở trạng thái tạm dừng.
4. Chúng ta kiểm tra giá trị của `x`, bây giờ là` 2`.
5. Chúng ta gọi `bar ()`, tăng `x` lần nữa với` x ++`.
6. Chúng ta kiểm tra giá trị của `x` một lần nữa bây giờ là` 3`.
7. Cuộc gọi `it.next ()` cuối cùng nối lại trình tạo `* foo ()` từ nơi nó bị tạm dừng và chạy câu lệnh `console.log (..)`, sử dụng giá trị hiện tại của` x` của `3`.

Rõ ràng, `* foo ()` đã bắt đầu, nhưng * không * chạy để hoàn thành - nó dừng lại ở `ield`. Chúng ta đã tiếp tục `* foo ()` sau đó và để nó kết thúc, nhưng điều đó thậm chí không bắt buộc.

Vì vậy, generator là một loại hàm đặc biệt có thể khởi động và dừng một hoặc nhiều lần và không nhất thiết phải hoàn thành. Mặc dù chưa không rõ ràng tại sao nó lại mạnh mẽ như vậy, khi chúng ta đi suốt phần còn lại của chương này, đó sẽ là một trong những khối xây dựng cơ bản mà chúng ta sử dụng để xây dựng generator-as-async-flow-control như một mẫu code.
