import sqlite3
import datetime

# ============================================
# DATABASE CONFIGURATION (SQLite)
# ============================================

DB_NAME = "arsdb.db"

# In-memory storage (as per Class 12 syllabus)
ep = {}  # stores email-password pairs
bookings = {}  # stores all bookings with email as key


# ============================================
# DATABASE CONNECTION
# ============================================

def get_db_connection():
    try:
        conn = sqlite3.connect(DB_NAME)
        return conn
    except:
        print("Database Error!")
        return None


# ============================================
# DATABASE SETUP
# ============================================

def setup_database():
    conn = get_db_connection()
    if conn is None:
        return
    cursor = conn.cursor()

    # Create users table
    try:
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                email VARCHAR(100) PRIMARY KEY,
                password VARCHAR(100)
            )
        """)
    except:
        pass

    # Create flight_classes table
    try:
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS flight_classes (
                class_name VARCHAR(50) PRIMARY KEY,
                available_seats INT
            )
        """)
    except:
        pass

    # Create bookings table
    try:
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS bookings (
                booking_id INTEGER PRIMARY KEY AUTOINCREMENT,
                email VARCHAR(100),
                airline VARCHAR(100),
                class_name VARCHAR(100),
                departure VARCHAR(50),
                destination VARCHAR(50),
                travel_date DATE
            )
        """)
    except:
        pass

    # Create passengers table
    try:
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS passengers (
                passenger_id INTEGER PRIMARY KEY AUTOINCREMENT,
                email VARCHAR(100),
                travel_date DATE,
                passenger_name VARCHAR(100),
                age INT,
                gender VARCHAR(10)
            )
        """)
    except:
        pass

    # Insert default flight classes
    try:
        cursor.execute("SELECT COUNT(*) FROM flight_classes")
        if cursor.fetchone()[0] == 0:
            classes = [
                ('Economy', 20),
                ('Business Class', 10),
                ('First Class', 7),
                ('Premium Economy', 5)
            ]
            cursor.executemany("INSERT INTO flight_classes VALUES (?, ?)", classes)
            conn.commit()
    except:
        pass

    # Load existing users
    try:
        cursor.execute("SELECT email, password FROM users")
        for row in cursor.fetchall():
            ep[row[0]] = row[1]
    except:
        pass

    cursor.close()
    conn.close()


# ============================================
# DATE VALIDATION FUNCTION
# ============================================

def validate_date(date_str):
    """Check if date is valid and not in past"""
    # Check format DD-MM-YYYY
    if len(date_str) != 10:
        return False, "Date must be in DD-MM-YYYY format!"
    
    if date_str[2] != '-' or date_str[5] != '-':
        return False, "Date must be in DD-MM-YYYY format!"
    
    # Extract day, month, year
    try:
        day = int(date_str[0:2])
        month = int(date_str[3:5])
        year = int(date_str[6:10])
    except:
        return False, "Date must contain numbers only!"
    
    # Check month (1-12)
    if month < 1 or month > 12:
        return False, "Invalid month! Month must be between 1-12"
    
    # Check days based on month
    if month in [1, 3, 5, 7, 8, 10, 12]:
        if day < 1 or day > 31:
            return False, "Invalid day! This month has 31 days"
    elif month in [4, 6, 9, 11]:
        if day < 1 or day > 30:
            return False, "Invalid day! This month has 30 days"
    elif month == 2:
        # Check leap year
        if (year % 400 == 0) or (year % 4 == 0 and year % 100 != 0):
            if day < 1 or day > 29:
                return False, "Invalid day! February has 29 days in leap year"
        else:
            if day < 1 or day > 28:
                return False, "Invalid day! February has 28 days"
    
    # Check if date is in past
    today = datetime.date.today()
    try:
        travel_date = datetime.date(year, month, day)
        if travel_date < today:
            return False, "Cannot book for past dates! Please enter a future date"
    except:
        return False, "Invalid date!"
    
    return True, "Valid date"


# ============================================
# REGISTRATION
# ============================================

def register():
    conn = get_db_connection()
    if conn is None:
        print("Cannot connect to database!")
        return
    cursor = conn.cursor()

    print("\n" + "="*40)
    print(" REGISTRATION ")
    print("="*40)

    # Email input
    while True:
        em = input("Enter Email ID: ")
        if '@' in em and '.' in em:
            if em in ep:
                print("Email already exists! Please login.")
                login()
                return
            break
        else:
            print("Invalid email! Must contain @ and .")

    # Password input
    while True:
        pw = input("Enter Password (min 6 characters): ")
        if len(pw) >= 6:
            break
        else:
            print("Password must be at least 6 characters!")

    try:
        cursor.execute("INSERT INTO users VALUES (?, ?)", (em, pw))
        conn.commit()
        ep[em] = pw
        print("\n Registration Successful!")
        cursor.close()
        conn.close()
        login()
    except:
        print("Error in registration!")
        conn.rollback()
        cursor.close()
        conn.close()


