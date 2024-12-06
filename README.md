# API-_-IRCTC
pip install flask flask-mysql flask-jwt-extended flask-bcrypt

from flask import Flask, request, jsonify
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
from flask_bcrypt import Bcrypt
from flask_mysqldb import MySQL
import MySQLdb.cursors

app = Flask(__name__)
app.config['MYSQL_HOST'] = 'localhost'
app.config['MYSQL_USER'] = 'root'
app.config['MYSQL_PASSWORD'] = 'password'
app.config['MYSQL_DB'] = 'railway_system'
app.config['JWT_SECRET_KEY'] = 'your_secret_key'

mysql = MySQL(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)


@app.route('/admin/train', methods=['POST'])
@jwt_required()
def add_train():

    current_user = get_jwt_identity()
    if current_user['role'] != 'admin':
        return jsonify({'message': 'Access denied'}), 403
    
    data = request.json
    cursor = mysql.connection.cursor()
    cursor.execute(
        "INSERT INTO Trains (train_name, source, destination, total_seats, departure_time, arrival_time) "
        "VALUES (%s, %s, %s, %s, %s, %s)",
        (data['train_name'], data['source'], data['destination'], data['total_seats'], 
         data['departure_time'], data['arrival_time'])
    )
    mysql.connection.commit()
    cursor.close()
    return jsonify({'message': 'Train added successfully'}), 201


@app.route('/bookings', methods=['POST'])
@jwt_required()
def book_seat():
    current_user = get_jwt_identity()
    if current_user['role'] != 'user':
        return jsonify({'message': 'Access denied'}), 403

    data = request.json
    train_id = data['train_id']
    seat_count = data['seat_count']

    cursor = mysql.connection.cursor()
    try:
        cursor.execute("SELECT available_seats FROM Train_Seats WHERE train_id = %s FOR UPDATE", (train_id,))
        seat_row = cursor.fetchone()
        if not seat_row or seat_row[0] < seat_count:
            return jsonify({'message': 'Not enough seats available'}), 400

        updated_seats = seat_row[0] - seat_count
        cursor.execute("UPDATE Train_Seats SET available_seats = %s WHERE train_id = %s", (updated_seats, train_id))
        cursor.execute(
            "INSERT INTO Bookings (user_id, train_id, booking_time, seat_count) VALUES (%s, %s, NOW(), %s)",
            (current_user['id'], train_id, seat_count)
        )
        mysql.connection.commit()
        return jsonify({'message': 'Booking successful'}), 201
    except Exception as e:
        mysql.connection.rollback()
        return jsonify({'message': 'Booking failed', 'error': str(e)}), 500
    finally:
        cursor.close()
if __name__ == '__main__':
    app.run(debug=True)
