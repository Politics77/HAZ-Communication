# app.py
from flask import Flask, render_template, request, redirect, url_for, session
from flask_socketio import SocketIO, send
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
app.config['SECRET_KEY'] = 'votre_cle_secrete'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///haz.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
socketio = SocketIO(app)

# Modèle Utilisateur
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False)
    is_admin = db.Column(db.Boolean, default=False)

# Modèle Message
class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.String(500), nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    timestamp = db.Column(db.DateTime, server_default=db.func.now())

@app.route('/')
def home():
    if 'user_id' in session:
        return redirect(url_for('chat'))
    return redirect(url_for('login'))

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        email = request.form['email']
        password = generate_password_hash(request.form['password'])
        
        new_user = User(username=username, email=email, password=password)
        db.session.add(new_user)
        db.session.commit()
        
        return redirect(url_for('login'))
    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form['email']
        password = request.form['password']
        
        user = User.query.filter_by(email=email).first()
        
        if user and check_password_hash(user.password, password):
            session['user_id'] = user.id
            session['username'] = user.username
            return redirect(url_for('chat'))
        
        return "Identifiants invalides"
    return render_template('login.html')

@app.route('/chat')
def chat():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    messages = Message.query.order_by(Message.timestamp).all()
    return render_template('chat.html', messages=messages)

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

@socketio.on('message')
def handle_message(msg):
    user_id = session.get('user_id')
    if user_id:
        user = User.query.get(user_id)
        new_message = Message(content=msg, user_id=user_id)
        db.session.add(new_message)
        db.session.commit()
        
        send({'username': user.username, 'message': msg}, broadcast=True)

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    socketio.run(app, debug=True)<!-- register.html -->
<!DOCTYPE html>
<html>
<head>
    <title>HAZ Communication - Inscription</title>
</head>
<body>
    <h2>Inscription</h2>
    <form method="POST">
        <input type="text" name="username" placeholder="Nom d'utilisateur" required>
        <input type="email" name="email" placeholder="Email" required>
        <input type="password" name="password" placeholder="Mot de passe" required>
        <button type="submit">S'inscrire</button>
    </form>
</body>
</html><!-- login.html -->
<!DOCTYPE html>
<html>
<head>
    <title>HAZ Communication - Connexion</title>
</head>
<body>
    <h2>Connexion</h2>
    <form method="POST">
        <input type="email" name="email" placeholder="Email" required>
        <input type="password" name="password" placeholder="Mot de passe" required>
        <button type="submit">Se connecter</button>
    </form>
</body>
</html><!-- chat.html -->
<!DOCTYPE html>
<html>
<head>
    <title>HAZ Communication</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.0.1/socket.io.js"></script>
    <script>
        var socket = io.connect('http://' + document.domain + ':' + location.port);
        
        socket.on('connect', function() {
            console.log('Connected to chat');
        });

        socket.on('message', function(data) {
            var messages = document.getElementById('messages');
            messages.innerHTML += '<p><strong>' + data.username + ':</strong> ' + data.message + '</p>';
        });

        function sendMessage() {
            var input = document.getElementById('message_input');
            socket.send(input.value);
            input.value = '';
        }
    </script>
</head>
<body>
    <h1>HAZ Communication - Chat</h1>
    <div id="messages">
        {% for message in messages %}
            <p><strong>{{ message.user.username }}:</strong> {{ message.content }}</p>
        {% endfor %}
    </div>
    <input type="text" id="message_input" placeholder="Votre message">
    <button onclick="sendMessage()">Envoyer</button>
    <a href="{{ url_for('logout') }}">Déconnexion</a>
</body>
</html>pip install flask flask-sqlalchemy flask-socketio werkzeugpython app.py