# ============================================
# LOGIN
# ============================================

def login():
    print("\n" + "="*40)
    print(" LOGIN ")
    print("="*40)

    # Load users if not in memory
    if not ep:
        conn = get_db_connection()
        if conn:
            cursor = conn.cursor()
            cursor.execute("SELECT email, password FROM users")
            for row in cursor.fetchall():
                ep[row[0]] = row[1]
            cursor.close()
            conn.close()

    em = input("Enter Email ID: ")
    while True:
        if em not in ep:
            print("Email not found!")
            ch = input("Press 1 to retry, 2 to register: ")
            if ch == '2':
                register()
                return
            em = input("Enter Email ID: ")
        else:
            break

    pw = input("Enter Password: ")
    while True:
        if pw != ep[em]:
            print("Wrong password!")
            ch = input("Press 1 to retry, 2 to register: ")
            if ch == '2':
                register()
                return
            pw = input("Enter Password: ")
        else:
            break

    print("\n WELCOME TO MAIN MENU")
    load_user_bookings(em)
    menu(em)


# ============================================
# LOAD USER BOOKINGS
# ============================================

def load_user_bookings(email):
    conn = get_db_connection()
    if conn is None:
        return
    cursor = conn.cursor()
    
    try:
        cursor.execute("SELECT booking_id, airline, class_name, departure, destination, travel_date FROM bookings WHERE email=?", (email,))
        booking_rows = cursor.fetchall()
        
        if email not in bookings:
            bookings[email] = []
        
        for row in booking_rows:
            # Get passengers
            cursor.execute("SELECT passenger_name, age, gender FROM passengers WHERE email=? AND travel_date=?", (email, row[5]))
            passenger_rows = cursor.fetchall()
            
            passenger_list = []
            for p in passenger_rows:
                passenger_list.append({
                    "name": p[0],
                    "age": p[1],
                    "gender": p[2]
                })
            
            bookings[email].append({
                "booking_id": row[0],
                "airline": row[1],
                "flight_details": {
                    "class": row[2],
                    "departure": row[3],
                    "destination": row[4],
                    "date": row[5]
                },
                "passengers": passenger_list
            })
    except:
        pass
    
    cursor.close()
    conn.close()


# ============================================
# SAVE BOOKING TO DATABASE
# ============================================

def save_booking_to_db(email, airline, flight_details, passengers):
    conn = get_db_connection()
    if conn is None:
        return None
    cursor = conn.cursor()
    
    try:
        # Parse date
        d = flight_details['date'].split('-')
        travel_date = d[2] + '-' + d[1] + '-' + d[0]  # Convert to YYYY-MM-DD
        
        # Insert booking
        cursor.execute("""
            INSERT INTO bookings (email, airline, class_name, departure, destination, travel_date)
            VALUES (?, ?, ?, ?, ?, ?)
        """, (email, airline, flight_details['class'], flight_details['departure'], 
              flight_details['destination'], travel_date))
        booking_id = cursor.lastrowid
        
        # Insert passengers
        for passenger in passengers:
            cursor.execute("""
                INSERT INTO passengers (email, travel_date, passenger_name, age, gender)
                VALUES (?, ?, ?, ?, ?)
            """, (email, travel_date, passenger['name'], passenger['age'], passenger['gender']))
        
        # Update available seats
        cursor.execute("""
            UPDATE flight_classes
            SET available_seats = available_seats - ?
            WHERE class_name = ?
        """, (len(passengers), flight_details['class']))
        
        conn.commit()
        return booking_id
    except:
        conn.rollback()
        return None
    finally:
        cursor.close()
        conn.close()


# ============================================
# MAIN MENU
# ============================================

def main():
    print("\n" + "="*50)
    print(" AIRLINE RESERVATION SYSTEM ")
    print("="*50)
    
    setup_database()
    
    print("\n1. Register")
    print("2. Login")
    print("3. Exit")
    
    while True:
        ch = input("\nEnter choice: ")
        if ch == "1":
            register()
            break
        elif ch == "2":
            login()
            break
        elif ch == "3":
            print("\n Thank you for using Airline Reservation System!")
            return
        else:
            print("Invalid choice!")


# ============================================
# MENU
# ============================================

