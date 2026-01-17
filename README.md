# Эт не джанго, а просто ооп, там один файл должен быть
## Эти три вопроса надо

### Зачем нужен .gitignore? Какие файлы и папки обычно добавляют в него для Python-проекта?
### Как полиморфизм реализуется в Python? Приведите пример.
### Объясните различия между реляционными и нереляционными БД. Когда что использовать?



# Можешь просто квену дать, но только без command  и memento
Задание ООП:

Вам необходимо реализовать систему бронирования мест на мероприятие (Event). Система поддерживает операции: ReserveSeat (бронирование), CancelReservation (отмена), PurchaseTicket (подтверждение брони и оплата), ChangeSeat. Каждая операция должна:

Иметь метод execute(EventSession session, String seatId, User user).

Менять состояние сеанса и конкретного места (статус: свободно, забронировано, продано).

Могла быть отменена (undo).

Все изменения должны происходить только через эти операции.



Ключевые сущности:

EventSession (id, time, map of seats).

Seat (id, row, number, status, currentUser).

User (id, name).

BookingCommand (интерфейс команды).

BookingProcessor, обеспечивающий последовательность и атомарность операций.


Реализуйте систему с привязкой к пользователям и сеансам мероприятия. В качестве ответа: ссылка на репозиторий с решением.
# Kod

```python
from enum import Enum
from datetime import datetime
from typing import Dict, Optional, Tuple, Any


# ================
# Модели
# ================

class SeatStatus(Enum):
    FREE = "free"
    RESERVED = "reserved"
    SOLD = "sold"


class User:
    def __init__(self, user_id: str, name: str):
        self.id = user_id
        self.name = name

    def __eq__(self, other):
        return isinstance(other, User) and self.id == other.id

    def __hash__(self):
        return hash(self.id)

    def __repr__(self):
        return f"User({self.id}, {self.name})"


class Seat:
    def __init__(self, seat_id: str, row: str, number: int):
        self.id = seat_id
        self.row = row
        self.number = number
        self.status = SeatStatus.FREE
        self.current_user: Optional[User] = None

    def __repr__(self):
        return f"Seat({self.id}, {self.status.value}, user={self.current_user.name if self.current_user else None})"


class EventSession:
    def __init__(self, session_id: str, event_time: datetime):
        self.id = session_id
        self.time = event_time
        self.seats: Dict[str, Seat] = {}

    def add_seat(self, seat: Seat):
        self.seats[seat.id] = seat

    def get_seat(self, seat_id: str) -> Seat:
        if seat_id not in self.seats:
            raise ValueError(f"Seat {seat_id} not found in session {self.id}")
        return self.seats[seat_id]


# ================
# Операции
# ================

class ReserveSeat:
    def execute(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status != SeatStatus.FREE:
            raise ValueError(f"Seat {seat_id} is not free")
        seat.status = SeatStatus.RESERVED
        seat.current_user = user

    def undo(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status == SeatStatus.RESERVED and seat.current_user == user:
            seat.status = SeatStatus.FREE
            seat.current_user = None


class CancelReservation:
    def execute(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status != SeatStatus.RESERVED:
            raise ValueError(f"Seat {seat_id} is not reserved")
        if seat.current_user != user:
            raise ValueError("Reservation belongs to another user")
        seat.status = SeatStatus.FREE
        seat.current_user = None

    def undo(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status == SeatStatus.FREE:
            seat.status = SeatStatus.RESERVED
            seat.current_user = user


class PurchaseTicket:
    def execute(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status != SeatStatus.RESERVED:
            raise ValueError("Seat must be reserved before purchase")
        if seat.current_user != user:
            raise ValueError("Reservation belongs to another user")
        seat.status = SeatStatus.SOLD

    def undo(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status == SeatStatus.SOLD and seat.current_user == user:
            seat.status = SeatStatus.RESERVED


class ChangeSeat:
    def __init__(self):
        self._old_seat_id: Optional[str] = None

    def execute(self, session: EventSession, new_seat_id: str, user: User):
        # Find currently reserved seat for this user
        old_seat = None
        for s in session.seats.values():
            if s.status == SeatStatus.RESERVED and s.current_user == user:
                old_seat = s
                break
        if old_seat is None:
            raise ValueError("User has no reserved seat to change from")

        new_seat = session.get_seat(new_seat_id)
        if new_seat.status != SeatStatus.FREE:
            raise ValueError(f"New seat {new_seat_id} is not free")

        # Perform change
        old_seat.status = SeatStatus.FREE
        old_seat.current_user = None
        new_seat.status = SeatStatus.RESERVED
        new_seat.current_user = user

        self._old_seat_id = old_seat.id

    def undo(self, session: EventSession, new_seat_id: str, user: User):
        if self._old_seat_id is None:
            return
        try:
            old_seat = session.get_seat(self._old_seat_id)
            new_seat = session.get_seat(new_seat_id)
        except ValueError:
            return  # seat disappeared — cannot undo safely

        if (new_seat.status == SeatStatus.RESERVED and
            new_seat.current_user == user and
            old_seat.status == SeatStatus.FREE):

            new_seat.status = SeatStatus.FREE
            new_seat.current_user = None
            old_seat.status = SeatStatus.RESERVED
            old_seat.current_user = user


# ================
# Процессор
# ================

class BookingProcessor:
    def __init__(self):
        self._last_operation: Optional[Any] = None
        self._last_args: Optional[Tuple[EventSession, str, User]] = None

    def reserve_seat(self, session: EventSession, seat_id: str, user: User):
        op = ReserveSeat()
        op.execute(session, seat_id, user)
        self._remember(op, session, seat_id, user)

    def cancel_reservation(self, session: EventSession, seat_id: str, user: User):
        op = CancelReservation()
        op.execute(session, seat_id, user)
        self._remember(op, session, seat_id, user)

    def purchase_ticket(self, session: EventSession, seat_id: str, user: User):
        op = PurchaseTicket()
        op.execute(session, seat_id, user)
        self._remember(op, session, seat_id, user)

    def change_seat(self, session: EventSession, new_seat_id: str, user: User):
        op = ChangeSeat()
        op.execute(session, new_seat_id, user)
        self._remember(op, session, new_seat_id, user)

    def _remember(self, op, session, seat_id, user):
        self._last_operation = op
        self._last_args = (session, seat_id, user)

    def undo_last(self):
        if self._last_operation is None or self._last_args is None:
            return
        op = self._last_operation
        session, seat_id, user = self._last_args
        op.undo(session, seat_id, user)
        self._last_operation = None
        self._last_args = None


# ================
# Пример использования
# ================

if __name__ == "__main__":
    # Создаём сессию и места
    session = EventSession("concert-001", datetime(2026, 3, 15, 20, 0))
    session.add_seat(Seat("A1", "A", 1))
    session.add_seat(Seat("A2", "A", 2))
    session.add_seat(Seat("B1", "B", 1))

    alice = User("u1", "Alice")
    processor = BookingProcessor()

    # Бронируем
    processor.reserve_seat(session, "A1", alice)
    print("After reserve:", session.get_seat("A1"))

    # Покупаем билет
    processor.purchase_ticket(session, "A1", alice)
    print("After purchase:", session.get_seat("A1"))

    # Отменяем покупку (через undo)
    processor.undo_last()
    print("After undo purchase:", session.get_seat("A1"))

    # Меняем место
    processor.change_seat(session, "A2", alice)
    print("After change: A1 =", session.get_seat("A1"), "| A2 =", session.get_seat("A2"))

    # Отменяем смену
    processor.undo_last()
    print("After undo change: A1 =", session.get_seat("A1"), "| A2 =", session.get_seat("A2"))

```

