
"""
ПР №3. Застосування метакласів, декораторів
та паттернів проектування у Python.

Автор: Григор'євська Вікторія
Група: БІКСб-2-25-4.О.д
"""

import functools
import inspect
import json
import time

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta


# ============================================================
# ЧАСТИНА 1. МЕТАКЛАСИ ТА МЕХАНІЗМИ СТВОРЕННЯ КЛАСІВ
# ============================================================

# -------------------- Завдання 1.1 ---------------------------

class PluginMeta(type):
    """Метаклас для реєстрації та перевірки плагінів."""

    _registry = {}
    REQUIRED_ATTRS = ("name", "version")

    def __new__(mcs, cls_name, bases, namespace):
        # Створюємо клас
        cls = super().__new__(mcs, cls_name, bases, namespace)

        # Базовий клас не реєструємо
        if bases:
            for attr in mcs.REQUIRED_ATTRS:
                if attr not in namespace:
                    raise TypeError(
                        f"Клас {cls_name} повинен містити "
                        f"обов'язковий атрибут '{attr}'"
                    )

            plugin_name = namespace["name"]

            if plugin_name in mcs._registry:
                raise TypeError(
                    f"Плагін з іменем '{plugin_name}' вже зареєстрований"
                )

            mcs._registry[plugin_name] = cls

        return cls

    @classmethod
    def get_registry(mcs):
        """Повертає копію реєстру плагінів."""
        return mcs._registry.copy()

    @classmethod
    def create_plugin(mcs, plugin_name):
        """Створює екземпляр плагіна за його іменем."""
        if plugin_name not in mcs._registry:
            raise ValueError(
                f"Плагін '{plugin_name}' не знайдено"
            )

        return mcs._registry[plugin_name]()


class BasePlugin(metaclass=PluginMeta):
    """Базовий клас плагіна."""

    @abstractmethod
    def execute(self, data):
        pass


class UpperPlugin(BasePlugin):
    name = "upper"
    version = "1.0"

    def execute(self, data):
        return data.upper()


class ReversePlugin(BasePlugin):
    name = "reverse"
    version = "1.0"

    def execute(self, data):
        return data[::-1]


class CensorPlugin(BasePlugin):
    name = "censor"
    version = "1.1"

    def execute(self, data):
        words_to_censor = ["погано", "помилка"]
        result = data

        for word in words_to_censor:
            result = result.replace(word, "***")
            result = result.replace(word.capitalize(), "***")

        return result


# Динамічне створення класу за допомогою type()
def hash_execute(self, data):
    return str(hash(data))


HashPlugin = type(
    "HashPlugin",
    (BasePlugin,),
    {
        "name": "hash",
        "version": "2.0",
        "execute": hash_execute
    }
)


def demonstrate_plugins():
    print("\n--- ЗАВДАННЯ 1.1. СИСТЕМА ПЛАГІНІВ ---")

    print("Зареєстровані плагіни:")

    for name, cls in PluginMeta.get_registry().items():
        print(f"{name} (v{cls.version}): {cls.__name__}")

    test_data = "Привіт Світ"

    print("\nРезультати виконання:")

    for name in PluginMeta.get_registry():
        plugin = PluginMeta.create_plugin(name)
        print(f"{name}: {plugin.execute(test_data)}")

    # Демонстрація помилки валідації
    try:
        class IncorrectPlugin(BasePlugin):
            name = "incorrect"

            def execute(self, data):
                return data

    except TypeError as error:
        print("\nОчікувана помилка:", error)


# -------------------- Завдання 1.2 ---------------------------

class ValidatedField:
    """Дескриптор із перевіркою типу та діапазону."""

    def __init__(self, expected_type, min_value=None, max_value=None):
        self.expected_type = expected_type
        self.min_value = min_value
        self.max_value = max_value

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self

        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if type(value) is not self.expected_type:
            raise TypeError(
                f"Поле '{self.name}' повинно мати тип "
                f"{self.expected_type.__name__}"
            )

        if self.min_value is not None and value < self.min_value:
            raise ValueError(
                f"Поле '{self.name}' не може бути меншим "
                f"за {self.min_value}"
            )

        if self.max_value is not None and value > self.max_value:
            raise ValueError(
                f"Поле '{self.name}' не може бути більшим "
                f"за {self.max_value}"
            )

        setattr(obj, self.private_name, value)


