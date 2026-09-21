---
title: "Creational Design Patterns in Python"
date: 2024-05-20T00:19:30+07:00
tags: ["OOP", "Python", "Design Patterns"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Implement creational design patterns such as Factory, Builder and Singleton in Python."
summary: "Implement creational design patterns such as Factory, Builder and Singleton in Python."
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

## Factory Method

Factory method là pattern giúp khởi tạo các đối tượng thông qua một interface chung, qua đó:

- Che giấu quá trình xử lý logic của phương thức khởi tạo.
- Giảm sự phụ thuộc, dễ dàng mở rộng.

```python
"""
Đoạn code này không sử dụng Factory Method
"""
class FrenchLocalizer:
    """it simply returns the french version """
    def __init__(self):
        self.translations = {"car": "voiture", "bike": "bicyclette",
                             "cycle":"cyclette"}
    def localize(self, msg):
        """change the message using translations"""
        return self.translations.get(msg, msg)

class SpanishLocalizer:
    """it simply returns the spanish version"""
    def __init__(self):
        self.translations = {"car": "coche", "bike": "bicicleta",
                             "cycle":"ciclo"}
    def localize(self, msg):
        """change the message using translations"""
        return self.translations.get(msg, msg)

class EnglishLocalizer:
    """Simply return the same message"""
    def localize(self, msg):
        return msg

if __name__ == "__main__":
    # Business logic
    f = FrenchLocalizer()
    e = EnglishLocalizer()
    s = SpanishLocalizer()

    msg = "car"
    lang = "French"
    if lang == "French":
        print(f.localize(msg))
    elif lang == "English":
        print(e.localize(msg))
    elif lang == "Spanish":
        print(s.localize(msg))
    else:
        raise ValueError()
```

```
voiture
```

Với đoạn code trên, ta có thể nhận thấy các vấn đề như sau:

- Code khởi tạo đối tượng nằm ở phía client, bao gồm: FrenchLocalizer, EnglishLocalizer, SpanishLocalizer. Như vậy sau này cứ mỗi lần phía thư viện có update mới, chẳng hạn như support thêm VietnameseLocalizer, JapaneseLocalizer hay ChineseLocalizer, … thì phía client cũng phải update thêm code khởi tạo để có thể sử dụng được.
- Tùy thuộc vào bussiness logic mà chúng ta sẽ tạo ra các object tương ứng, nếu logic này nằm ở nhiều nơi khác nhau thì nó sẽ bị lặp đi lặp lại. Đặc biệt là khi chúng ta muốn thay đổi, chỉnh sửa hay mở rộng, ta đều phải sửa tất cả những nơi có logic đó gây mất thời gian và dễ bị sót hay lỗi.

### Giải pháp

- Gom các business logic để khởi tạo object vào một nơi trong chương trình, gọi dân dã là đóng hàm, và nó chính là `factory method`.
- Khi đó, người sử dụng (client) đạt được mục đích tạo mới object và không cần quan tâm đến cách nó được tạo ra (vì `factory method` đã che giấu logic này).

```python
"""
Đoạn code áp dụng Factory Method
"""
class FrenchLocalizer:
    """it simply returns the french version """
    def __init__(self):
        self.translations = {"car": "voiture", "bike": "bicyclette",
                             "cycle":"cyclette"}
    def localize(self, msg):
        """change the message using translations"""
        return self.translations.get(msg, msg)

class SpanishLocalizer:
    """it simply returns the spanish version"""
    def __init__(self):
        self.translations = {"car": "coche", "bike": "bicicleta",
                             "cycle":"ciclo"}
    def localize(self, msg):
        """change the message using translations"""
        return self.translations.get(msg, msg)

class EnglishLocalizer:
    """Simply return the same message"""
    def localize(self, msg):
        return msg

def Factory(language="English"):
    """Factory Method"""
    localizers = {
        "French": FrenchLocalizer,
        "English": EnglishLocalizer,
        "Spanish": SpanishLocalizer,
    }
    return localizers[language]()

if __name__ == "__main__":
    message = "car"
    lang = "French"
    fn = Factory(lang)
    print(fn.localize(message))
```

```
voiture
```

### Factory method có ưu điểm gì?

- Chúng ta có một super class với nhiều class con và dựa trên đầu vào, chúng ta cần trả về một class con. Mô hình này giúp chúng ta đưa trách nhiệm (hay logic if else) của việc khởi tạo một class từ phía người dùng (client) sang lớp Factory là một nơi trong chương trình (đáp ứng Single Responsibility Principle).
- Nhờ việc che giấu hay kiểm soát logic khởi tạo, chúng ta có thể tiết kiệm tài nguyên bằng cách sử dụng lại đối tượng hiện có thay vì tạo mới chúng mỗi lần (nghe có vẻ liên quan đến `Singleton`):
    - Bạn cần nơi để lưu trữ tất cả các đối tượng đã tạo.
    - Khi ai đó yêu cầu một đối tượng, chương trình sẽ thực hiện tìm kiếm đối tượng đó trong pool.
    - …và trả về cho code client.
    - Nếu không có đối tượng, chương trình sẽ tạo ra một đối tượng mới (và thêm nó vào pool).
- Chúng ta không thể biết được liệu sau này còn có class con nào không. Khi cần mở rộng, hãy tạo subclass và thêm vào Factory Method để khởi tạo subclass này mà không làm ảnh hưởng đến code client hiện tại (đáp ứng Open/Closed Principle). Ví dụ:

```python
# Tạo subclass
class VietnameseLocalizer:
    """it simply returns the spanish version"""
    def __init__(self):
        self.translations = {"car": "xe hơi", "bike": "xe đạp",
                             "cycle":"xích lô"}
    def localize(self, msg):
        """change the message using translations"""
        return self.translations.get(msg, msg)

# Thêm vào Factory Method
def Factory(language="English"):
    """Factory Method"""
    localizers = {
        "French": FrenchLocalizer,
        "English": EnglishLocalizer,
        "Spanish": SpanishLocalizer,
        "Vietnamese": VietnameseLocalizer
    }
    return localizers[language]()

message = "car"
lang = "Vietnamese"
fn = Factory(lang)
print(lang)
print(fn.localize(message))
```

```
Vietnamese
xe hơi
```

Một cách khác tạo factory method sử dụng `staticmethod`

```python
class Factory:
    _localizers = {
        "French": FrenchLocalizer,
        "English": EnglishLocalizer,
        "Spanish": SpanishLocalizer,
        "Vietnamese": VietnameseLocalizer
    }
    @staticmethod
    def create_localizer(lang: str):
        return Factory._localizers[lang]()

fn = Factory.create_localizer("Vietnamese")
fn.localize(message)
```

```
'xe hơi'
```

### Ứng dụng

- Hệ thống quản lý tài liệu: có nhiều loại tài liệu khác nhau như Word, PDF, Excel, … Sử dụng Factory Method giúp chúng ta tạo ra các đối tượng tài liệu tương ứng mà không cần phải biết trước loại tài liệu cụ thể.
- Hệ thống thanh toán: hệ thống thanh toán hỗ trợ nhiều phương thức thanh toán khác nhau như thẻ ATM, thẻ tín dụng, PayPal, … Sử dụng Factory Method sẽ giúp chúng ta tạo ra đối tượng xử lý thanh toán tương ứng với phương thức thanh toán mà người dùng lựa chọn.
- Ứng dụng đồ họa: ứng dụng có thể xử lý với nhiều loại hình ảnh khác nhau (JPG, PNG, BMP, …). Một Factory Method có thể được sử dụng để tạo ra các đối tượng hình ảnh tương ứng tùy thuộc vào định dạng của file ảnh được mở.
- Kết nối database: sử dụng Factory Method để tạo ra các đối tượng kết nối với cơ sở dữ liệu cụ thể (MySQL, PostgreSQL, SQL Server, …), giúp chúng ta dễ dàng mở rộng khi có các cơ sở dữ liệu mới (Mongo, Elasticsearch, …)

## Abstract Factory Method

Abstract Factory Pattern còn được gọi là Factory of Factories, nó cũng tương tự như Factory Pattern, nhưng cung cấp một mức trừu tượng hơn (abstract) để tạo ra các đối tượng liên quan đến nhau.

Hãy ví dụ với hệ điều hành, mỗi hệ điều hành sẽ hiển thị UI style khác nhau từ button, dock, … Bạn sẽ không muốn một chương trình hiển thị
MacOS control khi đang chạy Windows. Đó chính là lý do mà Abstract Factory ra đời, nó hoạt động như sau:

1. Kiểm tra hệ điều hành hiện tại.
2. Tạo ra đối tượng `Factory` từ class tương ứng với hệ điều hành.
3. Các phần còn lại sử dụng đối tượng `Factory` này để tạo ra các phần tử UI tương thích với hệ điều hành đó.

Bằng cách tiếp cận này, mỗi lần thêm biến thể của phần tử UI trong ứng dụng, chúng ta chỉ cần tạo một class `Factory` mới tạo ra các phần tử UI này và sửa đổi một chút code khởi tạo để ứng dụng chọn class đó khi thích hợp.

```python
"""
Code with Abstract Factory Method
"""
from __future__ import annotations
from abc import ABC, abstractmethod

class Button(ABC):
    """
    Mỗi phần tử UI của từng OS phải có base interface.
    Ở đây ta tạo Button với các biến thể khác nhau: WindowsButton, MacOSButton 
    """
    @abstractmethod
    def render(self) -> str:
        pass

class WindowsButton(Button):
    def render(self) -> str:
        return f"This is {__class__.__name__}"

class MacOSButton(Button):
    def render(self) -> str:
        return f"This is {__class__.__name__}"

"""
Thực hiện tương tự với các phần tử UI khác, ví dụ: dock
"""
class Dock(ABC):
    """
    Ở đây ta tạo Dock với các biến thể khác nhau: WindowsDock, MacOSDock
    """
    @abstractmethod
    def render(self) -> None:
        pass

class WindowsDock(Dock):
    def render(self) -> str:
        return f"This is {__class__.__name__}"

class MacOSDock(Dock):
    def render(self) -> str:
        return f"This is {__class__.__name__}"
```

```python
"""
Abstract Factory là interface với các methods trả về Abstract Products.
Những Products trong này có liên quan đến nhau dựa trên concept nào đó, ví dụ: cùng OS
"""
class AbstractFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button:
        pass

    @abstractmethod
    def create_dock(self) -> Dock:
        pass

class WindowsFactory(AbstractFactory):
    """
    Concrete Factories override các methods để trả về các Products thuộc cùng 1 biến thể: Windows UI
    """
    def create_button(self):
        return WindowsButton()

    def create_dock(self):
        return WindowsDock()

class MacOSFactory(AbstractFactory):
    """
    Concrete Factories override các methods để trả về các Products thuộc cùng 1 biến thể: MacOS UI
    """
    def create_button(self):
        return MacOSButton()

    def create_dock(self):
        return MacOSDock()
```

```python
"""
Create Factory method
"""
class GUIFactory:
    _os = {
        "macos": MacOSFactory,
        "windows": WindowsFactory,
    }
    @staticmethod
    def create_factory(name: str):
        return GUIFactory._os[name]()

"""
Code client
"""
# Client
class Application:
    def __init__(self, factory: GUIFactory):
        self.factory = factory

    def create_ui(self):
        button = self.factory.create_button()
        print(button.render())

        dock = self.factory.create_dock()
        print(dock.render())

if __name__ == "__main__":
    factory = GUIFactory.create_factory("macos")
    app = Application(factory)
    app.create_ui()
    print()
    
    factory = GUIFactory.create_factory("windows")
    app = Application(factory)
    app.create_ui()
```

```
This is MacOSButton
This is MacOSDock

This is WindowsButton
This is WindowsDock
```

### Áp dụng Abstract Factory khi nào?

Ban đầu khi mới phát triển, chúng ta có thể chỉ cần dùng Factory Method. Tuy nhiên khi dự án ngày càng mở rộng, càng có nhiều loại đối tượng hơn, thì chúng ta nên sử dụng Abstract Factory để tách nhiều biến thể của một nhóm sản phẩm (products) thành một Factory. Khi đó:

- Các sản phẩm lấy từ một factory sẽ tương thích với nhau.
- Tránh được kết hợp quá chặt chẽ giữa code client và concrete product, bạn có thể thêm các biến thể mới vào chương trình, mà không làm ảnh hưởng đến code client hiện tại (✔️ Open/Closed Principle.)

## Builder

Builder là pattern cho phép chúng ta khởi tạo các object phức tạp theo từng bước. bằng cách sử dụng cùng một mã xây dựng, chúng ta có thể tạo ra các loại và cách biểu diễn khác nhau của đối tượng một cách dễ dàng.

### Builder có ưu điểm gì?

- Sử dụng Builder để loại bỏ các “hàm khởi tạo khổng lồ” với quá nhiều tham số, chỉ dùng những bước build đối tượng thực sự cần.
- Sử dụng Builder để tạo ra những cây `Composite` và các đối tượng phức tạp khác, bằng cách bỏ qua một số bước hoặc gọi đệ quy.
- Single Responsibility Principle: tách code khởi tạo phức tạp khỏi logic nghiệp vụ của sản phẩm. Bằng cách sử dụng class `Director`, bạn sẽ ẩn hoàn toàn chi tiết khởi tạo của sản phẩm với code client. Code client chỉ cần liên kết với `Director`, rồi nhận hàm khởi tạo từ director và kết quả từ builder.

```python
"""
Builder Design Pattern
Mục đích: Cho phép khởi tạo một đối tượng phức tạp theo từng bước,
nhằm tạo ra các kiểu và cách thể hiện khác nhau của một đối tượng bằng cách sử dụng cùng một mã xây dựng.
"""
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Any

"""
Xây dựng Product object. Với những builder cụ thể thì có thể tạo ra các đối tượng
không liên quan, hay nói cách khác là có thể không cùng một interface.
"""
class PizzaHut:
    def __init__(self) -> None:
        self.parts = []

    def add(self, part: Any) -> None:
        self.parts.append(part)

    def list_parts(self) -> None:
        print(f"Product parts: {', '.join(self.parts)}", end="")
```

```python
"""
Khởi tạo Builder interface với các methods giúp tạo Product objects theo các bước
"""
class Builder(ABC):
    @property
    @abstractmethod
    def product(self) -> None:
        pass

    @abstractmethod
    def produce_size(self) -> None:
        pass

    @abstractmethod
    def produce_sauce(self) -> None:
        pass

    @abstractmethod
    def produce_cheese(self) -> None:
        pass

    @abstractmethod
    def produce_topping(self) -> None:
        pass
```

```python
class PizzaHutBuilder(Builder):
    def __init__(self) -> None:
        """
        Reset builder trước khi xây dựng Product object
        """
        self.reset()

    def reset(self) -> None:
        self._product = PizzaHut()

    @property
    def product(self) -> PizzaHut:
        """
        Một một builder cụ thể (ở đây là PizzaHutBuilder) sẽ có những methods riêng
        để xây dựng đối tượng. Tại vì mỗi một builder khác nhau sẽ có những methods khác nhau
        Vì vậy, chúng ta không nên khai báo trong interface Builder. 
        Thay vào đó, khi tạo một builder cụ thể chúng ta sẽ thực hiện override các methods này.
        
        * Note: Sau khi trả về kết quả, builder sẽ đc reset để bắt đầu sản xuất một Product (object) khác.
        """
        product = self._product
        self.reset()
        return product

    def produce_size(self) -> None:
        self._product.add("20")

    def produce_sauce(self) -> None:
        self._product.add("Tomato")

    def produce_cheese(self) -> None:
        self._product.add("Ricotta")
    
    def produce_topping(self) -> None:
        self._product.add("Pepperoni")
```

```python
"""
Director sẽ chịu trách nhiệm xây dựng đối tượng theo một trình tự cụ thể.
Tuy nhiên chỉ là Optional vì client có thể điều khiển builder.
"""
class Director:
    def __init__(self) -> None:
        self._builder = None

    @property
    def builder(self) -> Builder:
        return self._builder

    @builder.setter
    def builder(self, builder: Builder) -> None:
        """
        Director vẫn chấp nhận với mọi yêu cầu xây dựng tượng từ client.
        """
        self._builder = builder

    """
    Director hỗ trợ một số methods để xây dựng đối tượng default
    """
    def build_minimal_viable_product(self) -> None:
        # Pizza chỉ có sốt
        self.builder.produce_sauce()

    def build_full_featured_product(self) -> None:
        # Pizza full topping
        self.builder.produce_sauce()
        self.builder.produce_cheese()
        self.builder.produce_topping()
```

```python
if __name__ == "__main__":
    """
    Client code tạo builder object, chuyển cho director để thực hiện xây dựng,
    sau đó trả về kết quả từ builder object.
    """
    director = Director()
    builder = PizzaHutBuilder()
    director.builder = builder

    print("Standard basic product: ")
    director.build_minimal_viable_product()
    builder.product.list_parts()

    print("\n")

    print("Standard full featured product: ")
    director.build_full_featured_product()
    builder.product.list_parts()

    print("\n")

    # Builder pattern vẫn work mà không cần Director.
    print("Custom product: ")
    builder.produce_size()
    builder.produce_cheese()
    builder.produce_sauce()
    builder.product.list_parts()
```

```
Standard basic product: 
Product parts: Tomato

Standard full featured product: 
Product parts: Tomato, Ricotta, Pepperoni

Custom product: 
Product parts: 20, Ricotta, Tomato
```

## Prototype

Prototype là một pattern nhằm mục đích giảm số lượng class được sử dụng cho một ứng dụng. Nó cho phép bạn sao chép các đối tượng hiện có một cách độc lập với việc triển khai cụ thể các lớp của chúng, đặc biệt là khi việc tạo đối tượng là một nhiệm vụ tốn kém về thời gian và tài nguyên và đã tồn tại một đối tượng tương tự. Phương pháp này cung cấp cách sao chép đối tượng ban đầu và sau đó sửa đổi nó theo nhu cầu của chúng ta.

### Vấn đề

Giả sử chúng ta có một object và muốn tạo ra một bản sao của nó, đơn giản nhất là:

- Khởi tạo object mới có cùng lớp
- Lấy giá trị từ tất cả các trường của đối tượng gốc và gán nó sang cho đối tượng mới.

Tuy nhiên không phải tất cả object đều có thể sao chép theo cách này vì có thể một vài trường của nó là private, không thể truy cập từ bên ngoài object.

Chưa kể nữa là code của bạn sẽ trở nên phụ thuộc vào lớp đó (vi phạm `Dependency inversion principle`), và đôi khi bạn chỉ biết interface của đối tượng chứ không biết đến lớp cụ thể, khi đó các tham số trong các method sẽ chấp nhận bất kỳ object nào theo interface đấy.

Để clone một object thì chúng ta có thể sử dụng shadow copy hoặc deep copy.

- `Shadow copy`: chỉ sao chép cấu trúc cao nhất (top-level) của object chứ không tạo ra các bản sao của các object lồng nhau của nó (nested objects), khi đó nested objects sẽ giữ references.
- `Deep copy`: sao chép cấu trúc cao nhất (top-level) và tất cả các nested objects để tạo ra các bản sao hoàn toàn độc lập.

### Ứng dụng

- Database Connection Pooling: sử dụng prototypes để tạo và reuse connection.
- Configuration Handling: sử dụng prototypes để quản lý danh sách các cấu hình, để đảm bảo khi copy sang chạy ở môi trường khác thì ko bị mismatch.
- Machine Learning Initialization: sử dụng prototypes để khởi tạo models, weights pretrained khi fine-tuning, tiết kiệm thời gian training.
- Distributed system: sử dụng prototypes để duy trì tính toàn vẹn và đồng nhất dữ liệu của distributed databases, hay với mô hình microservices thì các service khi scale sẽ sử dụng prototypes để copy các cấu hình hoặc trạng thái để duy trì tính nhất quán và phục vụ các yêu cầu của người dùng.

```python
"""
Prototype pattern
"""
from abc import abstractmethod, ABC

class Person(ABC):
    """
    Prototype abstract
    """
    def __init__(self, name):
        self._name = name

    def set_name(self, name: str):
        self._name = name

    def get_name(self):
        return self._name

    @abstractmethod
    def display(self):
        pass
    
    @abstractmethod
    def clone(self):
        """
        abstract method để clone đối tượng
        """
        pass
```

```python
import copy

"""
Teacher is Concrete Prototype
"""
class Teacher(Person):
    def __init__(self, name, course):
        super().__init__(name)
        self._course = course

    def set_course(self, course: str):
        self._course = course

    def get_course(self):
        return self._course

    def display(self):
        print("Teacher was cloned:")
        print("------------------:")
        print(f"Name: {self._name}")
        print(f"Who Teaches: {self._course}\n")
    
    def clone(self):
        return copy.copy(self)

"""
Student is Concrete Prototype
"""
class Student(Person):
    def __init__(self, name, teacher: Teacher):
        super().__init__(name)
        self._teacher = teacher

    def display(self):
        print("Student was cloned:")
        print("------------------:")
        print(f"Student Name: {self._name}")
        print(f"Enrolled in: {self._teacher.get_course()}")
        print(f"Taught by: {self._teacher.get_name()}\n")
    
    def clone(self):
        return copy.copy(self)
```

```python
"""
Client code
"""
if __name__ == "__main__":
    teacher = Teacher("Ian Goodfellow", "Deep Learning")
    teacher_clone = teacher.clone()
    teacher_clone.display()

    student = Student("Hoang", teacher=teacher_clone)
    student_clone = student.clone()
    student_clone.display()

    # Modify teacher clone
    teacher_clone.set_course("Machine Learning")
    student_clone.display()
```

```
Teacher was cloned:
------------------:
Name: Ian Goodfellow
Who Teaches: Deep Learning

Student was cloned:
------------------:
Student Name: Hoang
Enrolled in: Deep Learning
Taught by: Ian Goodfellow

Student was cloned:
------------------:
Student Name: Hoang
Enrolled in: Machine Learning
Taught by: Ian Goodfellow
```

Do sử dụng shadow copy, nên khi gọi `student.clone()`, nó vẫn giữ reference đến object `teacher_clone` và thông tin display sẽ bị thay đổi theo. Vậy nếu sử dụng deepcopy thì sao?

```python
import copy

"""
Teacher is Concrete Prototype
"""
class Teacher(Person):
    def __init__(self, name, course):
        super().__init__(name)
        self._course = course

    def set_course(self, course: str):
        self._course = course

    def get_course(self):
        return self._course

    def display(self):
        print("Teacher was cloned:")
        print("------------------:")
        print(f"Name: {self._name}")
        print(f"Who Teaches: {self._course}\n")
    
    def clone(self):
        return copy.deepcopy(self)

"""
Student is Concrete Prototype
"""
class Student(Person):
    def __init__(self, name, teacher: Teacher):
        super().__init__(name)
        self._teacher = teacher

    def display(self):
        print("Student was cloned:")
        print("------------------:")
        print(f"Student Name: {self._name}")
        print(f"Enrolled in: {self._teacher.get_course()}")
        print(f"Taught by: {self._teacher.get_name()}\n")
    
    def clone(self):
        return copy.deepcopy(self)
```

```python
"""
Client code
"""
if __name__ == "__main__":
    teacher = Teacher("Ian Goodfellow", "Deep Learning")
    teacher_clone = teacher.clone()
    teacher_clone.display()

    student = Student("Hoang", teacher=teacher_clone)
    student_clone = student.clone()
    student_clone.display()

    # Modify teacher clone
    teacher_clone.set_course("Machine Learning")
    student_clone.display()
```

```
Teacher was cloned:
------------------:
Name: Ian Goodfellow
Who Teaches: Deep Learning

Student was cloned:
------------------:
Student Name: Hoang
Enrolled in: Deep Learning
Taught by: Ian Goodfellow

Student was cloned:
------------------:
Student Name: Hoang
Enrolled in: Deep Learning
Taught by: Ian Goodfellow
```

Thông tin display được giữ nguyên -> deepcopy đã clone ra các bản sao hoàn toàn độc lập.

## Singleton

Singleton pattern cho phép bạn tạo một instance duy nhất của một class trong suốt vòng đời của chương trình, với mục đích:

- Hạn chế chế quyền truy cập đồng thời vào một tài nguyên được chia sẻ (vd: database connections)
- Tạo một điểm truy cập toàn cầu cho một tài nguyên (vd: load balancing)

### Non Thread Safe

```python
class Singleton:
    """
    Implement Singleton với staticmethod
    """
    __instance: dict = None

    @staticmethod
    def get_instance():
        if not Singleton.__instance:
            return Singleton()
        return Singleton.__instance

    def __init__(self):
        if Singleton.__instance:
            raise Exception("This class cannot initialize")
        else:
            Singleton.__instance = self

if __name__ == "__main__":
    s1 = Singleton()
    s2 = Singleton.get_instance()
    s3 = Singleton.get_instance()
    
    assert id(s1) == id(s2)
    assert id(s2) == id(s3)
```

```python
class Singleton:
    """
    Implement Singleton với magic method __new__
    """
    _instance = None

    def __new__(cls):
        """
        Create a new instance of the Singleton class if one does not already exist.
        Returns:
            Singleton: The unique instance of the Singleton class.
        """
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

if __name__ == "__main__":
    s1 = Singleton()
    s2 = Singleton()

    assert id(s1) == id(s2)
```

```python
"""
Implement Singleton với decorator pattern
"""
def singleton(cls):
    """
    A decorator to implement the Singleton design pattern.
    """

    instances = {}  # Dictionary to store instances of different classes

    def get_instance(*args, **kwargs):
        """
        Create a new instance if it doesn't exist, or return the existing one.

        Args:
            *args: Positional arguments for class initialization.
            **kwargs: Keyword arguments for class initialization.

        Returns:
            cls: The unique instance of the class.

        """
        if cls not in instances:
            # Create a new instance and store it
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]  # Return the existing instance
    
    # Return the closure function for class instantiation
    return get_instance  

@singleton  # Applying the singleton decorator
class SingletonClass:
    def __init__(self, data):
        self.data = data

    def display(self):
        print(f"Singleton instance with data: {self.data}")

if __name__ == "__main__":
    s1 = SingletonClass("instance 1")
    s2 = SingletonClass("instance 2")

    assert s1.display() == s2.display()
```

```
Singleton instance with data: instance 1
Singleton instance with data: instance 1
```

```python
class SingletonMeta(type):
    _instances = {}
    """
    Implement Singleton với metaclass
    _instances được khai báo ở đây là class attribute, nó được share (dùng chung) cho các instance khởi tạo từ class này.
    """
    def __call__(cls, *args, **kwargs):
        """
        Possible changes to the value of the `__init__` argument do not affect the returned instance.
        """
        if cls not in cls._instances:
            instance = super().__call__(*args, **kwargs)
            cls._instances[cls] = instance
        return cls._instances[cls]

class Singleton(metaclass=SingletonMeta):
    def some_business_logic(self):
        pass

if __name__ == "__main__":
    s1 = Singleton()
    s2 = Singleton()
    assert id(s1) == id(s2)
```

### Thread Safe

```python
from threading import Lock, Thread

class SingletonMeta(type):
    _instances = {}

    _lock: Lock = Lock()
    """
    We now have a lock object that will be used to synchronize threads during first access to the Singleton.
    """
    def __call__(cls, *args, **kwargs):
        """
        Possible changes to the value of the `__init__` argument do not affect the returned instance.
        """
        # Now, imagine that the program has just been launched.
        # Since there's no Singleton instance yet, multiple threads can
        # simultaneously pass the previous conditional and reach this
        # point almost at the same time. The first of them will acquire
        # lock and will proceed further, while the rest will wait here.

        with cls._lock:
            # The first thread to acquire the lock, reaches this
            # conditional, goes inside and creates the Singleton
            # instance. Once it leaves the lock block, a thread that
            # might have been waiting for the lock release may then
            # enter this section. But since the Singleton field is
            # already initialized, the thread won't create a new
            # object.

            if cls not in cls._instances:
                instance = super().__call__(*args, **kwargs)
                cls._instances[cls] = instance

        return cls._instances[cls]

class Singleton(metaclass=SingletonMeta):
    value: str = None
    """
    We'll use this property to prove that our Singleton really works.
    """
    def __init__(self, value: str) -> None:
        self.value = value

    def some_business_logic(self):
        """
        Finally, any singleton should define some business logic, which can be executed on its instance.
        """

def test_singleton(value: str) -> None:
    singleton = Singleton(value)
    print(singleton.value)

if __name__ == "__main__":
    print("If you see the same value, then singleton was reused (yay!)\n"
          "If you see different values, "
          "then 2 singletons were created (booo!!)\n\n"
          "RESULT:\n")

    process1 = Thread(target=test_singleton, args=("FOO",))
    process2 = Thread(target=test_singleton, args=("BAR",))
    process1.start()
    process2.start()
```

```
If you see the same value, then singleton was reused (yay!)
If you see different values, then 2 singletons were created (booo!!)

RESULT:

FOO
FOO
```

You can get full source code [here](https://github.com/hoangph3/Object-Oriented-Programming/blob/main/DesignPattern/Creational.ipynb)!