# Teoria

Вот краткие и ясные ответы на все три вопроса:

---

### 1. **Зачем нужен `.gitignore`? Какие файлы и папки обычно добавляют в него для Python-проекта?**

**Назначение `.gitignore`:**  
Файл `.gitignore` указывает Git, какие файлы и директории **не следует отслеживать** при коммитах. Это помогает избежать случайного добавления:
- временных или локальных файлов,
- секретных данных (паролей, ключей),
- артефактов сборки.

**Типичное содержимое `.gitignore` для Python-проекта:**
```gitignore
# Байт-код Python
__pycache__/
*.py[cod]
*$py.class

# Виртуальные окружения
venv/
env/
ENV/
.venv/
.env/

# Логи и базы данных
*.log
*.sqlite3
*.db

# IDE / редакторы
.vscode/
.idea/
*.swp
.DS_Store

# Переменные окружения
.env
.env.local

# Пакеты и зависимости
*.egg-info/
build/
dist/
```

Можно использовать шаблон с [gitignore.io](https://www.toptal.com/developers/gitignore) — например, запрос `python`.

---

### 2. **Как полиморфизм реализуется в Python? Приведите пример.**

**Полиморфизм** — это способность объектов разных типов обрабатывать один и тот же интерфейс (метод) по-разному.

В Python он реализуется **благодаря динамической типизации** и **наследованию**.

**Пример:**
```python
class Animal:
    def speak(self):
        raise NotImplementedError("Subclass must implement this method")

class Dog(Animal):
    def speak(self):
        return "Гав!"

class Cat(Animal):
    def speak(self):
        return "Мяу!"

# Полиморфное использование
animals = [Dog(), Cat()]
for animal in animals:
    print(animal.speak())  # Вызывается разный метод в зависимости от типа
```

**Вывод:**
```
Гав!
Мяу!
```

Python не требует явного объявления интерфейсов — достаточно, чтобы объект имел нужный метод (**утинная типизация**: «Если ходит как утка и крякает как утка — значит, это утка»).

---

### 3. **Различия между реляционными и нереляционными БД. Когда что использовать?**

| Характеристика            | Реляционные (SQL)                     | Нереляционные (NoSQL)                  |
|--------------------------|----------------------------------------|----------------------------------------|
| **Структура данных**     | Таблицы с фиксированной схемой         | Гибкая структура: документы, ключ-значение, графы и т.д. |
| **Язык запросов**        | SQL                                    | Зависит от СУБД (например, MongoDB Query Language) |
| **Связи между данными**  | Поддержка JOIN, внешние ключи          | Обычно денормализация, вложенные данные |
| **Масштабируемость**     | Вертикальная (сложнее горизонтальная)  | Горизонтальная (легко масштабируется)  |
| **ACID**                 | Полная поддержка                       | Часто eventual consistency (BASE)      |
| **Примеры**              | PostgreSQL, MySQL, SQLite              | MongoDB, Redis, Cassandra, Neo4j       |

**Когда использовать:**

- **Реляционные БД**, если:
  - Данные структурированы и взаимосвязаны (например, банковские транзакции).
  - Нужны строгие гарантии целостности (ACID).
  - Часто нужны сложные JOIN-запросы.

- **Нереляционные БД**, если:
  - Структура данных гибкая или часто меняется.
  - Требуется высокая производительность и горизонтальное масштабирование.
  - Данные можно хранить в виде документов, пар «ключ-значение» или графов (например, пользовательские профили, логи, рекомендательные системы).

---

Если хочешь — могу привести примеры архитектурных решений, где выбор СУБД критичен.