class ConfigSection:
    """Базовий клас автоматичної реєстрації секцій."""

    _sections = {}

    def __init_subclass__(cls, section_name="", **kwargs):
        super().__init_subclass__(**kwargs)

        if not section_name:
            raise ValueError(
                f"Для секції {cls.__name__} потрібно вказати section_name"
            )

        if section_name in ConfigSection._sections:
            raise ValueError(
                f"Секція '{section_name}' вже існує"
            )

        ConfigSection._sections[section_name] = cls

    @classmethod
    def get_section(cls, name):
        return cls._sections.get(name)

    @classmethod
    def list_sections(cls):
        return list(cls._sections.keys())


class DatabaseConfig(ConfigSection, section_name="database"):
    host = ValidatedField(str)
    port = ValidatedField(int, min_value=1, max_value=65535)
    max_connections = ValidatedField(
        int, min_value=1, max_value=1000
    )

    def __init__(self, host, port, max_connections):
        self.host = host
        self.port = port
        self.max_connections = max_connections


class LoggingConfig(ConfigSection, section_name="logging"):
    level = ValidatedField(str)
    max_file_size_mb = ValidatedField(
        int, min_value=1, max_value=1024
    )

    def __init__(self, level, max_file_size_mb):
        self.level = level
        self.max_file_size_mb = max_file_size_mb


def demonstrate_descriptors():
    print("\n--- ЗАВДАННЯ 1.2. ДЕСКРИПТОРИ ТА СЕКЦІЇ ---")

    print("Зареєстровані секції:", ConfigSection.list_sections())

    database = DatabaseConfig(
        "localhost", 5432, 100
    )

    logging = LoggingConfig(
        "INFO", 50
    )

    print(
        f"База даних: {database.host}, "
        f"порт: {database.port}, "
        f"підключення: {database.max_connections}"
    )

    print(
        f"Логування: {logging.level}, "
        f"розмір файлу: {logging.max_file_size_mb} МБ"
    )

    try:
        database.port = "5432"
    except TypeError as error:
        print("Очікувана помилка типу:", error)

    try:
        database.port = 70000
    except ValueError as error:
        print("Очікувана помилка діапазону:", error)


# ============================================================
# ЧАСТИНА 2. ДЕКОРАТОРИ ФУНКЦІЙ ТА КЛАСІВ
# ============================================================

# -------------------- Завдання 2.1 ---------------------------

def cache(max_size=128):
    """Декоратор кешування з обмеженням кількості записів."""

    def decorator(func):
        saved_results = {}
        call_order = []

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = (args, tuple(sorted(kwargs.items())))

            try:
                hash(key)
            except TypeError:
                return func(*args, **kwargs)

            if key in saved_results:
                print(f"[CACHE] cache hit: {func.__name__}")
                return saved_results[key]

            print(f"[CACHE] cache miss: {func.__name__}")

            result = func(*args, **kwargs)

            if max_size > 0:
                saved_results[key] = result
                call_order.append(key)

                if len(call_order) > max_size:
                    oldest = call_order.pop(0)
                    saved_results.pop(oldest, None)

            return result

        wrapper.cache_info = lambda: {
            "size": len(saved_results),
            "max_size": max_size
        }

        def clear_cache():
            saved_results.clear()
            call_order.clear()

        wrapper.cache_clear = clear_cache

        return wrapper

    return decorator


def rate_limit(max_calls, period=60.0):
    """Обмежує кількість викликів функції за певний час."""

    if max_calls < 1 or period <= 0:
        raise ValueError("Некоректні параметри rate_limit")

    def decorator(func):
        call_times = []

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            current_time = time.monotonic()

            # Видаляємо застарілі записи
            while (
                call_times
                and current_time - call_times[0] >= period
            ):
                call_times.pop(0)

            if len(call_times) >= max_calls:
                raise RuntimeError(
                    f"Перевищено ліміт: максимум "
                    f"{max_calls} викликів за {period} секунд"
                )

            call_times.append(current_time)

            return func(*args, **kwargs)

        return wrapper

    return decorator


