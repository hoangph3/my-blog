---
title: "SOLID Principles in Python"
date: 2024-05-17T23:22:35+07:00
tags: ["OOP", "Python"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Walk through the five SOLID object-oriented design principles with runnable Python examples."
summary: "Walk through the five SOLID object-oriented design principles with runnable Python examples."
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

## Single-responsibility principle (SRP)

Một class chỉ nên thực hiện một công việc. Nếu một class mà có nhiều hơn một công việc thì khi đó chúng ta nên tách ra mỗi class thực hiện một công việc khác nhau.

```python
# file_manager_srp.py

from pathlib import Path
from zipfile import ZipFile

class FileManager:
    def __init__(self, filename):
        self.path = Path(filename)

    def read(self, encoding="utf-8"):
        return self.path.read_text(encoding)

    def write(self, data, encoding="utf-8"):
        self.path.write_text(data, encoding)

    def compress(self):
        with ZipFile(self.path.with_suffix(".zip"), mode="w") as archive:
            archive.write(self.path)

    def decompress(self):
        with ZipFile(self.path.with_suffix(".zip"), mode="r") as archive:
            archive.extractall()
```

Class trên vi phạm nguyên tắc SRP, do nó cho phép thực hiện 2 nhiệm vụ khác nhau:

1. Đọc ghi file.
2. Nén/giải nén file.

Hai nhiệm vụ này độc lập (có thể theo ý kiến chủ quan), không nên để chung trong cùng 1 class mà nên tách ra để dễ quản lý hơn.

```python
# file_manager_srp.py

from pathlib import Path
from zipfile import ZipFile

class FileManager:
    def __init__(self, filename):
        self.path = Path(filename)

    def read(self, encoding="utf-8"):
        return self.path.read_text(encoding)

    def write(self, data, encoding="utf-8"):
        self.path.write_text(data, encoding)

class ZipFileManager:
    def __init__(self, filename):
        self.path = Path(filename)

    def compress(self):
        with ZipFile(self.path.with_suffix(".zip"), mode="w") as archive:
            archive.write(self.path)

    def decompress(self):
        with ZipFile(self.path.with_suffix(".zip"), mode="r") as archive:
            archive.extractall()
```

*** Note:

Class thực hiện một nhiệm vụ duy nhất không nhất thiết phải chỉ có chứa 1 method duy nhất, mà quan trọng là nhiệm vụ cốt lõi của nó.

Như ở trên, class FileManager đảm nhiệm việc quản lý tệp, trong khi ZipFileManager xử lý việc nén và giải nén tệp với định dạng ZIP.

Tại sao phải cần SRP?

Giả sử code chúng ta có 5 module khác nhau, trong đó 4 module cần quản lý file (đọc ghi), 1 module còn lại quản lý nén/giải nén file ZIP.
Sau một thời gian chạy, ta thấy module quản lý nén/giải nén file có bug -> cần thực hiện fix.

- Nếu không áp dụng SRP, ta sẽ phải upcode class FileManager và đồng bộ việc upcode này cho cả 5 module đang sử dụng nó.
- Nếu áp dụng SRP, ta chỉ cần upcode mỗi class ZipFileManager cho 1 module, 4 module kia chỉ đọc ghi file nên không cần phải tác động gì.

Tuy nhiên, không phải lúc nào cũng nên áp dụng nguyên tắc này vào code. Một trường hợp hay gặp trong thực tế là các class dạng Helper hay Utilities, đều vi phạm SRP. Nếu số lượng hàm ít, ta vẫn có thể cho tất cả các hàm này vào 1 class, nhưng cũng cần lưu ý rằng khi Helper có thêm nhiều chức năng, thì ta nên tách Helper thành các Helper nhỏ hơn, ví dụ: UserHelper, DatabaseHelper, DataHelper, …

## Open-Closed Principle (OCP)

Các entities (classes, modules, functions) phải “open” để mở rộng, nhưng phải “close” để sửa đổi.

Ví dụ chúng ta nên mở rộng class (dùng kế thừa) thay vì sửa đổi class gốc, tại vì việc sửa đổi trên class cũ sẽ có thể gây ra bug khi các module khác đang sử dụng class cũ.

```python
class Animal:
    def __init__(self, name: str):
        self.name = name
    
    def get_name(self) -> str:
        pass

animals = [
    Animal('lion'),
    Animal('mouse')
]

def animal_sound(animals: list):
    for animal in animals:
        if animal.name == 'lion':
            print('roar')

        elif animal.name == 'mouse':
            print('squeak')

animal_sound(animals)
```

```
roar
squeak
```

Hàm `animal_sound` không tuân theo nguyên tắc OCP, vì nó không thể mở rộng (open) đối với các loại động vật mới.

Dễ thấy là cứ thêm một động vật mới thì chúng ta lại phải đi sửa đổi function gốc, logic sẽ trở nên ngày càng phức tạp với nhiều câu lệnh if else lặp đi lặp lại. Vậy áp dụng OCP như thế nào?

```python
class Animal:
    def __init__(self, name: str):
        self.name = name
    
    def get_name(self) -> str:
        pass

    def make_sound(self):
        pass

class Lion(Animal):
    def make_sound(self):
        return 'roar'

class Mouse(Animal):
    def make_sound(self):
        return 'squeak'

class Snake(Animal):
    def make_sound(self):
        return 'hiss'

def animal_sound(animals: list):
    for animal in animals:
        print(animal.make_sound())

animals = [
    Lion('lion'),
    Mouse('mouse'),
    Snake('snake')
]
animal_sound(animals)
```

```
roar
squeak
hiss
```

Bằng cách áp dụng nguyên tắc OCP, mọi loại con vật đều có method `make_sound`, được mở rộng từ class `Animal`.

Bây giờ nếu có thêm một con vật mới, tất cả những gì chúng ta cần làm là thêm con vật mới vào mảng.

Hãy xem một ví dụ khác:

```python
class Discount:
    def __init__(self, customer, price):
        self.customer = customer
        self.price = price

    def give_discount(self):
        if self.customer == 'fav':
            return self.price * 0.2
        if self.customer == 'vip':
            return self.price * 0.4
```

Method `give_discount` sẽ cần phải update nếu như bạn có thêm nhiều đối tượng khách hàng mới, điều này vi phạm nguyên tắc OCP

Thay vào đó, chúng ta sẽ mở rộng bằng cách tạo class mới với method mới.

```python
class Discount:
    def __init__(self, customer, price):
        self.customer = customer
        self.price = price

    def get_discount(self):
        # discount 20%
        return self.price * 0.2

class VIPDiscount(Discount):
    def get_discount(self):
        # discount 40%
        return super().get_discount() * 2

class SuperVIPDiscount(VIPDiscount):
    def get_discount(self):
        # discount 80%
        return super().get_discount() * 2
```

Việc áp dụng nguyên tắc OCP sẽ phát sinh thêm nhiều class. Tuy nhiên ta sẽ không cần phải test lại các class cũ, thay vào đó chỉ cần test các class mới với các method mới.

Nếu sửa trực tiếp vào class hay method cũ, ta sẽ cần phải test lại cả những logic cũ để xem sau khi thêm code mới thì nó có hoạt động như cũ không -> tốn thời gian, có thể gây ra bug.

## Liskov Substitution Principle (LSP)

Nguyên tắc này nói đến việc các class con có thể thay thế class cha (base) mà chương trình vẫn hoạt động đúng.

```python
# shapes_lsp.py

class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def calculate_area(self):
        print(self.__dict__)
        return self.width * self.height

class Square(Rectangle):
    def __init__(self, side):
        super().__init__(side, side)

    def __setattr__(self, key, value):
        super().__setattr__(key, value)
        if key in ("width", "height"):
            self.__dict__["width"] = value
            self.__dict__["height"] = value
```

Trong toán học, hình vuông là trường hợp đặc biệt của hình chữ nhật. Do đó, với class `Square`, ta có thể kế thừa class base `Rectangle` như trên, với hai thuộc tính width và height đều bằng side.

```python
rec = Rectangle(width=2, height=3)
print(rec.calculate_area())

square = Square(side=4)
print(square.calculate_area())
```

```
{'width': 2, 'height': 3}
6
{'width': 4, 'height': 4}
16
```

Đoạn code trên không có gì sai, tuy nhiên nó vi phạm nguyên tắc LSP, khi mà chúng ta không thể thay thế `Rectangle` instance bằng những `Square` instance khác.

Trong lập trình, tránh bê nguyên các mối quan hệ của các object ngoài đời sống vào code. Mặc dù hình vuông là trường hợp đặc biệt của hình chữ nhật trong toán học, tuy nhiên ở đây chúng không nên có quan hệ cha-con mà chỉ nên là quan hệ anh-em (kế thừa từ một class base khác và đều có method `calculate_area`) như sau:

```python
# shapes_lsp.py

from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def calculate_area(self):
        pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def calculate_area(self):
        return self.width * self.height

class Square(Shape):
    def __init__(self, side):
        self.side = side

    def calculate_area(self):
        return self.side ** 2
```

Bằng cách áp dụng LSP, ta không cần quan tâm đối tượng thuộc loại nào, miễn là nó có method `calculate_area` thì code vẫn sẽ hoạt động.

```python
def get_total_area(shapes):
    return sum(shape.calculate_area() for shape in shapes)

get_total_area([Rectangle(10, 5), Square(5)])
```

```
75
```

## Interface Segregation Principle (ISP)

Nguyên tắc này nói đến việc không nên ép buộc các class phải có các methods mà nó ko cần dùng đến.

Nói cách khác, nếu một class không sử dụng các attributes hay methods (gọi chung là interfaces) thì các methods và attributes đó sẽ được tách riêng thành các class cụ thể hơn.

```python
"""
Base class
"""
class IShape:
    def draw_square(self):
        raise NotImplementedError
    
    def draw_rectangle(self):
        raise NotImplementedError
    
    def draw_circle(self):
        raise NotImplementedError

"""
Subclasses
"""
class Circle(IShape):
    def draw_square(self):
        pass

    def draw_rectangle(self):
        pass
    
    def draw_circle(self):
        pass

class Square(IShape):
    def draw_square(self):
        pass

    def draw_rectangle(self):
        pass
    
    def draw_circle(self):
        pass

class Rectangle(IShape):
    def draw_square(self):
        pass

    def draw_rectangle(self):
        pass
    
    def draw_circle(self):
        pass
```

Khi triển khai như trên, class `Rectangle` chứa các methods (`draw_circle` và `draw_square`) mà nó không dùng đến, tương tự với class `Square` với các methods `draw_circle`, `draw_ectangle`, và class `Circle` với các methods `draw_square`, `draw_rectangle`.

Để đáp ứng nguyên tắc ISP, ta sẽ sửa lại như sau:

```python
class IShape:
    def draw(self):
        raise NotImplementedError

class Circle(IShape):
    def draw(self):
        pass

class Square(IShape):
    def draw(self):
        pass

class Rectangle(IShape):
    def draw(self):
        pass
```

Ví dụ với trường hợp đa kế thừa:

```python
# printers_isp.py
from abc import ABC, abstractmethod

"""
Base classes
"""
class Printer(ABC):
    @abstractmethod
    def print(self, document):
        pass

class Fax(ABC):
    @abstractmethod
    def fax(self, document):
        pass

class Scanner(ABC):
    @abstractmethod
    def scan(self, document):
        pass

"""
Subclasses
"""
class OldPrinter(Printer):
    def print(self, document):
        print(f"Printing {document} in black and white...")

class NewPrinter(Printer, Fax, Scanner):
    def print(self, document):
        print(f"Printing {document} in color...")

    def fax(self, document):
        print(f"Faxing {document}...")

    def scan(self, document):
        print(f"Scanning {document}...")
```

Với triển khai như trên, ta đã tách biệt được các interfaces (Interface Segregation), bây giờ `Printer`, `Fax` và `Scanner` là các class base cung cấp các interfaces thỏa mãn nguyên tắc SRP (chỉ thực hiện một nhiệm vụ duy nhất).

Như vậy với `OldPrinter` chỉ cần kết thừa interface `Printer`, trong khi đó `NewPrinter` sẽ đa kế thừa từ các interfaces `Printer`, `Fax` và `Scanner` để có đủ các methods cần thiết.

## Dependency Inversion Principle (DIP)

Nguyên tắc này yêu cầu hai việc:

- Các mô-đun cấp cao không nên phụ thuộc vào các mô-đun cấp thấp. Cả hai nên phụ thuộc vào abstractions.
- Abstractions không nên phụ thuộc vào chi tiết (details), chi tiết nên phụ thuộc vào abstractions.

Sẽ có những thời điểm trong quá trình phát triển, ứng dụng của chúng ta sẽ có nhiều rất module, các module cấp cao phụ thuộc vào các module cấp thấp để chạy. Hãy xem ví dụ sau:

```python
class FrontEnd:
    def __init__(self, back_end):
        self.back_end = back_end

    def display_data(self):
        data = self.back_end.get_data_from_database()
        print("Display data:", data)

class BackEnd:
    def get_data_from_database(self):
        return "Data from the database"
```

Trong ví dụ trên, class `Frontend` bị phụ thuộc vào cách triển khai cụ thể của class `Backend`, chính sự liên kết chặt chẽ này khiến nó khó mở rộng.

Giả sử bây giờ chúng ta cần đọc dữ liệu từ một nguồn khác như REST API chẳng hạn, thì sẽ triển khai thế nào?

- Thêm method mới `get_data_from_api` trong class `Backend`
- Sửa class `Frontend` để có thể đọc dữ liệu từ method `get_data_from_api` -> vi phạm open-closed principle.

Bằng cách áp dụng DIP, ta sẽ làm cho class `Frontend` phụ thuộc vào abstractions, thay vì phụ thuộc vào class `Backend` được triển khai cụ thể.

```python
from abc import ABC, abstractmethod

class FrontEnd:
    def __init__(self, data_source):
        self.data_source = data_source

    def display_data(self):
        data = self.data_source.get_data()
        return f"Display data: {data}"

class DataSource(ABC):
    @abstractmethod
    def get_data(self):
        pass

class Database(DataSource):
    def get_data(self):
        return "Data from the database"

class API(DataSource):
    def get_data(self):
        return "Data from the API"

# Run
db_front_end = FrontEnd(Database())
print(db_front_end.display_data())

api_front_end = FrontEnd(API())
print(api_front_end.display_data())
```

```
Display data: Data from the database
Display data: Data from the API
```

Thêm một ví dụ khác:

```python
class IFood:
    def bake(self):
        raise NotImplemented

    def eat(self):
        raise NotImplemented

class Pizza(IFood):
    def bake(self):
        print("pizza was baked")

    def eat(self):
        print("pizza was ate")

class Bread(IFood):
    def bake(self):
        print("bread was baked")

    def eat(self):
        print("bread was ate")

class Production:
    def __init__(self, food: IFood):
        self.food = food

    def produce(self):
        self.food.bake()

    def consume(self):
        self.food.eat()

if __name__ == '__main__':
    pizza = Pizza()
    bread = Bread()
    
    p = Production(pizza)
    p.produce()
    p.consume()
    
    b = Production(bread)
    b.produce()
    b.consume()
```

```
pizza was baked
pizza was ate
bread was baked
bread was ate
```

Ở đây ta có các module cấp thấp là `Bread` và `Pizza`, module cấp cao là `Production`. 2 module này giao tiếp với nhau bằng interface `IFood`, giúp cho chương trình trở lên linh hoạt hơn. Module `Production` chỉ cần sử dụng các method trong `IFood` mà không bị ràng buộc hay cần quan tâm object nào sẽ được truyền vào. Ta có thể truyền vào pizza hoặc bread.

You can get full source code [here](https://github.com/hoangph3/Object-Oriented-Programming/blob/main/SOLID/SOLID.ipynb)!
