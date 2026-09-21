---
title: "Structural Design Patterns in Python"
date: 2024-05-13T00:49:10+07:00
tags: ["OOP", "Python", "Design Patterns"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Implement structural design patterns such as Adapter, Decorator and Facade in Python."
summary: "Implement structural design patterns such as Adapter, Decorator and Facade in Python."
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## Adapter Method

Adapter hiểu đơn giản là một lớp chuẩn hóa, nhằm để các đối tượng không tương tích có thể tương thích với nhau.

Ví dụ, chúng ta đang sử dụng điện thế 220V, trong khi các thiết bị điện tử chỉ tương thích với điện thế 15V. Chính vì thế các cục sạc đã được thiết kế có adapter để có thể tương thích với điện thế dân dụng 220V.

Một ví dụ khác, bạn có một module xử lý với dữ liệu dạng .json. Sau một thời gian, lượng dữ liệu tăng lên với nhiều định dạng khác nhau: .txt, .xml, .csv, … Thay vì bạn phải sửa module xử lý dữ liệu của mình, thì giờ chúng ta sẽ viết thêm một cục Adapter để parse toàn bộ dữ liệu sang dạng chỉ json.

```python
import json

class JsonView:
    def display(self):
        doc = {"text": "FOO"}
        print("Json object:", doc,'\n')

class TextView:
    def display(self):
        print("Text string:", "FOO",'\n')

class Adapter(JsonView):
    """
    The Adapter converts text string to json object
    """
    def __init__(self, adaptee: TextView) -> None:
        self.adaptee = adaptee

    def display(self, text: str):
        doc = {"text": text}
        print("Adapter:", doc,'\n')

if __name__ == "__main__":
    target = JsonView()
    target.display()

    adaptee = TextView()
    adaptee.display()

    adapter = Adapter(adaptee)
    adapter.display("BAR")
```

```
Json object: {'text': 'FOO'} 

Text string: FOO 

Adapter: {'text': 'BAR'}
```

## Bridge Method

Bridge method giúp bạn tách một lớp khổng lồ hoặc một tập hợp lớp có quan hệ gần gũi với nhau thành hai hệ thống phân cấp lớp riêng biệt là - abstraction (trừu tượng) và implementation (triển khai) - có thể phát triển độc lập với nhau. Method này có concept gần giống như nguyên tắc Single Responsibility, vì nó tách rời phần abstraction khỏi việc triển khai (implementation), thằng nào làm nhiệm vụ của thằng đó.

Ví dụ sau đây, chúng ta muốn xây dựng hệ thống điều khiển remote cho nhiều loại thiết bị điện tử: TV, DVD, … với các tính năng và lệnh khác nhau, thì triển khai như thế nào?

```python
from abc import ABC, abstractmethod

# Abstraction
class RemoteControl:
    """
    Xây dựng abstract interface, reference đến một đối tượng implementation (ở đây là device)
    """
    def __init__(self, device):
        self._device = device

    @abstractmethod
    def toggle_power(self):
        self._device.toggle_power()

    @abstractmethod
    def volume_up(self):
        self._device.volume_up()

    @abstractmethod
    def volume_down(self):
        self._device.volume_down()

# Implementation
class Device:
    """
    Xây dựng implementation interface
    """
    @abstractmethod
    def toggle_power(self):
        pass

    @abstractmethod
    def volume_up(self):
        pass

    @abstractmethod
    def volume_down(self):
        pass
```

```python
# Concrete Implementations for Devices
class TV(Device):
    """
    Xây dựng một Concrete Implementation cụ thể, ở đây là TV
    """
    def toggle_power(self):
        print("TV: Toggling power")

    def volume_up(self):
        print("TV: Volume up")

    def volume_down(self):
        print("TV: Volume down")

class DVDPlayer(Device):
    """
    Thêm một Concrete Implementation khác, ở đây là DVD
    """
    def toggle_power(self):
        print("DVD Player: Toggling power")

    def volume_up(self):
        print("DVD Player: Volume up")

    def volume_down(self):
        print("DVD Player: Volume down")

# Abstraction and Refined Abstraction
class BasicRemoteControl(RemoteControl):
    def toggle_power(self):
        print("Basic Remote: Press power button")
        super().toggle_power()

class AdvancedRemoteControl(RemoteControl):
    """
    Mở rộng abstract với các method khác.
    """
    def mute(self):
        print("Advanced Remote: Mute button pressed")
        self._device.volume_down()

# Client Code
if __name__ == "__main__":
    tv = TV()
    dvd_player = DVDPlayer()

    basic_remote_tv = BasicRemoteControl(tv)
    advanced_remote_dvd = AdvancedRemoteControl(dvd_player)

    basic_remote_tv.toggle_power()
    basic_remote_tv.volume_up()

    advanced_remote_dvd.toggle_power()
    advanced_remote_dvd.mute()
```

```
Basic Remote: Press power button
TV: Toggling power
TV: Volume up
DVD Player: Toggling power
Advanced Remote: Mute button pressed
DVD Player: Volume down
```

Sau này, chúng ta có thể dễ dàng thêm device mới, loại remote mới mà không cần thay đổi code hiện tại, sẽ dễ dàng bảo trì và mở rộng hơn.

## Composite Method

Composite method cho phép chúng ta sắp xếp các đối tượng thành cấu trúc Tree. Khi đó:

- Bạn có thể xử lý các đối tượng theo cùng một cách mà không phân biệt chúng là đối tượng đơn lẻ (leaf) hay là thành phần của một cấu trúc lớn hơn (composite).
- Dễ dàng thêm mới các thành phần: Bạn có thể thêm hoặc loại bỏ các thành phần từ cấu trúc cây mà không cần thay đổi code hiện tại, giúp code trở nên linh hoạt và dễ bảo trì.

Ví dụ sau đây biểu diễn một cây hệ thống tệp (file system tree):

- `FileSystemComponent` là một interface chung cho cả file và directory.
- `File` là một leaf class, biểu diễn một file trong hệ thống tệp.
- `Directory` là một composite class, biểu diễn một thư mục trong hệ thống tệp, có thể chứa cả các file và thư mục con.
- Tiến hành tạo các file và thư mục, sau đó thêm chúng vào các thư mục khác nhau.
- Cuối cùng, chúng ta tạo ra một thư mục gốc (root dir) chứa các thư mục và file đã tạo.

```python
from abc import ABC, abstractmethod

# Component interface
class FileSystemComponent:
    @abstractmethod
    def display(self):
        pass

# Leaf class
class File(FileSystemComponent):
    def __init__(self, name):
        self.name = name

    def display(self):
        """
        File là đơn vị nhỏ nhất (child element), nên chỉ display chính nó
        """
        print("File:", self.name)

# Composite class
class Directory(FileSystemComponent):
    def __init__(self, name):
        self.name = name
        self.components = []

    def add_component(self, component):
        """
        Thêm một component mới
        """
        self.components.append(component)

    def remove_component(self, component):
        """
        Xóa bỏ một component
        """
        self.components.remove(component)

    def display(self):
        """
        Display tất cả các components con
        """
        print("Directory:", self.name)
        for component in self.components:
            component.display()

# Client code
if __name__ == "__main__":
    # Create files
    file1 = File("file1.txt")
    file2 = File("file2.txt")
    file3 = File("file3.txt")

    # Create directories
    dir1 = Directory("Directory 1")
    dir2 = Directory("Directory 2")

    # Add files to directories
    dir1.add_component(file1)
    dir1.add_component(file2)
    dir2.add_component(file3)

    # Create composite directory
    root_dir = Directory("Root Directory")
    root_dir.add_component(dir1)
    root_dir.add_component(dir2)

    # Display the file system tree
    root_dir.display()
```

```
Directory: Root Directory
Directory: Directory 1
File: file1.txt
File: file2.txt
Directory: Directory 2
File: file3.txt
```

Như chúng ta có thể thấy, khi gọi `root_dir.display()`, composite sẽ đệ quy gọi vào các component nhỏ hơn và các leaf để hiển thị cây hệ thống tệp.

## Decorator Method

Decorator method cho phép chúng ta mở rộng hành vi của một đối tượng mà không cần phải thay đổi mã nguồn gốc của nó. Việc này có thể được thực hiện bằng cách truyền đối tượng gốc qua chuỗi các decorator, trong đó mỗi decorator là một function cung cấp hành vi cần bổ sung vào đổi tượng gốc.

Dưới đây là một ví dụ đơn giản sử dụng decorator để xử lý text:

```python
def uppercase_decorator(func):
    def wrapper(text):
        result = func(text)
        return result.upper()
    return wrapper

@uppercase_decorator
def greet(name):
    return f"Hello, {name}!"

print(greet("Alice"))
```

```
HELLO, ALICE!
```

Trong ví dụ trên, `uppercase_decorator` là một hàm decorator. Nó nhận một hàm khác làm đối số và trả về một hàm mới mở rộng hành vi của hàm ban đầu. Khi chúng ta gọi hàm greet, nó sẽ được thực thi thông qua hàm decorator và kết quả được chuyển thành chữ in hoa.

Một ví dụ khác khi sử dụng decorator để ghi log:

```python
import time

def log_writer(log_file):
    def decorator(func):
        def wrapper(*args, **kwargs):
            start_time = time.time()
            result = func(*args, **kwargs)
            total_time = time.time() - start_time
            output = f"Result: {result}, Time: {round(total_time, 4)}\n"
            with open(log_file, 'a') as f:
                f.write(output)
            return output
        return wrapper
    return decorator

@log_writer("test.txt")
def test():
    import random
    time.sleep(random.randint(0, 1))
    return random.randrange(0, 10)

@log_writer("test2.txt")
def test2():
    import random
    time.sleep(random.randint(1, 2))
    return random.choice(["apple", "banana", "cherry"])

if __name__ == "__main__":
    for _ in range(3):
        test()
        test2()
```

```python
!cat test.txt && echo $'\n' && cat test2.txt
```

```
Result: 4, Time: 0.0
Result: 3, Time: 1.0011
Result: 9, Time: 0.0

Result: apple, Time: 1.0007
Result: cherry, Time: 1.0011
Result: cherry, Time: 2.0021
```

Trong ví dụ trên, `log_writer` là một hàm decorator, nhận tham số `log_file` truyền vào. Decorator này chạy qua hàm chính, lấy kết quả và tính thời gian thực thi để ghi vào file log. Sau này chúng ta muốn ghi log khi chạy qua một hàm bất kỳ thì chỉ cần thêm hàm decorator `log_writer` là được.

## Facade Method

Facade method cho phép chúng ta triển khai interface giao tiếp giữa client và các hệ thống con một cách dễ dàng.

Ví dụ trong thực tế, một chiếc máy giặt có thể giặt, xả hay vắt quần áo nhưng tất cả các công việc đều riêng biệt. Khi đó chúng ta sẽ phải cần đến một hệ thống có thể tự động toàn bộ công việc mà không cần can thiệp, đó chính là Facade.

```python
"""Facade pattern with an example of WashingMachine"""
class Washing: 
    '''Subsystem # 1'''
    def wash(self): 
        print("Washing...") 

class Rinsing: 
    '''Subsystem # 2'''
    def rinse(self): 
        print("Rinsing...") 

class Spinning: 
    '''Subsystem # 3'''
    def spin(self): 
        print("Spinning...") 

class WashingMachine: 
    '''Facade'''
    def __init__(self): 
        self.washing = Washing() 
        self.rinsing = Rinsing() 
        self.spinning = Spinning() 

    def startWashing(self): 
        self.washing.wash() 
        self.rinsing.rinse() 
        self.spinning.spin() 

""" client code """
if __name__ == "__main__": 
    washingMachine = WashingMachine()
    washingMachine.startWashing()
```

```
Washing...
Rinsing...
Spinning...
```

### Khi nào thì sử dụng Facade?

- Gom nhóm chức năng lại để client dễ sử dụng, thay vì phải tìm hiểu quy trình xử lý của chương trình.
- Đóng gói nhiều chức năng, che giấu thuật toán phức tạp.
- Xây dựng một interface đơn giản, dễ sử dụng mà không bị phụ thuộc quá nhiều vào hệ thống con.

### Rủi ro khi sử dụng Facade?

- Facade của bạn có thể trở lên quá lớn, làm quá nhiều nhiệm vụ với nhiều hàm chức năng trong nó -> phá vỡ quy tắc SOLID.
- Dư thừa, khi mà hệ thống của bạn không quá phức tạp, ít hệ thống con.

## Proxy Method

Proxy Method cho phép chúng ta tạo ra một lớp trung gian (proxy) để kiểm soát quyền truy cập vào đối tượng thực sự, hay nói cách khác nó cung cấp một đối tượng thay thế cho một đối tượng thực sự, từ đó cho phép kiểm soát hoặc mở rộng hành vi của đối tượng đó một cách linh hoạt.

Ví dụ trong thực tế, thẻ ATM là thứ “đại diện” cho tiền mặt trong tài khoản ngân hàng của chúng ta, nó chính là `Proxy` để kiểm soát và quản lý quyền truy cập vào đối tượng cụ thể (ở đây chính là tiền mặt). Hay như nginx cũng là một proxy để điều hướng traffic từ port 80, 443 vào các service tương ứng.

### Ứng dụng

- Kiểm soát truy cập: proxy kiểm tra xem ai có quyền truy cập hay không, ví dụ: authen, nginx, …
- Quản lý hiệu năng: proxy lưu trữ các kết quả đã được tính toán trước đó, tránh việc tính lại nhiều lần, ví dụ: caching, …

```python
"""
Ví dụ sử dụng Proxy để làm cache
"""
import time

# Define the interface for the Real Subject
class DatabaseQuery:
    def execute_query(self, query):
        pass

# Real Subject: Represents the actual database
class RealDatabaseQuery(DatabaseQuery):
    def execute_query(self, query):
        print(f"Executing query: {query}")
        # Simulate a database query and return the results
        return f"Results for query: {query}\n"

# Proxy: Caching Proxy for Database Queries
class CacheProxy(DatabaseQuery):
    def __init__(self, real_database_query, cache_duration_seconds):
        self._real_database_query = real_database_query
        self._cache = {}
        self._cache_duration = cache_duration_seconds

    def execute_query(self, query):
        if query in self._cache and time.time() - self._cache[query]["timestamp"] <= self._cache_duration:
            # Return cached result if it's still valid
            print(f"CacheProxy: Returning cached result for query: {query}")
            return self._cache[query]["result"]
        else:
            # Execute the query and cache the result
            result = self._real_database_query.execute_query(query)
            self._cache[query] = {"result": result, "timestamp": time.time()}
            return result

# Client code
if __name__ == "__main__":
    # Create the Real Subject
    real_database_query = RealDatabaseQuery()

    # Create the Cache Proxy with a cache duration of 5 seconds
    cache_proxy = CacheProxy(real_database_query, cache_duration_seconds=5)

    # Perform database queries, some of which will be cached
    print(cache_proxy.execute_query("SELECT * FROM table1"))
    print(cache_proxy.execute_query("SELECT * FROM table2"))
    time.sleep(3)  # Sleep for 3 seconds
    
    # Should return cached result
    print(cache_proxy.execute_query("SELECT * FROM table1"))
    print(cache_proxy.execute_query("SELECT * FROM table3"))
```

```
Executing query: SELECT * FROM table1
Results for query: SELECT * FROM table1

Executing query: SELECT * FROM table2
Results for query: SELECT * FROM table2

CacheProxy: Returning cached result for query: SELECT * FROM table1
Results for query: SELECT * FROM table1

Executing query: SELECT * FROM table3
Results for query: SELECT * FROM table3
```

```python
"""
Ví dụ sử dụng Proxy để kiểm soát truy cập: validation, protection
"""
class College: 
    '''Resource-intensive object'''
    def studyingInCollege(self): 
        print("Studying In College....\n") 

class CollegeProxy: 
    '''Relatively less resource-intensive proxy acting as middleman. 
    Instantiates a College object only if there is no fee due.'''
    def __init__(self): 
        self.feeBalance = 1000
        self.college = None

    def studyingInCollege(self): 
        print(f"Proxy in action. Checking fee balance: {self.feeBalance}")
        if self.feeBalance <= 500:
            # If the balance is less than 500, let him study. 
            self.college = College() 
            self.college.studyingInCollege() 
        else: 
            # Otherwise, don't instantiate the college object. 
            print("Your fee balance is greater than 500, first pay the fee.\n") 

# Client code
if __name__ == "__main__": 
    # Instantiate the Proxy 
    collegeProxy = CollegeProxy() 

    # Client attempting to study in the college at the default balance of 1000. 
    # Logically, since he / she cannot study with such balance, 
    # there is no need to make the college object. 
    collegeProxy.studyingInCollege() 

    # Altering the balance of the student 
    collegeProxy.feeBalance = 100
    # Client attempting to study in college at the balance of 100. Should succeed. 
    collegeProxy.studyingInCollege()
```

```
Proxy in action. Checking fee balance: 1000
Your fee balance is greater than 500, first pay the fee.

Proxy in action. Checking fee balance: 100
Studying In College....
```

## Flyweight Method

Flyweight Method cho phép chúng ta giảm thiểu số lượng object mà chương trình yêu cầu khi đang chạy, hiểu nôm na là đối tượng Flyweight có thể được share cho các đối tượng, và khi đó chúng ta sẽ không thể phân biệt giữa một object và một Flyweight object.

Các bước triển khai Flyweight method:

- Xây dựng các phần được chia sẻ, không thể thay đổi của một đối tượng.
- Xây dựng các phần có thể thay đổi, theo ngữ cảnh cụ thể của một đối tượng.

Ví dụ thực tế, giả sử chúng ta đang xây dựng một trình soạn thảo văn bản đơn giản và muốn biểu diễn các ký tự dưới dạng đối tượng. Tuy nhiên, thay vì tạo một đối tượng riêng cho từng ký tự, chúng ta sẽ sử dụng mẫu Flyweight để chia sẻ thông tin chung về ký tự.

```python
class CharFlyweight:
    def __init__(self, char):
        self.char = char

class CharFactory:
    char_flyweights = {}

    @staticmethod
    def get_char(char):
        """
        Kiểm tra char object đã được khởi tạo hay chưa.
        - Đã khởi tạo -> trả về object đã khởi tạo, không tạo thêm để tiết kiệm bộ nhớ
        - Chưa khởi tạo -> khởi tạo -> lưu vào cache để sau dùng lại.
        """
        if char not in CharFactory.char_flyweights:
            CharFactory.char_flyweights[char] = CharFlyweight(char)
        return CharFactory.char_flyweights[char]

class Character:
    def __init__(self, char, font_size):
        # Thông tin về ký tự được chia sẻ
        self.char_flyweight = CharFactory.get_char(char)
        # Thông tin về font size thì có thể thay đổi
        self.font_size = font_size

    def render(self):
        print(f"Object id: {id(self.char_flyweight)}, Character: {self.char_flyweight.char}, Font Size: {self.font_size}\n")

# Client code
if __name__ == "__main__":
    characters = []
    characters.append(Character('A', 12))
    characters.append(Character('B', 13))
    characters.append(Character('A', 14))  # Reusing 'A' from flyweight

    for character in characters:
        character.render()
```

```
Object id: 139701667453152, Character: A, Font Size: 12

Object id: 139701666563264, Character: B, Font Size: 13

Object id: 139701667453152, Character: A, Font Size: 14
```

Đi đến đây chúng ta có thể thấy `Flyweight` khá giống với `Singleton`, ở chỗ nó sử dụng lại đối tượng đã được khởi tạo. Tuy nhiên cần lưu ý như sau:

- `Flyweight` sẽ giống với `Singleton` nếu chúng ta có thể giảm tất cả các trạng thái được chia sẻ của đối tượng cuống còn 1 đối tượng.
- `Singleton` chỉ có duy nhất một instance, trong khi `Flyweight` có thể có nhiều instance với các trạng thái nội tại khác nhau (Intrinsic state).
- `Singleton` có thể thay đổi hoặc bất biến (thay đổi trong trường hợp trong class Singleton chúng ta có các method setter), trong khi đó `Flyweight` là bất biến.

You can get full source code [here](https://github.com/hoangph3/Object-Oriented-Programming/blob/main/DesignPattern/Structural.ipynb)!