def log(level="INFO"):
    """Декоратор логування з вимірюванням часу виконання."""

    def decorator(func):

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print(
                f"[{level}] Виклик {func.__name__}"
                f"(args={args}, kwargs={kwargs})"
            )

            start_time = time.perf_counter()

            try:
                result = func(*args, **kwargs)

                elapsed = time.perf_counter() - start_time

                print(
                    f"[{level}] {func.__name__} -> {result} "
                    f"(час: {elapsed:.4f} с)"
                )

                return result

            except Exception as error:
                elapsed = time.perf_counter() - start_time

                print(
                    f"[ERROR] {func.__name__}: {error} "
                    f"(час: {elapsed:.4f} с)"
                )

                raise

        return wrapper

    return decorator


@cache(max_size=64)
def fibonacci(n):
    """Обчислення числа Фібоначчі."""

    if n < 2:
        return n

    return fibonacci(n - 1) + fibonacci(n - 2)


@rate_limit(max_calls=5, period=10.0)
def send_notification(message):
    return f"Надіслано: {message}"


@log(level="DEBUG")
def divide(a, b):
    if b == 0:
        raise ZeroDivisionError("Ділення на нуль неможливе")

    return a / b


def demonstrate_function_decorators():
    print("\n--- ЗАВДАННЯ 2.1. ДЕКОРАТОРИ ФУНКЦІЙ ---")

    print("Fibonacci(30):", fibonacci(30))
    print("Інформація про кеш:", fibonacci.cache_info())

    print("\nДемонстрація обмеження викликів:")

    for i in range(1, 7):
        try:
            print(send_notification(f"Повідомлення {i}"))
        except RuntimeError as error:
            print("Очікуване обмеження:", error)

    print("\nДемонстрація логування:")

    print("Результат ділення:", divide(10, 3))

    try:
        divide(10, 0)
    except ZeroDivisionError:
        print("Помилку ділення успішно перехоплено.")


# -------------------- Завдання 2.2 ---------------------------

def auto_repr(cls):
    """Автоматично створює метод __repr__."""

    signature = inspect.signature(cls.__init__)
    parameter_names = [
        name for name in signature.parameters
        if name != "self"
    ]

    def generated_repr(self):
        values = []

        for name in parameter_names:
            if hasattr(self, name):
                value = getattr(self, name)
                values.append(f"{name}={value!r}")

        return f"{cls.__name__}({', '.join(values)})"

    cls.__repr__ = generated_repr

    return cls


def frozen(cls):
    """Забороняє зміну атрибутів після ініціалізації."""

    original_init = cls.__init__
    original_setattr = cls.__setattr__

    @functools.wraps(original_init)
    def new_init(self, *args, **kwargs):
        object.__setattr__(self, "_frozen", False)

        original_init(self, *args, **kwargs)

        object.__setattr__(self, "_frozen", True)

    @functools.wraps(original_setattr)
    def new_setattr(self, name, value):
        if getattr(self, "_frozen", False):
            raise AttributeError(
                f"Об'єкт {cls.__name__} незмінний. "
                f"Не можна змінити атрибут '{name}'"
            )

        original_setattr(self, name, value)

    cls.__init__ = new_init
    cls.__setattr__ = new_setattr

    return cls


def log_methods(cls):
    """Додає логування до публічних методів класу."""

    for name, method in list(cls.__dict__.items()):
        if name.startswith("_") or not callable(method):
            continue

        if isinstance(method, (staticmethod, classmethod)):
            continue

        @functools.wraps(method)
        def wrapper(self, *args, __method=method,
                    __name=name, **kwargs):
            print(f"[METHOD] Виклик {__name}")

            try:
                result = __method(self, *args, **kwargs)
                print(f"[METHOD] Результат {__name}: {result}")
                return result

            except Exception as error:
                print(f"[METHOD ERROR] {__name}: {error}")
                raise

        setattr(cls, name, wrapper)

    return cls


@auto_repr
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def distance_to(self, other):
        return (
            (self.x - other.x) ** 2
            + (self.y - other.y) ** 2
        ) ** 0.5


@frozen
class ImmutableConfig:
    def __init__(self, host, port, debug=False):
        self.host = host
        self.port = port
        self.debug = debug