def menu(email):
    while True:
        print("\n" + "="*40)
        print(" MAIN MENU ")
        print("="*40)
        print("1. Book Flight")
        print("2. View Tickets")
        print("3. Edit Booking")
        print("4. Cancel Booking")
        print("5. View Profile")
        print("6. Logout")
        
        mm = input("\nEnter choice: ")
        
        if mm == "1":
            print("\n BOOK FLIGHT")
            flight_details = get_flight_details()
            airline = choose_airline()
            passenger_list = get_passenger_details()
            
            booking_id = save_booking_to_db(email, airline, flight_details, passenger_list)
            
            if booking_id:
                if email not in bookings:
                    bookings[email] = []
                bookings[email].append({
                    "booking_id": booking_id,
                    "airline": airline,
                    "flight_details": flight_details,
                    "passengers": passenger_list
                })
                print("\n BOOKING CONFIRMED!")
                print("Booking ID:", booking_id)
                print("Airline:", airline)
                print("Class:", flight_details['class'])
                print("Route:", flight_details['departure'], "->", flight_details['destination'])
                print("Date:", flight_details['date'])
                print("Passengers:", len(passenger_list))
                print("\nThank you for booking!")
            else:
                print("Booking failed!")
        
        elif mm == "2":
            print("\n VIEW TICKETS")
            view_tickets(email)
        
        elif mm == "3":
            print("\n EDIT BOOKING")
            edit_booking(email)
        
        elif mm == "4":
            print("\n CANCEL BOOKING")
            cancel_booking(email)
        
        elif mm == "5":
            print("\n VIEW PROFILE")
            view_profile(email)
        
        elif mm == "6":
            print("\n Logged out successfully!")
            break
        
        else:
            print("Invalid choice!")


# ============================================
# VIEW PROFILE
# ============================================

def view_profile(email):
    print("\n" + "="*40)
    print(" USER PROFILE ")
    print("="*40)
    print("Email:", email)
    
    if email in bookings:
        print("Total Bookings:", len(bookings[email]))
        total_passengers = 0
        for b in bookings[email]:
            total_passengers += len(b['passengers'])
        print("Total Passengers:", total_passengers)
    else:
        print("No bookings found.")


# ============================================
# CHOOSE AIRLINE
# ============================================

def choose_airline():
    print("\n" + "="*40)
    print(" AVAILABLE AIRLINES ")
    print("="*40)
    print("1. IndiGo")
    print("2. AirIndia")
    print("3. SpiceJet")
    print("4. Emirates")
    print("5. GoAir")
    print("6. Vistara")
    
    while True:
        ch = input("\nEnter choice (1-6): ")
        if ch == "1":
            return "IndiGo"
        elif ch == "2":
            return "AirIndia"
        elif ch == "3":
            return "SpiceJet"
        elif ch == "4":
            return "Emirates"
        elif ch == "5":
            return "GoAir"
        elif ch == "6":
            return "Vistara"
        else:
            print("Invalid choice!")


# ============================================
# GET FLIGHT DETAILS (WITH DATE VALIDATION)
# ============================================

def get_flight_details():
    print("\n" + "="*40)
    print(" FLIGHT CLASSES ")
    print("="*40)
    print("1. Economy - Available: 20")
    print("2. Business Class - Available: 10")
    print("3. First Class - Available: 7")
    print("4. Premium Economy - Available: 5")
    
    while True:
        cls = input("\nEnter choice (1-4): ")
        if cls == "1":
            class_name = "Economy"
            break
        elif cls == "2":
            class_name = "Business Class"
            break
        elif cls == "3":
            class_name = "First Class"
            break
        elif cls == "4":
            class_name = "Premium Economy"
            break
        else:
            print("Invalid choice!")
    
    departure = input("Enter Departure City: ")
    destination = input("Enter Destination City: ")
    
    while departure.lower() == destination.lower():
        print("Departure and destination cannot be same!")
        departure = input("Enter Departure City: ")
        destination = input("Enter Destination City: ")
    
    # Date validation with loop
    while True:
        date = input("Enter Travel Date (DD-MM-YYYY): ")
        valid, message = validate_date(date)
        if valid:
            break
        else:
            print(message)
            print("Please enter a valid date!\n")
    
    return {
        "class": class_name,
        "departure": departure,
        "destination": destination,
        "date": date
    }


# ============================================
# GET PASSENGER DETAILS
# ============================================

def get_passenger_details():
    while True:
        try:
            n = int(input("Enter number of passengers (max 10): "))
            if 0 < n <= 10:
                break
            print("Enter 1-10 only!")
        except:
            print("Enter a valid number!")
    
    passengers = []
    for i in range(n):
        print("\n Passenger", i+1)
        name = input("Name: ")
        while True:
            try:
                age = int(input("Age: "))
                if 1 <= age <= 120:
                    break
                print("Enter valid age!")
            except:
                print("Enter a number!")
        gender = input("Gender (M/F/O): ").upper()
        passengers.append({"name": name, "age": age, "gender": gender})
    
    return passengers


# ============================================
# VIEW TICKETS
# ============================================

