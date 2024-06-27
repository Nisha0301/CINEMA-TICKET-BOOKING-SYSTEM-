# CINEMA-TICKET-BOOKING-SYSTEM-
import sqlite3
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime

class Movie:
    def __init__(self, movie_id, title, duration, price):
        self.movie_id = movie_id
        self.title = title
        self.duration = duration
        self.price = price

class Booking:
    GST_RATE = 0.18
    
    def __init__(self, booking_id, movie, customer_email, quantity, booking_time):
        self.booking_id = booking_id
        self.movie = movie
        self.customer_email = customer_email
        self.quantity = quantity
        self.booking_time = booking_time
        self.total_cost = self.calculate_total_cost()

    def calculate_total_cost(self):
        base_cost = self.movie.price * self.quantity
        gst = base_cost * Booking.GST_RATE
        return base_cost + gst

    def send_email_confirmation(self):
        sender_email = "nis@123example.com"
        receiver_email = self.customer_email
        password = "your_password"

        message = MIMEMultipart()
        message["From"] = sender_email
        message["To"] = receiver_email
        message["Subject"] = f"Booking Confirmation for {self.movie.title}"

        body = f"""
        Thank you for your booking!

        Movie: {self.movie.title}
        Duration: {self.movie.duration} mins
        Quantity: {self.quantity}
        Total Cost: ${self.total_cost:.2f}
        Booking Time: {self.booking_time}

        Enjoy your movie!
        """
        message.attach(MIMEText(body, "plain"))

        try:
            server = smtplib.SMTP("smtp.example.com", 587)
            server.starttls()
            server.login(sender_email, password)
            server.sendmail(sender_email, receiver_email, message.as_string())
            server.close()
            print("Email sent successfully!")
        except Exception as e:
            print(f"Error sending email: {e}")

class Cinema:
    def __init__(self, db_name="cinema.db"):
        self.conn = sqlite3.connect(db_name)
        self.create_tables()

    def create_tables(self):
        with self.conn:
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS movies (
                    movie_id INTEGER PRIMARY KEY,
                    title TEXT NOT NULL,
                    duration INTEGER NOT NULL,
                    price REAL NOT NULL
                )
            """)
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS bookings (
                    booking_id INTEGER PRIMARY KEY,
                    movie_id INTEGER,
                    customer_email TEXT NOT NULL,
                    quantity INTEGER NOT NULL,
                    total_cost REAL NOT NULL,
                    booking_time TEXT NOT NULL,
                    FOREIGN KEY (movie_id) REFERENCES movies(movie_id)
                )
            """)

    def add_movie(self, title, duration, price):
        with self.conn:
            self.conn.execute("""
                INSERT INTO movies (title, duration, price)
                VALUES (?, ?, ?)
            """, (title, duration, price))

    def get_movies(self):
        with self.conn:
            return self.conn.execute("SELECT * FROM movies").fetchall()

    def book_tickets(self, movie_id, customer_email, quantity):
        movie = self.conn.execute("SELECT * FROM movies WHERE movie_id = ?", (movie_id,)).fetchone()
        if movie:
            movie_obj = Movie(*movie)
            booking_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            booking = Booking(None, movie_obj, customer_email, quantity, booking_time)
            with self.conn:
                self.conn.execute("""
                    INSERT INTO bookings (movie_id, customer_email, quantity, total_cost, booking_time)
                    VALUES (?, ?, ?, ?, ?)
                """, (movie_id, customer_email, quantity, booking.total_cost, booking_time))
            booking.send_email_confirmation()
            self.log_booking(booking)
        else:
            print("Movie not found")

    def log_booking(self, booking):
        with open("bookings.log", "a") as log_file:
            log_file.write(f"Booking ID: {booking.booking_id}, Movie: {booking.movie.title}, "
                           f"Customer: {booking.customer_email}, Quantity: {booking.quantity}, "
                           f"Total Cost: {booking.total_cost:.2f}, Booking Time: {booking.booking_time}\n")


cinema = Cinema()
cinema.add_movie("Movie A", 120, 10.0)
cinema.add_movie("Movie B", 90, 8.0)
print(cinema.get_movies())


cinema.book_tickets(1, "customer@example.com", 2)