@log_methods
class MathService:
    def add(self, a, b):
        return a + b

    def multiply(self, a, b):
        return a * b

    def divide(self, a, b):
        if b == 0:
            raise ValueError("Ділення на нуль")
        return a / b


def demonstrate_class_decorators():
    print("\n--- ЗАВДАННЯ 2.2. ДЕКОРАТОРИ КЛАСІВ ---")

    point = Point(3.0, 4.0)
    print("Автоматичне представлення:", repr(point))

    config = ImmutableConfig(
        "localhost", 8080, debug=True
    )

    print(f"Конфігурація: {config.host}:{config.port}")

    try:
        config.host = "remote"
    except AttributeError as error:
        print("Очікувана помилка:", error)

    service = MathService()

    service.add(10, 20)
    service.multiply(3, 7)

    try:
        service.divide(10, 0)
    except ValueError:
        print("Помилку ділення оброблено.")


# ============================================================
# ЧАСТИНА 3. ПАТТЕРНИ ПРОЕКТУВАННЯ ТА MVC
# ============================================================

# -------------------- Завдання 3.1 ---------------------------

class SingletonMeta(type):
    """Метаклас, який гарантує один екземпляр класу."""

    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(
                *args, **kwargs
            )

        return cls._instances[cls]


class LogFormatter(ABC):
    """Абстрактний форматер повідомлень."""

    @abstractmethod
    def format(self, level, message, timestamp):
        pass


class PlainFormatter(LogFormatter):
    def format(self, level, message, timestamp):
        return (
            f"[{level}] "
            f"{timestamp.strftime('%Y-%m-%d %H:%M:%S')} "
            f"- {message}"
        )


class JSONFormatter(LogFormatter):
    def format(self, level, message, timestamp):
        data = {
            "level": level,
            "time": timestamp.strftime("%Y-%m-%d %H:%M:%S"),
            "message": message
        }

        return json.dumps(data, ensure_ascii=False)


class ColorFormatter(LogFormatter):
    COLORS = {
        "DEBUG": "\033[90m",
        "INFO": "\033[92m",
        "WARNING": "\033[93m",
        "ERROR": "\033[91m"
    }

    RESET = "\033[0m"

    def format(self, level, message, timestamp):
        color = self.COLORS.get(level, self.RESET)

        return (
            f"{color}[{level}] "
            f"{timestamp.strftime('%H:%M:%S')} "
            f"- {message}{self.RESET}"
        )


class FormatterFactory:
    """Фабрика для створення форматерів."""

    _formatters = {
        "plain": PlainFormatter,
        "json": JSONFormatter,
        "color": ColorFormatter
    }

    @classmethod
    def register(cls, name, formatter_cls):
        if not isinstance(formatter_cls, type):
            raise TypeError("Потрібно передати клас форматера")

        if not issubclass(formatter_cls, LogFormatter):
            raise TypeError(
                "Форматер повинен наслідувати LogFormatter"
            )

        cls._formatters[name] = formatter_cls

    @classmethod
    def create(cls, name):
        if name not in cls._formatters:
            raise ValueError(
                f"Невідомий форматер: {name}"
            )

        return cls._formatters[name]()

    @classmethod
    def available(cls):
        return list(cls._formatters.keys())


class Logger(metaclass=SingletonMeta):
    """Логер із підтримкою Singleton та фабрики форматерів."""

    LEVELS = {
        "DEBUG": 0,
        "INFO": 1,
        "WARNING": 2,
        "ERROR": 3
    }

    def __init__(self, formatter_name="plain",
                 min_level="DEBUG"):
        self._formatter = FormatterFactory.create(
            formatter_name
        )

        if min_level not in self.LEVELS:
            raise ValueError("Невідомий рівень логування")

        self._min_level = min_level
        self._log_history = []

    def set_formatter(self, name):
        self._formatter = FormatterFactory.create(name)

    def log(self, level, message):
        if level not in self.LEVELS:
            raise ValueError(
                f"Невідомий рівень: {level}"
            )

        if self.LEVELS[level] < self.LEVELS[self._min_level]:
            return

        timestamp = datetime.now()

        formatted = self._formatter.format(
            level, message, timestamp
        )

        print(formatted)
        self._log_history.append(formatted)

    def debug(self, msg):
        self.log("DEBUG", msg)

    def info(self, msg):
        self.log("INFO", msg)

    def warning(self, msg):
        self.log("WARNING", msg)

    def error(self, msg):
        self.log("ERROR", msg)

    def get_history(self):
        return self._log_history[:]