def view_tickets(email):
    if email not in bookings or not bookings[email]:
        print("\n No bookings found!")
        return
    
    print("\n" + "="*60)
    print(" YOUR TICKETS ")
    print("="*60)
    
    for i, booking in enumerate(bookings[email], 1):
        print("\n" + "-"*40)
        print("Booking #", i)
        if booking.get('booking_id'):
            print("Booking ID:", booking['booking_id'])
        print("Airline:", booking['airline'])
        print("Class:", booking['flight_details']['class'])
        print("Route:", booking['flight_details']['departure'], "->", booking['flight_details']['destination'])
        print("Date:", booking['flight_details']['date'])
        print("\nPassengers:")
        for j, p in enumerate(booking['passengers'], 1):
            print("  ", j, ".", p['name'], "- Age:", p['age'], "- Gender:", p['gender'])
        print("Total:", len(booking['passengers']), "passengers")


# ============================================
# EDIT BOOKING
# ============================================

def edit_booking(email):
    if email not in bookings or not bookings[email]:
        print("No bookings to edit!")
        return
    
    view_tickets(email)
    
    try:
        num = int(input("\nEnter booking number to edit: "))
        if num < 1 or num > len(bookings[email]):
            print("Invalid number!")
            return
        num = num - 1
    except:
        print("Invalid input!")
        return
    
    booking = bookings[email][num]
    
    print("\n What to edit?")
    print("1. Flight Class")
    print("2. Route")
    print("3. Date")
    print("4. Passenger Details")
    print("5. Cancel")
    
    choice = input("Enter choice: ")
    
    if choice == "1":
        print("\n1. Economy")
        print("2. Business Class")
        print("3. First Class")
        print("4. Premium Economy")
        new_class = input("Enter new class: ")
        if new_class == "1":
            booking['flight_details']['class'] = "Economy"
        elif new_class == "2":
            booking['flight_details']['class'] = "Business Class"
        elif new_class == "3":
            booking['flight_details']['class'] = "First Class"
        elif new_class == "4":
            booking['flight_details']['class'] = "Premium Economy"
        else:
            print("Invalid choice!")
        print("Class updated!")
    
    elif choice == "2":
        dep = input("New departure: ")
        dest = input("New destination: ")
        booking['flight_details']['departure'] = dep
        booking['flight_details']['destination'] = dest
        print("Route updated!")
    
    elif choice == "3":
        while True:
            new_date = input("New date (DD-MM-YYYY): ")
            valid, message = validate_date(new_date)
            if valid:
                booking['flight_details']['date'] = new_date
                print("Date updated!")
                break
            else:
                print(message)
    
    elif choice == "4":
        print("\nPassengers:")
        for i, p in enumerate(booking['passengers'], 1):
            print(i, ".", p['name'])
        
        try:
            p_num = int(input("Enter passenger number: "))
            if p_num < 1 or p_num > len(booking['passengers']):
                print("Invalid number!")
                return
            p_num = p_num - 1
        except:
            print("Invalid input!")
            return
        
        passenger = booking['passengers'][p_num]
        
        print("\n1. Name")
        print("2. Age")
        print("3. Gender")
        opt = input("Enter choice: ")
        
        if opt == "1":
            passenger['name'] = input("New name: ")
        elif opt == "2":
            try:
                passenger['age'] = int(input("New age: "))
            except:
                print("Invalid age!")
        elif opt == "3":
            passenger['gender'] = input("New gender: ").upper()
        else:
            print("Invalid choice!")
        print("Passenger updated!")
    
    elif choice == "5":
        print("Edit cancelled!")
    else:
        print("Invalid choice!")


# ============================================
# CANCEL BOOKING
# ============================================

def cancel_booking(email):
    if email not in bookings or not bookings[email]:
        print("No bookings to cancel!")
        return
    
    view_tickets(email)
    
    try:
        num = int(input("\nEnter booking number to cancel: "))
        if num < 1 or num > len(bookings[email]):
            print("Invalid number!")
            return
        num = num - 1
    except:
        print("Invalid input!")
        return
    
    confirm = input("Are you sure? (y/n): ").lower()
    
    if confirm == 'y':
        cancelled = bookings[email].pop(num)
        
        # Delete from database
        if cancelled.get('booking_id'):
            conn = get_db_connection()
            if conn:
                cursor = conn.cursor()
                try:
                    cursor.execute("DELETE FROM bookings WHERE booking_id=?", (cancelled['booking_id'],))
                    cursor.execute("DELETE FROM passengers WHERE booking_id=?", (cancelled['booking_id'],))
                    conn.commit()
                except:
                    pass
                cursor.close()
                conn.close()
        
        print("Booking cancelled successfully!")
        print("Refund for", len(cancelled['passengers']), "passengers.")
    else:
        print("Cancellation aborted!")


# ============================================
# RUN PROGRAM
# ============================================

if __name__ == "__main__":
    main()
 
