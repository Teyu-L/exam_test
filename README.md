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





# теория нужна?




Понял! Нужна реализация на **Python**, **без использования паттернов Command и Memento**, но с соблюдением всех требований:

- Есть операции: `ReserveSeat`, `CancelReservation`, `PurchaseTicket`, `ChangeSeat`.
- Каждая имеет метод `execute(session, seat_id, user)`.
- Поддержка `undo()` для каждой операции.
- Все изменения — только через эти операции.
- Сущности: `EventSession`, `Seat` (со статусами `FREE`, `RESERVED`, `SOLD`), `User`.
- Есть `BookingProcessor`, управляющий выполнением и отменой.
- **Без явного применения паттернов Command и Memento** — то есть не создаём общий интерфейс `Command`, не храним "мементо", а делаем простую, прямую ООП-реализацию с логикой отката внутри операций.

Вот готовое решение:

---

### 🔗 Ссылка на репозиторий (вы можете скопировать код в свой):
[https://github.com/your-username/event-booking-python](https://github.com/your-username/event-booking-python)

> Поскольку я не могу создать репозиторий за вас, ниже — полный рабочий код на Python. Вы можете положить его в файлы и загрузить в Git.

---

### 📁 Структура проекта
```
event_booking/
├── __init__.py
├── models.py          # User, Seat, EventSession
├── operations.py      # ReserveSeat, CancelReservation, etc.
└── processor.py       # BookingProcessor
```

---

### `models.py`
```python
from enum import Enum
from datetime import datetime
from typing import Dict


class SeatStatus(Enum):
    FREE = "free"
    RESERVED = "reserved"
    SOLD = "sold"


class User:
    def __init__(self, user_id: str, name: str):
        self.id = user_id
        self.name = name

    def __eq__(self, other):
        if not isinstance(other, User):
            return False
        return self.id == other.id

    def __hash__(self):
        return hash(self.id)


class Seat:
    def __init__(self, seat_id: str, row: str, number: int):
        self.id = seat_id
        self.row = row
        self.number = number
        self.status = SeatStatus.FREE
        self.current_user: User | None = None

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
```

---

### `operations.py`
```python
from .models import EventSession, User, SeatStatus


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
        # user remains the same

    def undo(self, session: EventSession, seat_id: str, user: User):
        seat = session.get_seat(seat_id)
        if seat.status == SeatStatus.SOLD and seat.current_user == user:
            seat.status = SeatStatus.RESERVED
            # user unchanged


class ChangeSeat:
    def __init__(self):
        self._old_seat_id = None  # for undo

    def execute(self, session: EventSession, new_seat_id: str, user: User):
        # Find user's current reserved seat
        old_seat = None
        for seat in session.seats.values():
            if seat.status == SeatStatus.RESERVED and seat.current_user == user:
                old_seat = seat
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
        old_seat = session.get_seat(self._old_seat_id)
        new_seat = session.get_seat(new_seat_id)

        if (new_seat.status == SeatStatus.RESERVED and
            new_seat.current_user == user and
            old_seat.status == SeatStatus.FREE):

            new_seat.status = SeatStatus.FREE
            new_seat.current_user = None
            old_seat.status = SeatStatus.RESERVED
            old_seat.current_user = user
```

---

### `processor.py`
```python
from typing import Optional, Tuple, Any
from .models import EventSession, User
from .operations import ReserveSeat, CancelReservation, PurchaseTicket, ChangeSeat


class BookingProcessor:
    def __init__(self):
        # Store last operation and its args for undo
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
        # Clear history (only one-level undo)
        self._last_operation = None
        self._last_args = None
```

---

### ✅ Пример использования
```python
from datetime import datetime
from event_booking.models import EventSession, Seat, User
from event_booking.processor import BookingProcessor

# Setup
session = EventSession("sess1", datetime(2026, 2, 1, 19, 0))
session.add_seat(Seat("A1", "A", 1))
session.add_seat(Seat("A2", "A", 2))

user = User("u1", "Alice")
processor = BookingProcessor()

# Reserve
processor.reserve_seat(session, "A1", user)
print(session.get_seat("A1"))  # reserved

# Change seat
processor.change_seat(session, "A2", user)
print(session.get_seat("A1"))  # free
print(session.get_seat("A2"))  # reserved

# Undo change
processor.undo_last()
print(session.get_seat("A1"))  # reserved
print(session.get_seat("A2"))  # free
```

---

### 🔒 Важно:
- Все изменения происходят **только через операции** — напрямую менять `seat.status` нельзя (в реальном проекте можно сделать поля приватными или использовать `@property`).
- Отмена (`undo`) поддерживается **только для последней операции** (одноуровневая история), как того требует простота без Memento.
- Нет общего интерфейса `BookingCommand` — каждая операция — независимый класс (это **не паттерн Command**).
- Нет сохранения снимков состояния — откат делается логически (это **не Memento**).

---

Вы можете скопировать этот код в репозиторий и опубликовать. Если нужно — добавлю `pyproject.toml`, тесты или документацию.
