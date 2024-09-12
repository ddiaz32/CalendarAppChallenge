from dataclasses import dataclass, field
from datetime import datetime, date, time
from typing import ClassVar

from app.services.util import generate_unique_id, date_lower_than_today_error, event_not_found_error, \
    reminder_not_found_error, slot_not_available_error


# TODO: Implement Reminder class here
@dataclass
class Reminder:
    EMAIL:str ='email'
    
    SYSTEM:str ='system' 
    
    date_time : datetime

    type:str= "EMAIL"

    def __str__(self):
        return f"Recordar {self.date_time} de {self.type}"


# TODO: Implement Event class here
class Event:
    title: str
    description: str
    date_: date
    start_at: time
    end_at: time
    reminders: list[Reminder] = field(default_factory=list)
    id: str = field(default_factory=generate_unique_id)
    def add_reminder(self, date_time: datetime, reminder_type: str = "EMAIL") -> None:
        new_reminder = Reminder(date_time=date_time, type=reminder_type)
        self.reminders.append(new_reminder)

    def delete_reminder(self, reminder_index: int) -> None:
        if 0 <= reminder_index < len(self.reminders):
            del self.reminders[reminder_index]
        else:
            reminder_not_found_error()

    def __str__(self) -> str:
        return (f"ID: {self.id}\n"
                f"Event title: {self.title}\n"
                f"Description: {self.description}\n"
                f"Time: {self.start_at} - {self.end_at}")

# TODO: Implement Day class here
class Day:
    def __init__(self,date_:date):
        self.date_ =date_
        self.slots:dict[time,Union[str,None]] = {}
        self._init_slots()
        
    def _init_slots(self) -> None:
        current_time = time(0, 0)
        while current_time < time(23, 45):
            self.slots[current_time] = None

            minutes = (current_time.hour * 60 + current_time.minute + 15) % (24 * 60)
            current_time = time(minutes // 60, minutes % 60)
            
    def add_event(self, event_id: str, start_at: time, end_at: time) -> None:
        current_time = start_at
        while current_time < end_at:
            if self.slots.get(current_time) is not None:
                slot_not_available_error()
            self.slots[current_time] = event_id

            minutes = (current_time.hour * 60 + current_time.minute + 15) % (24 * 60)
            current_time = time(minutes // 60, minutes % 60)

    def delete_event(self, event_id: str) -> None:
        deleted = False
        for slot, saved_id in self.slots.items():
            if saved_id == event_id:
                self.slots[slot] = None
                deleted = True
        if not deleted:
            event_not_found_error()

    def update_event(self, event_id: str, start_at: time, end_at: time) -> None:

        for slot in self.slots:
            if self.slots[slot] == event_id:
                self.slots[slot] = None


        current_time = start_at
        while current_time < end_at:
            if self.slots.get(current_time) is not None and self.slots[current_time] != event_id:
                slot_not_available_error()
            self.slots[current_time] = event_id

            minutes = (current_time.hour * 60 + current_time.minute + 15) % (24 * 60)
            current_time = time(minutes // 60, minutes % 60)           



class Calendar:
    def __init__(self):
        self.days: Dict[date, 'Day'] = {}
        self.events: Dict[str, 'Event'] = {}

    def add_event(self, title: str, description: str, date_: date, start_at: time, end_at: time) -> str:
        if date_ < datetime.now().date():
            date_lower_than_today_error()

        if date_ not in self.days:
            self.days[date_] = Day(date_)

        event = Event(title=title, description=description, date_=date_, start_at=start_at, end_at=end_at)
        self.days[date_].add_event(event.id, start_at, end_at)
        self.events[event.id] = event

        return event.id

    def add_reminder(self, event_id: str, date_time: datetime, type_: str) -> None:
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()
        event.add_reminder(date_time=date_time, reminder_type=type_)

    def find_available_slots(self, date_: date) -> List[time]:
        if date_ not in self.days:
            return [slot for slot in self.days[date_].slots if self.days[date_].slots[slot] is None]
        return []

    def update_event(self, event_id: str, title: str, description: str, date_: date, start_at: time, end_at: time) -> None:
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()

        is_new_date = False

        if event.date_ != date_:
            self.delete_event(event_id)
            event = Event(title=title, description=description, date_=date_, start_at=start_at, end_at=end_at)
            event.id = event_id
            self.events[event_id] = event
            is_new_date = True
            if date_ not in self.days:
                self.days[date_] = Day(date_)
            day = self.days[date_]
            day.add_event(event_id, start_at, end_at)
        else:
            event.title = title
            event.description = description
            event.date_ = date_
            event.start_at = start_at
            event.end_at = end_at

        for day in self.days.values():
            if not is_new_date and event_id in day.slots.values():
                day.delete_event(event.id)
                day.update_event(event.id, start_at, end_at)

    def delete_event(self, event_id: str) -> None:
        if event_id not in self.events:
            event_not_found_error()

        self.events.pop(event_id)

        for day in self.days.values():
            if event_id in day.slots.values():
                day.delete_event(event_id)
                break

    def find_events(self, start_at: date, end_at: date) -> Dict[date, List['Event']]:
        events: Dict[date, List['Event']] = {}
        for event in self.events.values():
            if start_at <= event.date_ <= end_at:
                if event.date_ not in events:
                    events[event.date_] = []
                events[event.date_].append(event)
        return events

    def delete_reminder(self, event_id: str, reminder_index: int) -> None:
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()
        event.delete_reminder(reminder_index)

    def list_reminders(self, event_id: str) -> List['Reminder']:
        event = self.events.get(event_id)
        if not event:
            event_not_found_error()
        return event.reminders