def demonstrate_singleton_factory():
    print("\n--- ЗАВДАННЯ 3.1. SINGLETON ТА FACTORY ---")

    logger1 = Logger("plain", "DEBUG")
    logger2 = Logger("json", "INFO")

    print("Один і той самий об'єкт:", logger1 is logger2)

    logger1.info("Система запущена")
    logger1.warning("Мало пам'яті")
    logger1.error("Помилка з'єднання")

    print("\nЗміна форматера на JSON:")

    logger1.set_formatter("json")
    logger1.debug("Діагностика")
    logger1.info("Роботу продовжено")

    print("\nДоступні форматери:")
    print(FormatterFactory.available())

    print("\nКількість записів в історії:")
    print(len(logger1.get_history()))


# -------------------- Завдання 3.2 ---------------------------

class EventEmitter:
    """Реалізація паттерну Observer."""

    def __init__(self):
        self._listeners = {}

    def on(self, event, callback):
        self._listeners.setdefault(event, [])

        if callback not in self._listeners[event]:
            self._listeners[event].append(callback)

    def off(self, event, callback):
        if event in self._listeners:
            if callback in self._listeners[event]:
                self._listeners[event].remove(callback)

    def emit(self, event, *args, **kwargs):
        for callback in self._listeners.get(event, [])[:]:
            callback(*args, **kwargs)


class SortStrategy(ABC):
    """Абстрактна стратегія сортування."""

    @abstractmethod
    def sort(self, tasks):
        pass


class SortByPriority(SortStrategy):
    def sort(self, tasks):
        return sorted(
            tasks,
            key=lambda task: task.priority
        )


class SortByDeadline(SortStrategy):
    def sort(self, tasks):
        return sorted(
            tasks,
            key=lambda task: task.deadline
        )


class SortByCreationDate(SortStrategy):
    def sort(self, tasks):
        return sorted(
            tasks,
            key=lambda task: task.created_at,
            reverse=True
        )


@dataclass
class Task:
    """Модель одного завдання."""

    title: str
    priority: int
    deadline: datetime
    completed: bool = False
    created_at: datetime = field(
        default_factory=datetime.now
    )


class TaskModel(EventEmitter):
    """Model: зберігання даних і бізнес-логіка."""

    def __init__(self):
        super().__init__()
        self._tasks = []

    def add_task(self, title, priority, deadline):
        if not title.strip():
            raise ValueError("Назва завдання не може бути порожньою")

        if not 1 <= priority <= 5:
            raise ValueError("Пріоритет повинен бути від 1 до 5")

        task = Task(title, priority, deadline)

        self._tasks.append(task)
        self.emit("task_added", task)

        return task

    def complete_task(self, index):
        if not 0 <= index < len(self._tasks):
            raise IndexError("Неправильний номер завдання")

        task = self._tasks[index]
        task.completed = True

        self.emit("task_completed", task)

        return task

    def remove_task(self, index):
        if not 0 <= index < len(self._tasks):
            raise IndexError("Неправильний номер завдання")

        task = self._tasks.pop(index)

        self.emit("task_removed", task)

        return task

    def get_tasks(self, include_completed=True):
        if include_completed:
            return self._tasks[:]

        return [
            task for task in self._tasks
            if not task.completed
        ]

    def get_statistics(self):
        total = len(self._tasks)
        completed = sum(
            task.completed for task in self._tasks
        )
        active = total - completed

        return {
            "total": total,
            "completed": completed,
            "active": active
        }


class TaskView:
    """View: відображення завдань у консолі."""

    def show_tasks(self, tasks, title="Завдання"):
        print(f"\n{title}")

        print("-" * 85)
        print(
            f"{'№':<4} {'Назва':<25} {'Пріоритет':<12} "
            f"{'Дедлайн':<22} {'Статус':<12}"
        )
        print("-" * 85)

        for index, task in enumerate(tasks, start=1):
            status = (
                "Виконано" if task.completed
                else "Активне"
            )

            print(
                f"{index:<4} "
                f"{task.title:<25} "
                f"{task.priority:<12} "
                f"{task.deadline.strftime('%d.%m.%Y %H:%M'):<22} "
                f"{status:<12}"
            )

        print("-" * 85)

    def show_message(self, message):
        print(f"[INFO] {message}")

    def show_statistics(self, stats):
        print("\nСТАТИСТИКА ЗАВДАНЬ")
        print(f"Усього завдань: {stats['total']}")
        print(f"Виконано: {stats['completed']}")
        print(f"Активних: {stats['active']}")


