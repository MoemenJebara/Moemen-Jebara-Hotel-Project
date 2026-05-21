"""
╔══════════════════════════════════════════════════════════════╗
║          GRAND AZURE HOTEL — FRONT DESK MANAGEMENT          ║
║                   Interactive Booking System                 ║
╚══════════════════════════════════════════════════════════════╝
"""

from abc import ABC, abstractmethod
from datetime import datetime


class BookableItem(ABC):
    """General template for any bookable hotel item."""

    def __init__(self, item_id: str, name: str, base_price: float):
        self._item_id   = item_id      
        self._name      = name
        self.__price    = None         
        self.base_price = base_price  

   
    @property
    def item_id(self):
        return self._item_id

    @property
    def name(self):
        return self._name

    @property
    def base_price(self):
        return self.__price

    @base_price.setter
    def base_price(self, value):
        if not isinstance(value, (int, float)):
            raise TypeError("Price must be a number.")
        if value < 0:
            raise ValueError("Price cannot be negative.")
        self.__price = float(value)


    @abstractmethod
    def calculate_item_cost(self) -> float:
        """Each subclass applies its own tax/gratuity logic."""
        pass

    @abstractmethod
    def display_details(self) -> str:
        """Returns a formatted detail string for the catalog."""
        pass




class HotelRoom(BookableItem):
    """
    A bookable overnight room.
    Unique properties: bed_size, smoking_allowed.
    Tax: 15% city hospitality tax applied at checkout.
    """

    VALID_BED_SIZES   = {"King", "Queen", "Twin"}
    CITY_TAX_RATE     = 0.15

    def __init__(self, item_id, name, base_price,
                 bed_size: str, smoking_allowed: bool = False):
        super().__init__(item_id, name, base_price)
        self.bed_size        = bed_size          # validated via setter
        self.smoking_allowed = smoking_allowed   # validated via setter

    @property
    def bed_size(self):
        return self.__bed_size

    @bed_size.setter
    def bed_size(self, value):
        if value not in self.VALID_BED_SIZES:
            raise ValueError(f"bed_size must be one of {self.VALID_BED_SIZES}")
        self.__bed_size = value

    @property
    def smoking_allowed(self):
        return self.__smoking_allowed

    @smoking_allowed.setter
    def smoking_allowed(self, value):
        if not isinstance(value, bool):
            raise TypeError("smoking_allowed must be True or False.")
        self.__smoking_allowed = value

    def calculate_item_cost(self) -> float:
        tax    = self.base_price * self.CITY_TAX_RATE
        return self.base_price + tax

    def display_details(self) -> str:
        smoking = "✔ Smoking" if self.smoking_allowed else "✘ Non-Smoking"
        return (f"[{self.item_id}]  {self.name:<32}"
                f"Bed: {self.bed_size:<6}  {smoking:<14}"
                f"  ${self.base_price:>8.2f}/night")



class HotelService(BookableItem):
    """
    A bookable extra service (spa, dining, etc.).
    Unique properties: time_slot, duration_minutes.
    Gratuity: 20% mandatory staff gratuity at checkout.
    """

    GRATUITY_RATE = 0.20

    def __init__(self, item_id, name, base_price,
                 time_slot: str, duration_minutes: int):
        super().__init__(item_id, name, base_price)
        self.time_slot        = time_slot
        self.duration_minutes = duration_minutes


    @property
    def time_slot(self):
        return self.__time_slot

    @time_slot.setter
    def time_slot(self, value):
        if not isinstance(value, str) or not value.strip():
            raise ValueError("time_slot must be a non-empty string (e.g. '10:00 AM').")
        self.__time_slot = value.strip()

    @property
    def duration_minutes(self):
        return self.__duration_minutes

    @duration_minutes.setter
    def duration_minutes(self, value):
        if not isinstance(value, int) or value <= 0:
            raise ValueError("duration_minutes must be a positive integer.")
        self.__duration_minutes = value

    
    def calculate_item_cost(self) -> float:
        gratuity = self.base_price * self.GRATUITY_RATE
        return self.base_price + gratuity

    def display_details(self) -> str:
        return (f"[{self.item_id}]  {self.name:<32}"
                f"Slot: {self.time_slot:<10}  "
                f"{self.duration_minutes} min"
                f"  ${self.base_price:>8.2f}/session")



