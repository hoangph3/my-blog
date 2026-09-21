---
title: "Behavioral Design Patterns in Python"
date: 2024-05-19T21:39:26+07:00
tags: ["OOP", "Python", "Design Patterns"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Implement behavioral design patterns such as Strategy, Observer and Command in Python."
summary: "Implement behavioral design patterns such as Strategy, Observer and Command in Python."
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

## Chain of Responsibility

Trong thực tế, một hệ thống thường có rất nhiều giai đoạn xử lý phải xử lý các yêu cầu gửi đến. Mỗi giai đoạn có một trách nhiệm cụ thể và thứ tự thực hiện các giai đoạn này là rất quan trọng. Đó chính là lý do mà chúng ta có thể sử dụng Chain of Responsibility, nó cho phép chúng ta xử lý các yêu cầu theo một chuỗi (chain) công việc cụ thể.

Chuỗi công việc ở đây không đơn thuần chỉ là một chuỗi thực hiện tuần tự (Basic Chain), mà nó có thể có một số biến thể khác như:

- Bidirectional Chain: forward and backward directions.
- Hierarchical Chain: tổ chức dưới dạng hierarchical structure, có chứa các sub-chains khác.
- Dynamic Chain: chain có thể thay đổi tùy thuộc vào kết quả lúc chạy chương trình, điều kiện, …

Một số hệ thống sử dụng Chain of Responsibility như:

- Middleware trong web frameworks
- Workflow Automation
- Financial Transaction Processing
- Data Transformation Pipelines

Ví dụ sau đây triển khai Middleware trong web frameworks, áp dụng Basic Chain:

```python
from abc import ABC, abstractmethod

# Section 1: Abstract Middleware Base
class Middleware(ABC):
    """Abstract Middleware class serving as the base of the chain."""
    @abstractmethod
    def handle_request(self, request):
        """Handle the request; must be implemented by concrete classes."""
        pass

# Section 2: Concrete Middleware Implementations
class AuthenticationMiddleware(Middleware):
    """Middleware responsible for user authentication."""
    def handle_request(self, request):
        """Handle authentication or pass to the next middleware in the chain."""
        if self.authenticate(request):
            print("Authentication middleware: Authenticated successfully")
            # Pass the request to the next middleware or handler in the chain.
            return super().handle_request(request)
        else:
            print("Authentication middleware: Authentication failed")
            # Stop the chain if authentication fails.
            return None

    def authenticate(self, request):
        """Implement authentication logic here."""
        # Return True if authentication is successful, else False.
        pass

class LoggingMiddleware(Middleware):
    """Middleware responsible for logging requests."""
    def handle_request(self, request):
        """Handle request logging and pass to the next middleware in the chain."""
        print("Logging middleware: Logging request")
        # Pass the request to the next middleware or handler in the chain.
        return super().handle_request(request)

class DataValidationMiddleware(Middleware):
    """Middleware responsible for data validation."""
    def handle_request(self, request):
        """Handle data validation or pass to the next middleware in the chain."""
        if self.validate_data(request):
            print("Data Validation middleware: Data is valid")
            # Pass the request to the next middleware or handler in the chain.
            return super().handle_request(request)
        else:
            print("Data Validation middleware: Invalid data")
            # Stop the chain if data validation fails.
            return None

    def validate_data(self, request):
        """Implement data validation logic here."""
        # Return True if data is valid, else False.
        pass
```

```python
# Section 3: Request Handling Class and Client Code
# Chain class to handle the final request and manage middleware.
class Chain:
    def __init__(self):
        self.middlewares = []

    def add_middleware(self, middleware):
        self.middlewares.append(middleware)

    def handle_request(self, request):
        for middleware in self.middlewares:
            request = middleware.handle_request(request)
            if request is None:
                print("Request processing stopped.")
                break

# Client code to create and configure the middleware chain.
if __name__ == "__main__":
    # Create middleware instances.
    auth_middleware = AuthenticationMiddleware()
    logging_middleware = LoggingMiddleware()
    data_validation_middleware = DataValidationMiddleware()

    # Create the chain and add middleware.
    chain = Chain()
    chain.add_middleware(auth_middleware)
    chain.add_middleware(logging_middleware)
    chain.add_middleware(data_validation_middleware)

    # Simulate an HTTP request.
    http_request = {"user": "username", "data": "valid_data"}
    chain.handle_request(http_request)
```

```
Authentication middleware: Authentication failed
Request processing stopped.
```

## Command

Command Method cho phép bạn đóng gói các yêu cầu hoặc hoạt động vào một đối tượng riêng biệt, cho phép bạn thực hiện các thao tác như di chuyển, hoặc gửi các yêu cầu một cách dễ dàng mà không cần biết chi tiết về thực hiện cụ thể của nó.

```python
from abc import ABC, abstractmethod

# Command interface
class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

# Receiver
class Light:
    def turn_on(self):
        print("Light is on")

    def turn_off(self):
        print("Light is off")

# Concrete commands
class TurnOnCommand(Command):
    def __init__(self, light: Light):
        self.light = light

    def execute(self):
        self.light.turn_on()

class TurnOffCommand(Command):
    def __init__(self, light: Light):
        self.light = light

    def execute(self):
        self.light.turn_off()

# Invoker
class RemoteControl:
    def __init__(self):
        self.command = None

    def set_command(self, command: Command):
        self.command = command

    def press_button(self):
        self.command.execute()

# Client code
if __name__ == "__main__":
    light = Light()
    turn_on = TurnOnCommand(light)
    turn_off = TurnOffCommand(light)

    remote = RemoteControl()
    remote.set_command(turn_on)
    remote.press_button()  # Output: Light is on

    remote.set_command(turn_off)
    remote.press_button()  # Output: Light is off
```

```
Light is on
Light is off
```

Trong đoạn code trên:

- `Command` là một interface, chỉ có method `execute()` để thực thi command.
- Các lớp cụ thể như `TurnOnCommand` và `TurnOffCommand` này là các Concrete Command để thực hiện các hành động cụ thể, được tạo từ Client.
- `RemoteControl` có nhiệm vụ kích hoạt các hành động (Invoker: nhận các commands được client giao cho).
- `Light` có nhiệm vụ nhận và thực thi các hành động (Receiver: chứa một số logic nghiệp vụ).

## Iterator

Iterator là giúp chúng ta duyệt phần tử của một tập hợp mà không để lộ dạng cơ bản của nó (list, stack, tree, …)

```python
class Iterator:
    """Step 1: Create the Iterator interface."""
    def __iter__(self):
        """Defines the __iter__() method to return self as an iterator."""
        raise NotImplementedError

    def __next__(self):
        """Defines the __next__() method to retrieve elements sequentially."""
        raise NotImplementedError

class MyIterator(Iterator):
    """Step 2: Implement a Concrete Iterator."""
    def __init__(self, data):
        self.data = data
        self.index = 0

    def __iter__(self):
        return self  # Returning self as an iterator

    def __next__(self):
        if self.index >= len(self.data):
            raise StopIteration
        value = self.data[self.index]
        self.index += 1
        return value

class Collection:
    """Step 3: Create the Collection interface."""
    def create_iterator(self):
        """Method to create an Iterator compatible with the collection."""
        raise NotImplementedError

class MyCollection(Collection):
    """Step 4: Implement a Concrete Collection."""
    def __init__(self):
        self.data = []

    def add(self, value):
        self.data.append(value)

    def create_iterator(self):
        return MyIterator(self.data)

# Client code
if __name__ == "__main__":
    """Demonstrate the usage of the implemented Iterator Pattern."""
    # Creating a collection
    my_collection = MyCollection()
    my_collection.add(1)
    my_collection.add(2)
    my_collection.add(3)

    # Creating an iterator for the collection
    my_iterator = my_collection.create_iterator()

    # Using the iterator to traverse through the elements
    for element in my_iterator:
        print(element)
```

```
1
2
3
```

## Mediator

Mediator pattern giúp chúng ta hạn chế việc giao tiếp trực tiếp giữa các đối tượng, buộc nó giao tiếp thông qua đối tượng mediator. Khi đó, bạn có thể:

- Quản lý giao tiếp: các đối tượng muốn trao đổi với nhau sẽ phải thông qua một người trung gian, đó chính là mediator.
- Giảm sự phụ thuộc: do các đối tượng chỉ giao tiếp qua trung gian là mediator nên chúng nó sẽ không quá gắn bó và phụ thuộc nhau, giúp việc thay đổi trở nên dễ dàng hơn.

Tuy nhiên cũng cần chú ý rằng, mediator chịu trách nhiệm giao tiếp giữa các đối tượng với nhau nên dần theo thời gian nó sẽ đảm nhiệm rất nhiều nhiệm vụ và trở thành God object (một trong những anti-pattern). Khi đó chúng ta có thể tách nhỏ mediator này thành các mediator nhỏ hơn.

Trong thực tế, Message Broker hoạt động như một Mediator, nó sẽ đóng vai trò làm trung gian, cho phép các dịch vụ của hệ thống gửi và nhận message mà không cần biết và kết nối trực tiếp với nhau. Hãy xem đoạn code sau:

```python
from abc import ABC, abstractmethod

class Participant:
    """
    Represents a message participant.
    """
    def __init__(self, mediator, name):
        self._mediator = mediator
        self.name = name

    def send_message(self, message):
        """
        Sends a message through the mediator.
        """
        self._mediator.send_message(message, self)

    def receive_message(self, message):
        """
        Receives and processes messages from the mediator.
        """
        print(f"{self.name} received message: {message}")

class MessageBroker(ABC):
    """
    Mediator interface (Message Broker) declares message handling methods.
    """
    @abstractmethod
    def send_message(self, message, participant):
        """
        Sends a message to a participant.
        """
        pass

class ConcreteMessageBroker(MessageBroker):
    """
    Concrete Message Broker manages message passing between participants.
    """
    def __init__(self):
        self._participants = []

    def add_participant(self, participant):
        """
        Adds a participant to the broker.
        """
        self._participants.append(participant)

    def send_message(self, message, participant):
        """
        Sends a message to all participants except the sender.
        """
        for p in self._participants:
            # if p != participant:
            p.receive_message(message)

# Client code
if __name__ == "__main__":
    # Create message broker
    message_broker = ConcreteMessageBroker()

    # Create participants and link them to the broker
    participant1 = Participant(message_broker, "Participant 1")
    participant2 = Participant(message_broker, "Participant 2")

    # Add participants to the broker
    message_broker.add_participant(participant1)
    message_broker.add_participant(participant2)

    # Send messages through participants
    participant1.send_message("Hello from Participant 1")
    print()
    participant2.send_message("Hi from Participant 2")
```

```
Participant 1 received message: Hello from Participant 1
Participant 2 received message: Hello from Participant 1

Participant 1 received message: Hi from Participant 2
Participant 2 received message: Hi from Participant 2
```

## Memento

Memento pattern được sử dụng trong việc lưu trữ trạng thái của một đối tượng để sau này có thể khôi phục lại trạng thái đó mà không làm thay đổi cấu trúc của đối tượng đó.

Trong Memento pattern, có ba thành phần chính:

- Originator (Nguyên tác): Đây là đối tượng mà trạng thái cần được lưu trữ. Nó có thể tạo ra một memento để lưu trữ trạng thái hiện tại của nó và cung cấp một phương thức để khôi phục lại trạng thái từ memento.
- Memento (Bản ghi nhớ): Là một đối tượng chứa trạng thái của nguyên tác tại một thời điểm cụ thể. Memento không nên bị sửa đổi từ bên ngoài, chỉ nguyên tác mới có quyền truy cập vào nó.
- Caretaker: Là đối tượng được sử dụng để quản lý và duy trì các memento. Nó không biết về cấu trúc nội bộ của memento, chỉ biết cách lưu trữ và khôi phục lại chúng.

Memento được ứng dụng trong rất nhiều phần mềm như:

- Git: lưu trữ commit, branch, cho phép checkout version, …
- Text editor: lưu trữ các action, cho phép undo, …
- Database: cho phép rollback transaction, …

```python
class Memento:
    def __init__(self, state):
        self._state = state

    def get_state(self):
        return self._state

class Originator:
    _state = None

    def set_state(self, state):
        print(f"Setting state to {state}")
        self._state = state

    def save_to_memento(self):
        print("Saving state to Memento")
        return Memento(self._state)

    def restore_from_memento(self, memento):
        self._state = memento.get_state()
        print(f"Restored state to {self._state}")

class Caretaker:
    _mementos = []

    def add_memento(self, memento):
        print("Adding Memento to the list")
        self._mementos.append(memento)

    def get_memento(self, index):
        print("Getting Memento from the list")
        return self._mementos[index]

# Client code
if __name__ == "__main__":
    originator = Originator()
    caretaker = Caretaker()
    
    # Thiết lập trạng thái ban đầu của originator
    originator.set_state("State 1")
    # Lưu trạng thái hiện tại vào memento và thêm vào caretaker
    caretaker.add_memento(originator.save_to_memento())
    
    # Thiết lập trạng thái mới cho originator
    originator.set_state("State 2")
    # Lưu trạng thái mới vào một memento mới và thêm vào caretaker
    caretaker.add_memento(originator.save_to_memento())
    
    # Khôi phục trạng thái trước đó
    originator.restore_from_memento(caretaker.get_memento(0))  # Output: "State 1"
```

```
Setting state to State 1
Saving state to Memento
Adding Memento to the list
Setting state to State 2
Saving state to Memento
Adding Memento to the list
Getting Memento from the list
Restored state to State 1
```

## Observer

Observer pattern cho phép nhiều đối tượng nhận được cập nhật khi có thay đổi xảy ra ở đối tượng khác mà chúng đang quan sát. Trong thực tế ví dụ như Youtube, kênh sẽ thông báo cho các subscribers khi có video mới, live stream, …

Observer khá giống với Mediator, đều cố gắng tạo ra sự giao tiếp một cách gián tiếp giữa các đối tượng. Trong khi Mediator loại bỏ các kết nối trực tiếp giữa các thành phần, thiết lập một mediator duy nhất làm trung gian, thì Observer linh hoạt hơn khi cho phép chủ động subscribe hoặc unsubscribe người nhận.

Ví dụ sau đây triển khai một ChatRoom, tất cả mọi người cùng join vào một room (group) sẽ nhận được thông báo khi có message từ một người bất kỳ gửi đến.

```python
from abc import ABC, abstractmethod

# Step 1: The ChatRoom - Publisher
class ChatRoom:
    def __init__(self):
        self._participants = set()

    def join(self, participant):
        """Adds a new participant to the chat room."""
        self._participants.add(participant)

    def leave(self, participant):
        """Removes a participant from the chat room."""
        self._participants.remove(participant)

    def broadcast(self, message):
        """Sends a message to all participants in the chat room."""
        for participant in self._participants:
            participant.receive(message)

# Step 2: Participant - Subscriber Interface
class Participant(ABC):
    @abstractmethod
    def receive(self, message):
        """Abstract method for receiving messages."""
        pass

# Step [3]: ChatMember - Concrete Subscribers
class ChatMember(Participant):
    def __init__(self, name):
        self.name = name

    def receive(self, message):
        """Receives and displays the message."""
        print(f"{self.name} received: {message}")

# Step [4]: Client
if __name__ == "__main__":
    # Create a chat room
    general_chat = ChatRoom()

    # Create participants
    user1 = ChatMember("User1")
    user2 = ChatMember("User2")
    user3 = ChatMember("User3")

    # Participants join the chat room
    general_chat.join(user1)
    general_chat.join(user2)
    general_chat.join(user3)

    # Send a message to the chat room
    general_chat.broadcast("Welcome to the chat!")
```

```
User1 received: Welcome to the chat!
User3 received: Welcome to the chat!
User2 received: Welcome to the chat!
```

## State

State pattern giúp chúng ta quản lý hành vi của đối tượng khi trạng thái của nó thay đổi.

Trong thực tế, tại bất kỳ thời điểm nào cũng có một hữu hạn trạng thái (state) mà chương trình có thể có. Với từng trạng thái cụ thể, chương trình sẽ có các hành vi khác nhau, nó có thể chuyển từ trạng thái này sang trạng thái khác ngay lập tức, hoặc giữ nguyên trạng thái đó, … Quy luật chuyển đổi này gọi là transitions, nó hữu hạn và có thể định trước.

Cách đơn giản nhất mà chúng ta có thể nghĩ ra ngay đó chính là xử lý logic nghiệp vụ cho từng trạng thái (if-else) trong một class duy nhất, tuy nhiên khi logic nghiệp vụ phát sinh thêm khiến số lượng trạng thái và quá trình chuyển đổi tăng lên thì câu lệnh if-else rất khó bảo trì và dễ mắc lỗi. Đó chính là lý do mà chúng ta nên sử dụng State pattern, giúp đóng gói các trạng thái này trong các đối tượng riêng biệt và chuyển đổi giữa chúng một cách liền mạch.

Trong State pattern, có ba thành phần chính:

- Context: Là đối tượng chứa trạng thái hiện tại và gọi các method tương ứng với trạng thái đó.
- State (Interface): lớp base chứa các method chung cho tất cả các trạng thái.
- Concrete States (Trạng thái cụ thể): triển khai các method để thực hiện các hành vi cụ thể cho mỗi trạng thái.

Trong ví dụ sau, hãy triển khai State pattern cho hệ thống thanh toán thương mại điện tử.

```python
# Step 1: Define the State Interface
class CheckoutState:
    """Interface defining the methods for various checkout states."""

    def add_item(self, item):
        """Add an item to the cart."""
        pass

    def review_cart(self):
        """Review the cart contents."""
        pass

    def enter_shipping_info(self, info):
        """Enter shipping information."""
        pass

    def process_payment(self):
        """Process the payment."""
        pass

"""
Triển khai các State cụ thể, ở đây chúng ta có 4 trạng thái cụ thể, thể hiện các giai đoạn khác nhau khi thanh toán:
- EmptyCartState: giỏ hàng trống
- ItemAddedState: thêm sản phẩm vào giỏ
- CartReviewedState: Check lại giỏ hàng
- ShippingInfoEnteredState: thêm thông tin vận chuyển
Với mỗi trạng thái sẽ triển khai hành vi cụ thể của nó.
"""
# Step 2: Create Concrete State Classes
class EmptyCartState(CheckoutState):
    """Concrete state representing an empty cart."""

    def add_item(self, item):
        """Add an item to the cart and transition to ItemAddedState."""
        print("Item added to the cart.")
        return ItemAddedState()

    def review_cart(self):
        """Display inability to review an empty cart."""
        print("Cannot review an empty cart.")

    def enter_shipping_info(self, info):
        """Display inability to enter shipping info with an empty cart."""
        print("Cannot enter shipping info with an empty cart.")

    def process_payment(self):
        """Display inability to process payment with an empty cart."""
        print("Cannot process payment with an empty cart.")

class ItemAddedState(CheckoutState):
    """Concrete state representing a cart with added items."""

    def add_item(self, item):
        """Add an item to the cart."""
        print("Item added to the cart.")

    def review_cart(self):
        """Review cart contents and transition to CartReviewedState."""
        print("Reviewing cart contents.")
        return CartReviewedState()

    def enter_shipping_info(self, info):
        """Display inability to enter shipping info without reviewing the cart."""
        print("Cannot enter shipping info without reviewing the cart.")

    def process_payment(self):
        """Display inability to process payment without entering shipping info."""
        print("Cannot process payment without entering shipping info.")

class CartReviewedState(CheckoutState):
    """Concrete state representing a reviewed cart."""

    def add_item(self, item):
        """Display inability to add items after reviewing the cart."""
        print("Cannot add items after reviewing the cart.")

    def review_cart(self):
        """Display that the cart has already been reviewed."""
        print("Cart already reviewed.")

    def enter_shipping_info(self, info):
        """Enter shipping information and transition to ShippingInfoEnteredState."""
        print("Entering shipping information.")
        return ShippingInfoEnteredState(info)

    def process_payment(self):
        """Display inability to process payment without entering shipping info."""
        print("Cannot process payment without entering shipping info.")

class ShippingInfoEnteredState(CheckoutState):
    """Concrete state representing the entry of shipping information."""
    def __init__(self, info):
        self.info = info

    def add_item(self, item):
        """Display inability to add items after entering shipping info."""
        print("Cannot add items after entering shipping info.")

    def review_cart(self):
        """Display inability to review cart after entering shipping info."""
        print("Cannot review cart after entering shipping info.")

    def enter_shipping_info(self, info):
        """Display that shipping information has already been entered."""
        print("Shipping information already entered.")

    def process_payment(self):
        """Process payment with the entered shipping info."""
        print("Processing payment with the entered shipping info.")
```

```python
"""
CheckoutContext: class đóng vai trò là người quản lý quá trình thanh toán.
- Theo dõi trạng thái hiện tại.
- Thực hiện hành vi của trạng thái.
- Đảm bảo sự chuyển đổi liền mạch giữa các trạng thái.
"""
# Step 3: Create the Context Class
class CheckoutContext:
    """Context class managing the checkout states."""

    def __init__(self):
        self.current_state = EmptyCartState()

    def add_item(self, item):
        """Add an item to the cart, updating the current state."""
        self.current_state = self.current_state.add_item(item)

    def review_cart(self):
        """Review the cart, updating the current state."""
        self.current_state = self.current_state.review_cart()

    def enter_shipping_info(self, info):
        """Enter shipping information, updating the current state."""
        self.current_state = self.current_state.enter_shipping_info(info)

    def process_payment(self):
        """Process the payment, updating the current state."""
        self.current_state.process_payment()

# Step 4: Example of Usage
if __name__ == "__main__":
    cart = CheckoutContext()

    cart.add_item("Product 1")
    cart.review_cart()
    cart.enter_shipping_info("123 Main St, City")
    cart.process_payment()
```

```
Item added to the cart.
Reviewing cart contents.
Entering shipping information.
Processing payment with the entered shipping info.
```

## Strategy

Strategy pattern giúp chúng ta gom nhóm các thuật toán mà có thể thay thế cho nhau, cho phép phía client có thể chuyển đổi thuật toán một cách linh hoạt (Dynamic Algorithm Switching).

### Ứng dụng:

- Hệ thống đặt xe: ví dụ bạn muốn đến sân bay, bạn có thể bắt xe máy, xe 4 chỗ, xe 7 chỗ. Các phương tiện của bạn ở đây là strategy. Bạn có thể chọn một trong các strategy dựa vào các nhân tố như ví tiền hay thời gian.
- Hệ thống giao dịch: khi chúng ta muốn thực hiện một giao dịch thì chắc hẳn chúng ta cũng đã thực hiện một số chiến lược (strategy) nào đó để đưa ra quyết định, ví dụ: Moving Average, Mean Reversion, … Chúng ta có thể chọn một trong các Strategy này dựa vào các nhân tố như ví tiền, đồ thị xu hướng, giá cả, …
- Chính sách giảm giá của các cửa hàng: bất kỳ chính sách nào cũng phải có chiến lược (strategy), ví dụ: discount X%, discount X% + quà tặng, discount khi mua với số lượng lớn, tặng phiếu mua hàng lần tiếp theo, …

Một vài usecase sử dụng trong thực tế:

- `Python's sort function`: hàm này cho phép truyền vào custom comparison function.
- `Sklearn ML Library`: module preprocessing data hỗ trợ nhiều strategy khác nhau để xử lý dữ liệu: mean removal, variance scaling, standardization, …
- `TensorFlow ML Library`: module optimizers hỗ trợ nhiều strategy khác nhau để chạy gradient descent: adam, adagrad, …

### Tại sao lại cần đến Strategy?

Cứ mỗi khi thêm thuật toán mới, tư duy đơn giản nhất là chúng ta sẽ copy code của thuật toán cũ, sau đó sửa lại logic của thuật toán mới là work. Nếu ứng dụng nhỏ với vài ba thuật toán thì có thể chấp nhận được, tuy nhiên lâu dần nó sẽ nảy sinh ra vấn đề khi lượng thuật toán tăng lên:

- Duplicate code
- Quá trình copy code và sửa đổi dễ nhầm lẫn và sinh ra bug.
- Có những phần tính toán chung nằm trong các thuật toán, nếu có bug hay cần sửa đổi ở phần này thì phải sửa lại ở tất cả các lớp thuật toán.

### Triển khai Strategy pattern

- Strategy Interface: chứa interface thực thi một strategy cụ thể
- Concrete Strategy: triển khai strategy cụ thể
- Context: refer đến một strategy cụ thể

Hãy xem ví dụ triển khai strategy bên dưới cho hệ thống giao dịch (trading system):

```python
# Step 1: Create the Strategy Interface
class TradingStrategy(ABC):
    @abstractmethod
    def execute_trade(self, data):
        pass

# Step 2: Create Concrete Strategies
class MovingAverageStrategy(TradingStrategy):
    def execute_trade(self, data):
        # Calculate Moving Average (Simple example for illustration)
        window_size = 3  # Adjust as needed
        moving_average = sum(data[-window_size:]) / window_size
        return f"Executing Moving Average Trading Strategy. Moving Average: {moving_average:.2f}"

class MeanReversionStrategy(TradingStrategy):
    def execute_trade(self, data):
        # Calculate Mean Reversion (Simple example for illustration)
        mean_value = sum(data) / len(data)
        deviation = data[-1] - mean_value
        return f"Executing Mean Reversion Trading Strategy. Deviation from Mean: {deviation:.2f}"

# Step 3: Create the Context
class TradingContext:
    def __init__(self, strategy):
        self._strategy = strategy

    def set_strategy(self, strategy):
        self._strategy = strategy

    def execute_trade(self, data):
        return self._strategy.execute_trade(data)

# Main Section to Showcase Usage
if __name__ == "__main__":
    # Sample data for trading
    trading_data = [50, 55, 45, 60, 50]

    # Create concrete strategy objects
    moving_average_strategy = MovingAverageStrategy()
    mean_reversion_strategy = MeanReversionStrategy()

    # Create context with a default strategy
    trading_context = TradingContext(moving_average_strategy)

    # Execute the default strategy
    result = trading_context.execute_trade(trading_data)
    print(result)  # Output: Executing Moving Average Trading Strategy. Moving Average: 51.67

    # Switch to a different strategy at runtime
    trading_context.set_strategy(mean_reversion_strategy)
    
    # Execute the updated strategy
    result = trading_context.execute_trade(trading_data)
    print(result)  # Output: Executing Mean Reversion Trading Strategy. Deviation from Mean: -1.00
```

```
Executing Moving Average Trading Strategy. Moving Average: 51.67
Executing Mean Reversion Trading Strategy. Deviation from Mean: -2.00
```

## Template Method

Template Method giúp chúng ta định nghĩa các interface của thuật toán ở lớp cha (superclass) - cái này gọi là abstraction, còn về chi tiết cách thực hiện thì lớp con sẽ triển khai.

### Tại sao lại cần đến Template method?

Giả sử chúng ta có một hệ thống datamining để xử lý dữ liệu của công ty, phiên bản đầu tiên ứng dụng làm việc với file doc, trong các phiên bản tiếp theo nó hỗ trợ thêm các định dạng khác: csv, json, pdf, xml, … Và lúc đó chúng ta nhận ra rằng, code xử lý và phân tích dữ liệu với các loại định dạng file là hoàn toàn giống nhau, chỉ khác nhau ở bước tiền xử lý dữ liệu tương ứng với từng loại định dạng file, và chúng ta hoàn toàn có thể loại bỏ sự trùng lặp code này.

Tư tưởng của Template method khá giống với Strategy, chỉ khác ở chỗ:

- Template Method dựa trên sự kế thừa: cho phép thay đổi các phần của một thuật toán bằng cách mở rộng các phần đó trong các lớp con.
- Strategy dựa trên cấu tạo: cho phép thay đổi các phần trong hành vi của đối tượng bằng cách cung cấp cho đối tượng các strategy khác nhau tương ứng với hành vi đó.

Template Method hoạt động ở cấp độ lớp, vì vậy nó là tĩnh. Strategy hoạt động ở cấp độ đối tượng, cho phép bạn chuyển đổi hành vi khi chạy chương trình (run-time).

### Hạn chế của template method

❌ Các template method có thể khó bảo trì hơn khi chúng có nhiều bước hơn.

❌ Áp dụng template method có thể đã khiến chúng ta vi phạm nguyên tắc Liskov Substitution, khi mà việc triển khai Concrete Template của class con đã chặn bước triển khai mặc định của class cha, khi đó chúng ta không thể dùng class con để thay thế (substitute) cho class cha được nữa.

### Triển khai Template method pattern

Hãy xem ví dụ triển khai template method bên dưới cho hệ thống web scraper, xử lý với nhiều loại data input khác nhau (url, html, …):

```python
from abc import ABC, abstractmethod
import requests
import urllib

from bs4 import BeautifulSoup

# Step 1: Abstract Class (Web Scraper Template)
class WebScraperTemplate(ABC):
    @abstractmethod
    def send_request(self, url):
        """Abstract method for sending HTTP requests."""
        pass

    @abstractmethod
    def parse_html(self, content):
        """Abstract method for parsing HTML content."""
        pass

    @abstractmethod
    def extract_data(self, soup):
        """Abstract method for extracting data from parsed HTML."""
        pass

    # Template method orchestrating the steps
    def scrape_website(self, url):
        """The template method for web scraping."""
        content = self.send_request(url)
        soup = self.parse_html(content)
        data = self.extract_data(soup)
        return data

# Step 2: Concrete Class 1 (RequestsAndBeautifulSoupWebScraper)
class RequestsAndBeautifulSoupWebScraper(WebScraperTemplate):
    def send_request(self, url):
        """Concrete implementation for sending HTTP requests using requests library."""
        response = requests.get(url)
        return response.content

    def parse_html(self, content):
        """Concrete implementation for parsing HTML content using BeautifulSoup."""
        soup = BeautifulSoup(content, 'html.parser')
        return soup

    def extract_data(self, soup):
        """Concrete implementation for extracting data from parsed HTML."""
        # Example: Extracting all hyperlinks from the webpage
        links = [a['href'] for a in soup.find_all('a', href=True)]
        return links

# Step 3: Concrete Class 2 (UrllibAndRegexWebScraper)
class UrllibAndRegexWebScraper(WebScraperTemplate):
    def send_request(self, url):
        """Concrete implementation for sending HTTP requests using urllib."""
        with urllib.request.urlopen(url) as response:
            return response.read()

    def parse_html(self, content):
        """Concrete implementation for parsing HTML content using custom parser."""
        # Custom HTML parsing logic
        return content

    def extract_data(self, content):
        """Concrete implementation for extracting data using regex."""
        import re
        # Example: Extracting all email addresses from the webpage
        emails = re.findall(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', content.decode('utf-8'))
        return emails

# Main Section (Client Code)
if __name__ == "__main__":
    # Creating instances of the concrete classes
    web_scraper_requests_beautifulsoup = RequestsAndBeautifulSoupWebScraper()
    web_scraper_urllib_regex = UrllibAndRegexWebScraper()

    # Using the template method to scrape websites
    target_url = "https://amirlavasani.vercel.app/"

    print("Scraped Data using RequestsAndBeautifulSoupWebScraper:")
    data_requests_beautifulsoup = web_scraper_requests_beautifulsoup.scrape_website(target_url)
    for link in data_requests_beautifulsoup:
        print(link)

    print("\nScraped Data using UrllibAndRegexWebScraper:")
    data_urllib_regex = web_scraper_urllib_regex.scrape_website(target_url)
    for email in data_urllib_regex:
        print(email)
```

```
Scraped Data using RequestsAndBeautifulSoupWebScraper:
/
/blog
/projects
/about
/resume
/
/blog
/projects
/about
/resume
/about
/projects
/blog/lora-a-revolution-in-fine-tuning
/blog/lora-a-revolution-in-fine-tuning
/tags/ai
/tags/deep-learning
/tags/paper
/tags/llm
/tags/fine-tuning
/blog/lora-a-revolution-in-fine-tuning
/blog/attention-is-all-you-need
/blog/attention-is-all-you-need
/tags/ai
/tags/deep-learning
/tags/attention
/tags/paper
/tags/transformers
/blog/attention-is-all-you-need
/blog/beyond-recognition-openAIs-whisper
/blog/beyond-recognition-openAIs-whisper
/tags/openai
/tags/deep-learning
/tags/asr
/tags/paper
/blog/beyond-recognition-openAIs-whisper
/blog
mailto:amirm.lavasani@gmail.com
https://github.com/AmirLavasani
https://www.linkedin.com/in/amir-lavasani/ 
https://twitter.com/AmirMLavasani
/
https://nextjs.org/
https://tailwindcss.com/

Scraped Data using UrllibAndRegexWebScraper:
amirm.lavasani@gmail.com
amirm.lavasani@gmail.com
```

## Visitor

Visitor là pattern cho phép một “visitor” (khách thăm) thêm các phép toán lên các phần tử (element) của một cấu trúc đối tượng mà không thay đổi các lớp của các phần tử đó. Hãy tưởng tượng visitor như một khách thăm đến nhà bạn và thực hiện các hoạt động khác nhau trong mỗi phòng mà không thay đổi cấu trúc của ngôi nhà.

### Tại sao lại cần đến Visitor?

Giả sử bạn đang cần phát triển một module ghi log cho một hệ thống, cách đơn giản nhất là thêm code logging cho những class mà cần ghi log, tuy nhiên nó sẽ vi phạm nguyên tắc open-closed khi mà chúng ta sửa trực tiếp trên code hiện tại

Bây giờ chúng ta thử áp dụng Visitor, chúng ta sẽ đặt hành vi mới (ở đây là ghi log) vào một class riêng biệt được gọi là visitor, thay vì cố gắng tích hợp nó vào các class hiện có. Lúc này, đối tượng gốc cần ghi log sẽ không cần phải thực hiện hành vi ghi log mà sẽ ủy quyền cho một method của Visitor dưới dạng tham số, cung cấp cho method này quyền truy cập vào các dữ liệu cần thiết có trong đối tượng gốc để có thể ghi log được.

Tư tưởng của Visitor khá giống với Command, đều cho phép đối tượng thực hiện các hành vi trên các đối tượng khác nhau thuộc các lớp khác nhau, chỉ khác nhau ở chỗ:

- Command tập trung vào việc đóng gói (encapsulating) các yêu cầu dưới dạng đối tượng, giúp chúng ta dễ dàng quản lý các hành động trong chương trình.
- Visitor tập trung vào việc mở rộng, thêm các hành vi (thuật toán) mới vào các lớp mà không ảnh hưởng đến lớp hiện có.

### Triển khai Visitor

Để triển khai visitor, chúng ta cần làm hai thứ:

- Triển khai method `Visit()` được quản lý bởi Visitor, có quyền truy cập vào các phần tử (element).
- Triển khai method `Accept()` trong các đối tượng gốc để accept các visitor.

Như vậy là chúng ta vẫn phải thay đổi các class gốc (thêm method `Accept()`), tuy nhiên sự thay đổi này là nhỏ và không hề ảnh hưởng đến code đang chạy, cũng không phải thay đổi theo một số hành vi cụ thể (ghi log, tính toán, …), và đặc biệt là sau này nếu có thêm hành vi thì chúng ta sẽ không phải update code một lần nữa, chỉ cần thêm visitor là được.

Hãy xem ví dụ dưới đây triển khai module export document:

```python
# Step 1: Define the Visitor Interface
class Exporter:
    def export_paragraph(self, paragraph):
        pass

    def export_heading(self, heading):
        pass

    def export_image(self, image):
        pass

# Step 2: Implement Concrete Visitors
class HTMLExporter(Exporter):
    def export_paragraph(self, paragraph):
        return f"<p>{paragraph.content}</p>"

    def export_heading(self, heading):
        return f"<h{heading.level}>{heading.text}</h{heading.level}>"

    def export_image(self, image):
        return f'<img src="{image.source}" alt="{image.alt}">'

class PDFExporter(Exporter):
    def export_paragraph(self, paragraph):
        return f"PDF Paragraph: {paragraph.content}"

    def export_heading(self, heading):
        return f"PDF Heading ({heading.level}): {heading.text}"

    def export_image(self, image):
        return f'PDF Image: {image.alt}'

# Step 3: Define Element Interface
class Element:
    def accept(self, visitor):
        pass

# Step 4: Implement Concrete Elements
class Paragraph(Element):
    def __init__(self, content):
        self.content = content

    def accept(self, visitor):
        return visitor.export_paragraph(self)

class Heading(Element):
    def __init__(self, level, text):
        self.level = level
        self.text = text

    def accept(self, visitor):
        return visitor.export_heading(self)

class Image(Element):
    def __init__(self, source, alt):
        self.source = source
        self.alt = alt

    def accept(self, visitor):
        return visitor.export_image(self)

# Step 5: Client Code
class Document:
    def __init__(self, elements):
        self.elements = elements

    def accept(self, visitor):
        result = []
        for element in self.elements:
            result.append(element.accept(visitor))
        return result

# Usage
html_exporter = HTMLExporter()
pdf_exporter = PDFExporter()

document = Document([
    Paragraph("This is a paragraph."),
    Heading(1, "Main Heading"),
    Image("image.jpg", "A beautiful image")
])

# Export to HTML
html_result = document.accept(html_exporter)
print("HTML Export:")
print('\n'.join(html_result))
print("\n")

# Export to PDF
pdf_result = document.accept(pdf_exporter)
print("PDF Export:")
print('\n'.join(pdf_result))
```

```
HTML Export:
<p>This is a paragraph.</p>
<h1>Main Heading</h1>
<img src="image.jpg" alt="A beautiful image">

PDF Export:
PDF Paragraph: This is a paragraph.
PDF Heading (1): Main Heading
PDF Image: A beautiful image
```

You can get full source code [here](https://github.com/hoangph3/Object-Oriented-Programming/blob/main/DesignPattern/Behavior.ipynb)!
