# Python Design Patterns
---
# Creational Patterns
## Singleton
```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a = Singleton()
b = Singleton()
print(a is b)  # True
```
### singleton with decorator
```python
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Database:
    def __init__(self):
        self.connection = "connected"

db1 = Database()
db2 = Database()
print(db1 is db2)  # True
```
### thread-safe singleton
```python
import threading

class Singleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```
## Factory Method
```python
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, message: str): ...

class EmailNotification(Notification):
    def send(self, message: str):
        print(f"Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str):
        print(f"SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str):
        print(f"Push: {message}")

class NotificationFactory:
    _creators = {
        "email": EmailNotification,
        "sms": SMSNotification,
        "push": PushNotification,
    }

    @classmethod
    def create(cls, channel: str) -> Notification:
        creator = cls._creators.get(channel)
        if not creator:
            raise ValueError(f"Unknown channel: {channel}")
        return creator()

notif = NotificationFactory.create("email")
notif.send("hello")  # Email: hello
```
## Abstract Factory
```python
from abc import ABC, abstractmethod

class Button(ABC):
    @abstractmethod
    def render(self): ...

class TextField(ABC):
    @abstractmethod
    def render(self): ...

class DarkButton(Button):
    def render(self):
        return "[ Dark Button ]"

class DarkTextField(TextField):
    def render(self):
        return "| Dark Input |"

class LightButton(Button):
    def render(self):
        return "[ Light Button ]"

class LightTextField(TextField):
    def render(self):
        return "| Light Input |"

class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: ...
    @abstractmethod
    def create_text_field(self) -> TextField: ...

class DarkThemeFactory(UIFactory):
    def create_button(self):
        return DarkButton()
    def create_text_field(self):
        return DarkTextField()

class LightThemeFactory(UIFactory):
    def create_button(self):
        return LightButton()
    def create_text_field(self):
        return LightTextField()

def build_ui(factory: UIFactory):
    btn = factory.create_button()
    txt = factory.create_text_field()
    print(btn.render(), txt.render())

build_ui(DarkThemeFactory())   # [ Dark Button ] | Dark Input |
build_ui(LightThemeFactory())  # [ Light Button ] | Light Input |
```
## Builder
```python
class Query:
    def __init__(self):
        self._select = []
        self._from = None
        self._where = []
        self._order = None
        self._limit = None

    def select(self, *fields):
        self._select.extend(fields)
        return self

    def from_table(self, table):
        self._from = table
        return self

    def where(self, condition):
        self._where.append(condition)
        return self

    def order_by(self, field):
        self._order = field
        return self

    def limit(self, n):
        self._limit = n
        return self

    def build(self) -> str:
        parts = [f"SELECT {', '.join(self._select) or '*'}"]
        parts.append(f"FROM {self._from}")
        if self._where:
            parts.append(f"WHERE {' AND '.join(self._where)}")
        if self._order:
            parts.append(f"ORDER BY {self._order}")
        if self._limit:
            parts.append(f"LIMIT {self._limit}")
        return " ".join(parts)

sql = (
    Query()
    .select("name", "age")
    .from_table("users")
    .where("age > 18")
    .where("active = true")
    .order_by("name")
    .limit(10)
    .build()
)
print(sql)
# SELECT name, age FROM users WHERE age > 18 AND active = true ORDER BY name LIMIT 10
```
## Prototype
```python
import copy

class Config:
    def __init__(self):
        self.settings = {"debug": False, "db": "postgres", "cache": {"ttl": 300}}

    def clone(self):
        return copy.deepcopy(self)

base = Config()
dev = base.clone()
dev.settings["debug"] = True
dev.settings["cache"]["ttl"] = 10

print(base.settings["debug"])        # False (unchanged)
print(base.settings["cache"]["ttl"]) # 300 (unchanged)
print(dev.settings["debug"])          # True
print(dev.settings["cache"]["ttl"])   # 10
```
---
# Structural Patterns
## Adapter
```python
class OldPaymentGateway:
    def make_payment(self, amount_in_cents: int):
        return f"paid {amount_in_cents} cents"

class NewPaymentInterface:
    def pay(self, amount: float): ...

class PaymentAdapter(NewPaymentInterface):
    def __init__(self, old_gateway: OldPaymentGateway):
        self._gateway = old_gateway

    def pay(self, amount: float):
        cents = int(amount * 100)
        return self._gateway.make_payment(cents)

adapter = PaymentAdapter(OldPaymentGateway())
print(adapter.pay(19.99))  # paid 1999 cents
```
## Decorator (wrapper, not Python decorator)
```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    @abstractmethod
    def write(self, data: str): ...
    @abstractmethod
    def read(self) -> str: ...

class FileDataSource(DataSource):
    def __init__(self):
        self._data = ""

    def write(self, data: str):
        self._data = data

    def read(self) -> str:
        return self._data

class DataSourceDecorator(DataSource):
    def __init__(self, source: DataSource):
        self._source = source

    def write(self, data: str):
        self._source.write(data)

    def read(self) -> str:
        return self._source.read()

class EncryptionDecorator(DataSourceDecorator):
    def write(self, data: str):
        encrypted = data[::-1]  # simple reverse as "encryption"
        super().write(encrypted)

    def read(self) -> str:
        return super().read()[::-1]

class CompressionDecorator(DataSourceDecorator):
    def write(self, data: str):
        compressed = data.replace("  ", " ")
        super().write(compressed)

    def read(self) -> str:
        return super().read()

source = CompressionDecorator(EncryptionDecorator(FileDataSource()))
source.write("hello  world")
print(source.read())  # hello world
```
## Facade
```python
class CPU:
    def freeze(self): print("CPU freeze")
    def jump(self, addr): print(f"CPU jump to {addr}")
    def execute(self): print("CPU executing")

class Memory:
    def load(self, addr, data): print(f"Memory load {data} at {addr}")

class Disk:
    def read(self, sector): return f"data@sector{sector}"

class ComputerFacade:
    def __init__(self):
        self._cpu = CPU()
        self._memory = Memory()
        self._disk = Disk()

    def start(self):
        self._cpu.freeze()
        data = self._disk.read(0)
        self._memory.load(0x00, data)
        self._cpu.jump(0x00)
        self._cpu.execute()

computer = ComputerFacade()
computer.start()
```
## Proxy
```python
class HeavyImage:
    def __init__(self, path: str):
        self._path = path
        self._load()

    def _load(self):
        print(f"loading {self._path} from disk...")

    def display(self):
        print(f"displaying {self._path}")

class LazyImageProxy:
    def __init__(self, path: str):
        self._path = path
        self._real = None

    def display(self):
        if self._real is None:
            self._real = HeavyImage(self._path)
        self._real.display()

img = LazyImageProxy("photo.jpg")  # nothing loaded yet
img.display()  # loads on first access, then displays
img.display()  # just displays (already loaded)
```
## Composite
```python
from abc import ABC, abstractmethod

class FileSystemItem(ABC):
    @abstractmethod
    def size(self) -> int: ...
    @abstractmethod
    def display(self, indent: int = 0): ...

class File(FileSystemItem):
    def __init__(self, name: str, size: int):
        self.name = name
        self._size = size

    def size(self) -> int:
        return self._size

    def display(self, indent=0):
        print(" " * indent + f"📄 {self.name} ({self._size}B)")

class Directory(FileSystemItem):
    def __init__(self, name: str):
        self.name = name
        self._children: list[FileSystemItem] = []

    def add(self, item: FileSystemItem):
        self._children.append(item)
        return self

    def size(self) -> int:
        return sum(child.size() for child in self._children)

    def display(self, indent=0):
        print(" " * indent + f"📁 {self.name} ({self.size()}B)")
        for child in self._children:
            child.display(indent + 2)

root = Directory("src")
root.add(File("main.py", 1200))
root.add(File("utils.py", 800))
tests = Directory("tests")
tests.add(File("test_main.py", 600))
root.add(tests)
root.display()
# 📁 src (2600B)
#   📄 main.py (1200B)
#   📄 utils.py (800B)
#   📁 tests (600B)
#     📄 test_main.py (600B)
```
---
# Behavioral Patterns
## Strategy
```python
from abc import ABC, abstractmethod

class CompressionStrategy(ABC):
    @abstractmethod
    def compress(self, data: bytes) -> bytes: ...

class GzipStrategy(CompressionStrategy):
    def compress(self, data: bytes) -> bytes:
        import gzip
        return gzip.compress(data)

class LzmaStrategy(CompressionStrategy):
    def compress(self, data: bytes) -> bytes:
        import lzma
        return lzma.compress(data)

class Compressor:
    def __init__(self, strategy: CompressionStrategy):
        self._strategy = strategy

    def compress(self, data: bytes) -> bytes:
        return self._strategy.compress(data)

raw = b"hello world" * 1000
gz = Compressor(GzipStrategy()).compress(raw)
lz = Compressor(LzmaStrategy()).compress(raw)
print(f"original: {len(raw)}, gzip: {len(gz)}, lzma: {len(lz)}")
```
### strategy with functions (pythonic)
```python
def gzip_compress(data: bytes) -> bytes:
    import gzip
    return gzip.compress(data)

def lzma_compress(data: bytes) -> bytes:
    import lzma
    return lzma.compress(data)

def compress(data: bytes, strategy=gzip_compress) -> bytes:
    return strategy(data)

raw = b"hello world" * 1000
print(len(compress(raw, gzip_compress)))
print(len(compress(raw, lzma_compress)))
```
## Observer
```python
class EventEmitter:
    def __init__(self):
        self._listeners: dict[str, list] = {}

    def on(self, event: str, callback):
        self._listeners.setdefault(event, []).append(callback)
        return self

    def off(self, event: str, callback):
        self._listeners.get(event, []).remove(callback)
        return self

    def emit(self, event: str, *args, **kwargs):
        for cb in self._listeners.get(event, []):
            cb(*args, **kwargs)

emitter = EventEmitter()
emitter.on("user_created", lambda u: print(f"send welcome email to {u}"))
emitter.on("user_created", lambda u: print(f"init dashboard for {u}"))
emitter.on("user_deleted", lambda u: print(f"cleanup data for {u}"))

emitter.emit("user_created", "alice")
# send welcome email to alice
# init dashboard for alice
```
## Command
```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self): ...
    @abstractmethod
    def undo(self): ...

class Editor:
    def __init__(self):
        self.text = ""

    def __repr__(self):
        return f"Editor({self.text!r})"

class InsertCommand(Command):
    def __init__(self, editor: Editor, text: str):
        self._editor = editor
        self._text = text

    def execute(self):
        self._editor.text += self._text

    def undo(self):
        self._editor.text = self._editor.text[:-len(self._text)]

class CommandHistory:
    def __init__(self):
        self._history: list[Command] = []

    def push(self, cmd: Command):
        cmd.execute()
        self._history.append(cmd)

    def pop(self):
        if self._history:
            cmd = self._history.pop()
            cmd.undo()

editor = Editor()
history = CommandHistory()

history.push(InsertCommand(editor, "hello"))
history.push(InsertCommand(editor, " world"))
print(editor)  # Editor('hello world')

history.pop()
print(editor)  # Editor('hello')

history.pop()
print(editor)  # Editor('')
```
## Chain of Responsibility
```python
from abc import ABC, abstractmethod

class Handler(ABC):
    def __init__(self):
        self._next: Handler | None = None

    def set_next(self, handler: "Handler") -> "Handler":
        self._next = handler
        return handler

    def handle(self, request: dict):
        if self._next:
            return self._next.handle(request)
        return None

class AuthHandler(Handler):
    def handle(self, request: dict):
        if not request.get("token"):
            return "401 Unauthorized"
        print("auth passed")
        return super().handle(request)

class RateLimitHandler(Handler):
    def __init__(self, max_requests=5):
        super().__init__()
        self._count = 0
        self._max = max_requests

    def handle(self, request: dict):
        self._count += 1
        if self._count > self._max:
            return "429 Too Many Requests"
        print("rate limit passed")
        return super().handle(request)

class BusinessHandler(Handler):
    def handle(self, request: dict):
        return f"200 OK: processed {request.get('body', '')}"

auth = AuthHandler()
rate = RateLimitHandler(max_requests=2)
biz = BusinessHandler()
auth.set_next(rate).set_next(biz)

print(auth.handle({"token": "abc", "body": "data"}))   # 200 OK: processed data
print(auth.handle({"token": "abc", "body": "data2"}))  # 200 OK: processed data2
print(auth.handle({"token": "abc", "body": "data3"}))  # 429 Too Many Requests
print(auth.handle({"body": "no token"}))                # 401 Unauthorized
```
## State
```python
from abc import ABC, abstractmethod

class State(ABC):
    @abstractmethod
    def handle(self, order: "Order"): ...
    @abstractmethod
    def __str__(self) -> str: ...

class PendingState(State):
    def handle(self, order: "Order"):
        print("payment confirmed, moving to shipping")
        order.state = ShippingState()

    def __str__(self):
        return "Pending"

class ShippingState(State):
    def handle(self, order: "Order"):
        print("shipped, moving to delivered")
        order.state = DeliveredState()

    def __str__(self):
        return "Shipping"

class DeliveredState(State):
    def handle(self, order: "Order"):
        print("already delivered, no further action")

    def __str__(self):
        return "Delivered"

class Order:
    def __init__(self):
        self.state: State = PendingState()

    def proceed(self):
        print(f"[{self.state}] -> ", end="")
        self.state.handle(self)

order = Order()
order.proceed()  # [Pending] -> payment confirmed, moving to shipping
order.proceed()  # [Shipping] -> shipped, moving to delivered
order.proceed()  # [Delivered] -> already delivered, no further action
```
## Template Method
```python
from abc import ABC, abstractmethod

class DataPipeline(ABC):
    def run(self, source: str):
        data = self.extract(source)
        transformed = self.transform(data)
        self.load(transformed)

    @abstractmethod
    def extract(self, source: str) -> list: ...

    @abstractmethod
    def transform(self, data: list) -> list: ...

    def load(self, data: list):
        print(f"loaded {len(data)} records")

class CSVPipeline(DataPipeline):
    def extract(self, source: str) -> list:
        print(f"reading CSV from {source}")
        return [{"name": "alice"}, {"name": "bob"}]

    def transform(self, data: list) -> list:
        return [{"name": r["name"].upper()} for r in data]

class APIPipeline(DataPipeline):
    def extract(self, source: str) -> list:
        print(f"fetching API from {source}")
        return [{"name": "charlie"}]

    def transform(self, data: list) -> list:
        return [{"name": r["name"].title(), "source": "api"} for r in data]

CSVPipeline().run("data.csv")   # reading CSV... loaded 2 records
APIPipeline().run("/api/users")  # fetching API... loaded 1 records
```
## Iterator (using protocol)
```python
class Range:
    def __init__(self, start: int, end: int):
        self._start = start
        self._end = end

    def __iter__(self):
        current = self._start
        while current < self._end:
            yield current
            current += 1

for n in Range(1, 5):
    print(n)  # 1, 2, 3, 4

print(list(Range(10, 15)))  # [10, 11, 12, 13, 14]
```
### infinite iterator
```python
class Fibonacci:
    def __iter__(self):
        a, b = 0, 1
        while True:
            yield a
            a, b = b, a + b

from itertools import islice
print(list(islice(Fibonacci(), 10)))  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```
## Mediator
```python
class ChatRoom:
    def __init__(self):
        self._users: dict[str, "User"] = {}

    def register(self, user: "User"):
        self._users[user.name] = user
        user.room = self

    def send(self, message: str, sender: "User", to: str | None = None):
        if to:
            if to in self._users:
                self._users[to].receive(message, sender.name)
        else:
            for name, user in self._users.items():
                if name != sender.name:
                    user.receive(message, sender.name)

class User:
    def __init__(self, name: str):
        self.name = name
        self.room: ChatRoom | None = None

    def send(self, message: str, to: str | None = None):
        self.room.send(message, self, to)

    def receive(self, message: str, sender: str):
        print(f"[{self.name}] message from {sender}: {message}")

room = ChatRoom()
alice = User("alice")
bob = User("bob")
charlie = User("charlie")
room.register(alice)
room.register(bob)
room.register(charlie)

alice.send("hello everyone")           # bob and charlie receive
bob.send("hey alice", to="alice")      # only alice receives
```
---
# Pythonic Patterns
## descriptor (reusable property logic)
```python
class Validated:
    def __init__(self, min_val=None, max_val=None):
        self._min = min_val
        self._max = max_val

    def __set_name__(self, owner, name):
        self._name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self._name)

    def __set__(self, obj, value):
        if self._min is not None and value < self._min:
            raise ValueError(f"{self._name} must be >= {self._min}")
        if self._max is not None and value > self._max:
            raise ValueError(f"{self._name} must be <= {self._max}")
        obj.__dict__[self._name] = value

class Product:
    price = Validated(min_val=0)
    quantity = Validated(min_val=0, max_val=10000)

    def __init__(self, name, price, quantity):
        self.name = name
        self.price = price
        self.quantity = quantity

p = Product("widget", 9.99, 100)
print(p.price)  # 9.99
# p.price = -1  # raises ValueError: price must be >= 0
```
## __init_subclass__ (hook into subclass creation)
```python
_registry: dict[str, type] = {}

class Serializer:
    def __init_subclass__(cls, format_name: str = "", **kwargs):
        super().__init_subclass__(**kwargs)
        if format_name:
            _registry[format_name] = cls

    @classmethod
    def get(cls, name: str):
        return _registry[name]()

class JSONSerializer(Serializer, format_name="json"):
    def serialize(self, data):
        import json
        return json.dumps(data)

class YAMLSerializer(Serializer, format_name="yaml"):
    def serialize(self, data):
        return str(data)

serializer = Serializer.get("json")
print(serializer.serialize({"key": "value"}))  # {"key": "value"}
print(_registry)  # {'json': <class 'JSONSerializer'>, 'yaml': <class 'YAMLSerializer'>}
```
## context manager with __enter__ / __exit__
```python
class Timer:
    def __enter__(self):
        import time
        self._start = time.perf_counter()
        return self

    def __exit__(self, *exc):
        import time
        self.elapsed = time.perf_counter() - self._start
        print(f"elapsed: {self.elapsed:.4f}s")
        return False

with Timer() as t:
    sum(range(1_000_000))

print(t.elapsed)
```
### contextmanager decorator
```python
from contextlib import contextmanager

@contextmanager
def temp_env(key, value):
    import os
    old = os.environ.get(key)
    os.environ[key] = value
    try:
        yield
    finally:
        if old is None:
            del os.environ[key]
        else:
            os.environ[key] = old

with temp_env("DEBUG", "1"):
    import os
    print(os.environ["DEBUG"])  # 1
```
## __class_getitem__ (generic-like syntax)
```python
class Response:
    def __init__(self, data, status=200):
        self.data = data
        self.status = status

    def __class_getitem__(cls, item):
        def factory(data, **kwargs):
            return cls(data, **kwargs)
        factory.__name__ = f"Response[{item.__name__}]"
        return factory

    def __repr__(self):
        return f"Response(data={self.data!r}, status={self.status})"

StringResponse = Response[str]
print(StringResponse("hello"))  # Response(data='hello', status=200)
```
## dataclass with slots and frozen
```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float

    def distance_to(self, other: "Point") -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1.distance_to(p2))  # 5.0
# p1.x = 10  # raises FrozenInstanceError
print({p1, p2})  # hashable, can be used in sets
```
## __slots__ (memory optimization)
```python
class WithSlots:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithoutSlots:
    def __init__(self, x, y):
        self.x = x
        self.y = y

import sys
a = WithSlots(1, 2)
b = WithoutSlots(1, 2)
print(sys.getsizeof(a))  # ~56 bytes
print(sys.getsizeof(b))  # ~48 bytes, but b.__dict__ adds ~104 bytes
# total b memory: ~152 bytes vs ~56 bytes for a
# a.z = 3  # raises AttributeError (no __dict__)
```