class CustomerReservation:
    """
    Holds all items a guest has chosen.
    Calls calculate_item_cost() polymorphically at checkout.
    """

    def __init__(self, guest_name: str):
        if not guest_name.strip():
            raise ValueError("Guest name cannot be empty.")
        self.__guest_name = guest_name.strip()
        self.__items: list[BookableItem] = []

    @property
    def guest_name(self):
        return self.__guest_name

    def add_item(self, item: BookableItem):
        self.__items.append(item)

    def is_empty(self) -> bool:
        return len(self.__items) == 0

    def view_items(self):
        if self.is_empty():
            print("\n  ⚠  Your reservation is currently empty.\n")
            return
        print(f"\n  {'─'*62}")
        print(f"  Current Reservation for: {self.__guest_name}")
        print(f"  {'─'*62}")
        for item in self.__items:
            kind = "Room" if isinstance(item, HotelRoom) else "Service"
            print(f"  [{kind}]  {item.name:<32}  Base: ${item.base_price:.2f}")
        print(f"  {'─'*62}\n")

    def print_final_folio(self):
        """
        Prints a formatted checkout bill.
        Uses polymorphism: same function name, different behaviour per type.
        """
        now = datetime.now().strftime("%d %b %Y  %H:%M")

        print()
        print("  ╔" + "═"*60 + "╗")
        print("  ║{:^60}║".format("GRAND AZURE HOTEL"))
        print("  ║{:^60}║".format("Final Checkout Folio"))
        print("  ║{:^60}║".format(now))
        print("  ╠" + "═"*60 + "╣")
        print("  ║  {:<58}║".format(f"Guest: {self.__guest_name}"))
        print("  ╠" + "═"*60 + "╣")
        print("  ║  {:<30} {:>10}  {:>12}  ║".format("Item", "Base ($)", "Total ($)"))
        print("  ║" + "─"*60 + "║")

        grand_total = 0.0
        for item in self.__items:
            total = item.calculate_item_cost()      # ← polymorphic call
            grand_total += total
            if isinstance(item, HotelRoom):
                note = f"+15% city tax"
            else:
                note = f"+20% gratuity"
            print("  ║  {:<30} {:>10.2f}  {:>12.2f}  ║".format(
                item.name[:30], item.base_price, total))
            print("  ║    {:<56}║".format(f"  ↳ {note}"))

        print("  ╠" + "═"*60 + "╣")
        print("  ║  {:<44} {:>12.2f}  ║".format("GRAND TOTAL (USD)", grand_total))
        print("  ╚" + "═"*60 + "╝")
        print()
        print("  Thank you for staying at Grand Azure Hotel.")
        print("  We hope to welcome you again soon. 🌟")
        print()



def build_catalog() -> dict[str, BookableItem]:
    offerings = [
        HotelRoom("R01", "Deluxe King Room",        180.00, "King",  False),
        HotelRoom("R02", "Superior Queen Room",     140.00, "Queen", False),
        HotelRoom("R03", "Twin Bed Room",            110.00, "Twin",  False),
        HotelRoom("R04", "Executive King Suite",    320.00, "King",  False),
        HotelRoom("R05", "Smoking King Room",        160.00, "King",  True),
        HotelService("S01", "Couples Spa Package",   90.00, "10:00 AM", 90),
        HotelService("S02", "Deep Tissue Massage",   60.00, "02:00 PM", 60),
        HotelService("S03", "Fine Dining Experience",75.00, "07:30 PM", 90),
        HotelService("S04", "Breakfast Buffet",      25.00, "08:00 AM", 45),
        HotelService("S05", "Rooftop Yoga Session",  35.00, "06:30 AM", 60),
    ]
    return {item.item_id: item for item in offerings}



DIVIDER = "  " + "─" * 62

def print_header():
    print()
    print("  ╔" + "═"*62 + "╗")
    print("  ║{:^62}║".format("🏨  GRAND AZURE HOTEL  🏨"))
    print("  ║{:^62}║".format("Front Desk Management System"))
    print("  ╚" + "═"*62 + "╝")
    print()

def print_menu():
    print(DIVIDER)
    print("  MAIN MENU")
    print(DIVIDER)
    print("   [1]  View Hotel Offerings")
    print("   [2]  Add Item to Reservation")
    print("   [3]  View Current Reservation")
    print("   [4]  Print Final Bill / Checkout")
    print("   [5]  Exit")
    print(DIVIDER)

def view_catalog(catalog: dict):
    print()
    print(DIVIDER)
    print("  HOTEL OFFERINGS — ROOMS")
    print(DIVIDER)
    for item in catalog.values():
        if isinstance(item, HotelRoom):
            print("  " + item.display_details())
    print()
    print(DIVIDER)
    print("  HOTEL OFFERINGS — SERVICES")
    print(DIVIDER)
    for item in catalog.values():
        if isinstance(item, HotelService):
            print("  " + item.display_details())
    print(DIVIDER)
    print()

def main():
    print_header()

    # Get guest name
    while True:
        guest_name = input("  Please enter the guest's name: ").strip()
        if guest_name:
            break
        print("  ⚠  Name cannot be empty. Please try again.")

    catalog     = build_catalog()
    reservation = CustomerReservation(guest_name)

    print(f"\n  Welcome, {guest_name}! Your reservation has been created.\n")

    while True:
        print_menu()
        choice = input("  Enter your choice (1–5): ").strip()

        if choice == "1":
            view_catalog(catalog)

        elif choice == "2":
            view_catalog(catalog)
            item_id = input("  Enter the Item ID to add (e.g. R01, S03): ").strip().upper()
            if item_id in catalog:
                reservation.add_item(catalog[item_id])
                print(f"\n  ✔  '{catalog[item_id].name}' has been added to the reservation.\n")
            else:
                print(f"\n  ⚠  Item ID '{item_id}' not found. Please check the catalog.\n")

      
        elif choice == "3":
            reservation.view_items()

   
        elif choice == "4":
            if reservation.is_empty():
                print("\n  ⚠  No items in reservation. Please add items first.\n")
            else:
                reservation.print_final_folio()

        elif choice == "5":
            print()
            print("  Thank you for using the Grand Azure Hotel system.")
            print("  Goodbye! 👋")
            print()
            break

        else:
            print(f"\n  ⚠  '{choice}' is not a valid option. "
                  "Please enter a number between 1 and 5.\n")


if __name__ == "__main__":
    main()