class TaskController:
    """Controller: взаємодія моделі та вигляду."""

    def __init__(self, model, view):
        self._model = model
        self._view = view

        self._sort_strategy = SortByPriority()

        # Підписка на події Observer
        self._model.on(
            "task_added",
            lambda task: self._view.show_message(
                f"Додано завдання: {task.title}"
            )
        )

        self._model.on(
            "task_completed",
            lambda task: self._view.show_message(
                f"Виконано завдання: {task.title}"
            )
        )

        self._model.on(
            "task_removed",
            lambda task: self._view.show_message(
                f"Видалено завдання: {task.title}"
            )
        )

    def set_sort_strategy(self, strategy):
        if not isinstance(strategy, SortStrategy):
            raise TypeError("Некоректна стратегія сортування")

        self._sort_strategy = strategy

    def add_task(self, title, priority, deadline):
        return self._model.add_task(
            title, priority, deadline
        )

    def complete_task(self, index):
        return self._model.complete_task(index)

    def show_all(self):
        tasks = self._model.get_tasks()
        sorted_tasks = self._sort_strategy.sort(tasks)

        self._view.show_tasks(
            sorted_tasks,
            title="Усі завдання"
        )

    def show_active(self):
        tasks = self._model.get_tasks(
            include_completed=False
        )

        sorted_tasks = self._sort_strategy.sort(tasks)

        self._view.show_tasks(
            sorted_tasks,
            title="Активні завдання"
        )

    def show_stats(self):
        self._view.show_statistics(
            self._model.get_statistics()
        )


def demonstrate_mvc():
    print("\n--- ЗАВДАННЯ 3.2. OBSERVER, STRATEGY ТА MVC ---")

    model = TaskModel()
    view = TaskView()
    controller = TaskController(model, view)

    now = datetime.now()

    # Додавання п'яти завдань
    controller.add_task(
        "Вивчити Python",
        1,
        now + timedelta(days=1)
    )

    controller.add_task(
        "Підготувати звіт",
        2,
        now + timedelta(days=3)
    )

    controller.add_task(
        "Виконати лабораторну",
        1,
        now + timedelta(days=2)
    )

    controller.add_task(
        "Повторити теорію",
        3,
        now + timedelta(days=5)
    )

    controller.add_task(
        "Завантажити роботу",
        2,
        now + timedelta(days=4)
    )

    # Сортування за пріоритетом
    controller.set_sort_strategy(SortByPriority())
    controller.show_all()

    # Зміна стратегії сортування
    print("\nЗміна стратегії на сортування за дедлайном:")

    controller.set_sort_strategy(SortByDeadline())
    controller.show_all()

    # Демонстрація третьої стратегії
    print("\nСортування за датою створення:")

    controller.set_sort_strategy(SortByCreationDate())
    controller.show_all()

    # Позначаємо перше завдання як виконане
    controller.complete_task(0)

    # Показуємо тільки активні завдання
    controller.show_active()

    # Виводимо статистику
    controller.show_stats()


# ============================================================
# ГОЛОВНА ФУНКЦІЯ ПРОГРАМИ
# ============================================================

def main():
    print("=" * 65)
    print("ПРАКТИЧНА РОБОТА №3")
    print("Метакласи, декоратори та паттерни проектування")
    print("=" * 65)

    demonstrate_plugins()
    demonstrate_descriptors()

    demonstrate_function_decorators()
    demonstrate_class_decorators()

    demonstrate_singleton_factory()
    demonstrate_mvc()

    print("\n" + "=" * 65)
    print("ВИКОНАННЯ ПРАКТИЧНОЇ РОБОТИ ЗАВЕРШЕНО")
    print("=" * 65)


if __name__ == "__main__":
    main()
