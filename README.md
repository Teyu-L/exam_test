class User:
    """Представляет пользователя системы бронирования."""

    def __init__(self, user_id, name):
        """Инициализирует пользователя.

        Args:
            user_id (str): Уникальный идентификатор пользователя.
            name (str): Имя пользователя.
        """
        self.id = user_id
        self.name = name

    def __eq__(self, other):
        """Сравнивает пользователей по идентификатору."""
        return isinstance(other, User) and self.id == other.id


class Seat:
    """Представляет место на мероприятии.

    Статусы места:
        - 'free': свободно
        - 'reserved': забронировано
        - 'sold': куплено
    """

    def __init__(self, seat_id, row, number):
        """Инициализирует место.

        Args:
            seat_id (str): Уникальный идентификатор места.
            row (int): Номер ряда.
            number (int): Номер места в ряду.
        """
        self.id = seat_id
        self.row = row
        self.number = number
        self.status = "free"
        self.user = None

    def reserve(self, user):
        """Бронирует место для пользователя.

        Args:
            user (User): Пользователь, который бронирует место.

        Returns:
            bool: True, если бронирование успешно, иначе False.
        """        if self.status == "free":
            self.status = "reserved"
            self.user = user
            return True
        return False

    def cancel(self):
        """Отменяет бронь или покупку, возвращая место в свободное состояние.

        Returns:
            bool: Всегда True.
        """
        self.status = "free"
        self.user = None
        return True

    def purchase(self, user):
        """Подтверждает покупку забронированного места.

        Args:
            user (User): Пользователь, совершающий покупку.

        Returns:
            bool: True, если покупка успешна, иначе False.
        """
        if self.status == "reserved" and self.user == user:
            self.status = "sold"
            return True
        return False

    def change_to(self, new_seat, user):
        """Меняет текущее место на другое.

        Args:
            new_seat (Seat): Новое место для бронирования.
            user (User): Пользователь, меняющий место.

        Returns:
            bool: True, если смена прошла успешно, иначе False.
        """
        self.cancel()
        return new_seat.reserve(user)


class EventSession:
    """Представляет сессию мероприятия (например, концерт, спектакль)."""

    def __init__(self, session_id, time):
        """Инициализирует сессию.
        Args:
            session_id (str): Уникальный идентификатор сессии.
            time (str): Время проведения (в формате строки, например '2026-01-17 19:00').
        """
        self.id = session_id
        self.time = time
        self.seats = {}

    def add_seat(self, seat):
        """Добавляет место в сессию.

        Args:
            seat (Seat): Объект места.
        """
        self.seats[seat.id] = seat

    def get_seat(self, seat_id):
        """Возвращает место по его идентификатору.

        Args:
            seat_id (str): Идентификатор места.

        Returns:
            Seat or None: Место, если найдено, иначе None.
        """
        return self.seats.get(seat_id)


class BookingProcessor:
    """Обрабатывает операции бронирования и поддерживает отмену действий."""

    def __init__(self):
        """Инициализирует процессор с пустой историей операций."""
        self.history = []

    def execute(self, session, seat_id, user, action, new_seat_id=None):
        """Выполняет операцию над местом.

        Поддерживаемые действия: 'reserve', 'cancel', 'purchase', 'change'.

        Args:
            session (EventSession): Сессия мероприятия.
            seat_id (str): Идентификатор текущего места.
            user (User): Пользователь, выполняющий действие.
            action (str): Тип операции.
            new_seat_id (str, optional): Идентификатор нового места (для 'change').

        Returns:
            bool: True, если операция выполнена успешно, иначе False.
        """        seat = session.get_seat(seat_id)
        old_status = seat.status
        old_user = seat.user

        if action == "reserve":
            success = seat.reserve(user)
        elif action == "cancel":
            success = seat.cancel()
        elif action == "purchase":
            success = seat.purchase(user)
        elif action == "change":
            new_seat = session.get_seat(new_seat_id)
            success = seat.change_to(new_seat, user)
        else:
            success = False

        if success:
            self.history.append({
                "seat": seat,
                "old_status": old_status,
                "old_user": old_user,
            })

        return success

    def undo(self):
        """Отменяет последнюю выполненную операцию.

        Returns:
            bool: True, если отмена выполнена, иначе False (нет операций).
        """
        if not self.history:
            return False
        last = self.history.pop()
        last["seat"].status = last["old_status"]
        last["seat"].user = last["old_user"]
        return True
