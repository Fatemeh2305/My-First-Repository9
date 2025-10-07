my_fastapi_app/
│
├── main.py
│   └──
        from fastapi import FastAPI, Request, Depends, Forms
        from fastapi.responses import HTMLResponse, RedirectResponse
        from fastapi.templating import Jinja2Templates
        from sqlalchemy.orm import Session
        from database import SessionLocal, engine
       
        from auth import get_current_user, register_user, login_user
        from models import Message

        models.Base.metadata.create_all(bind=engine)

        app = FastAPI()
        templates = Jinja2Templates(directory="templates")

        def get_db():
            db = SessionLocal()
            try:
                yield db
            finally:
                db.close()

        @app.get("/", response_class=HTMLResponse)
        def read_home(request: Request, user: str = Depends(get_current_user)):
            return templates.TemplateResponse("index.html", {"request": request, "user": user})

        @app.get("/register", response_class=HTMLResponse)
        def register_form(request: Request):
            return templates.TemplateResponse("register.html", {"request": request})

        @app.post("/register")
        def register(username: str = Form(...), password: str = Form(...), db: Session = Depends(get_db)):
            register_user(db, username, password)
            return RedirectResponse(url="/login", status_code=303)

        @app.get("/login", response_class=HTMLResponse)
        def login_form(request: Request):
            return templates.TemplateResponse("login.html", {"request": request})

        @app.post("/login")
        def login(username: str = Form(...), password: str = Form(...), db: Session = Depends(get_db)):
            token = login_user(db, username, password)
            response = RedirectResponse(url="/", status_code=303)
            response.set_cookie(key="session", value=token)
            return response

        @app.get("/logout")
        def logout():
            response = RedirectResponse(url="/")
            response.delete_cookie("session")
            return response

        @app.get("/contact", response_class=HTMLResponse)
        def contact_form(request: Request, user: str = Depends(get_current_user)):
            return templates.TemplateResponse("contact.html", {"request": request, "user": user})

        @app.post("/contact")
        def contact(name: str = Form(...), email: str = Form(...), message: str = Form(...), db: Session = Depends(get_db)):
            msg = Message(name=name, email=email, message=message)
            db.add(msg)
            db.commit()
            return RedirectResponse(url="/contact", status_code=303)

        @app.get("/admin", response_class=HTMLResponse)
        def admin_dashboard(request: Request, db: Session = Depends(get_db), user: str = Depends(get_current_user)):
            if not user:
                return RedirectResponse(url="/login", status_code=303)
            messages = db.query(Message).all()
            return templates.TemplateResponse("admin.html", {"request": request, "user": user, "messages": messages})

├── database.py
│   └──
        from sqlalchemy import create_engine
        from sqlalchemy.ext.declarative import declarative_base
        from sqlalchemy.orm import sessionmaker

        SQLALCHEMY_DATABASE_URL = "sqlite:///./app.db"

        engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
        SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

        Base = declarative_base()

├── models.py
│   └──
        from sqlalchemy import Column, Integer, String, Text
        from database import Base

        class User(Base):
            __tablename__ = "users"
            id = Column(Integer, primary_key=True, index=True)
            username = Column(String(150), unique=True, index=True, nullable=False)
            password = Column(String(255), nullable=False)

        class Message(Base):
            __tablename__ = "messages"
            id = Column(Integer, primary_key=True, index=True)
            name = Column(String(150), nullable=False)
            email = Column(String(150), nullable=False)
            message = Column(Text, nullable=False)

├── auth.py
│   └──
        from sqlalchemy.orm import Session
        from models import User
        from werkzeug.security import generate_password_hash, check_password_hash
        from itsdangerous import URLSafeSerializer
        from fastapi import Request

        SECRET_KEY = "supersecretkey"
        serializer = URLSafeSerializer(SECRET_KEY)

        def register_user(db: Session, username: str, password: str):
            hashed = generate_password_hash(password)
            user = User(username=username, password=hashed)
            db.add(user)
            db.commit()

        def login_user(db: Session, username: str, password: str):
            user = db.query(User).filter(User.username == username).first()
            if user and check_password_hash(user.password, password):
                return serializer.dumps({"username": username})
            return ""

        def get_current_user(request: Request):
            token = request.cookies.get("session")
            if not token:
                return None
            try:
                data = serializer.loads(token)
                return data["username"]
            except:
                return None

├── templates/
│   ├── base.html
│   │   └──
                <!DOCTYPE html>
                <html lang="en">
                <head>
                  <meta charset="UTF-8">
                  <meta name="viewport" content="width=device-width, initial-scale=1.0">
                  <title>{{ user or 'FastAPI App' }}</title>
                  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
                </head>
                <body>
                <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
                  <div class="container">
                    <a class="navbar-brand" href="/">FastAPI App</a>
                    <div>
                      <a class="nav-link d-inline text-white" href="/">Home</a>
                      <a class="nav-link d-inline text-white" href="/contact">Contact</a>
                      {% if user %}
                        <a class="nav-link d-inline text-warning" href="/admin">Admin</a>
                        <a class="nav-link d-inline text-danger" href="/logout">Logout</a>
                      {% else %}
                        <a class="nav-link d-inline text-info" href="/login">Login</a>
                        <a class="nav-link d-inline text-success" href="/register">Register</a>
                      {% endif %}
                    </div>
                  </div>
                </nav>
                <div class="container mt-4">
                  {% block content %}{% endblock %}
                </div>
                </body>
                </html>
│
│   ├── index.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Welcome to the FastAPI App</h1>
                <p class="lead">Modern FastAPI + SQLAlchemy + Jinja2 app.</p>
                {% endblock %}
│
│   ├── login.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Login</h1>
                <form method="POST">
                  <div class="mb-3">
                    <label>Username</label>
                    <input type="text" name="username" class="form-control">
                  </div>
                  <div class="mb-3">
                    <label>Password</label>
                    <input type="password" name="password" class="form-control">
                  </div>
                  <button type="submit" class="btn btn-success">Login</button>
                </form>
                {% endblock %}
│
│   ├── register.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Register</h1>
                <form method="POST">
                  <div class="mb-3">
                    <label>Username</label>
                    <input type="text" name="username" class="form-control">
                  </div>
                  <div class="mb-3">
                    <label>Password</label>
                    <input type="password" name="password" class="form-control">
                  </div>
                  <button type="submit" class="btn btn-primary">Register</button>
                </form>
                {% endblock %}
│
│   ├── contact.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Contact</h1>
                <form method="POST">
                  <div class="mb-3">
                    <label>Name</label>
                    <input type="text" name="name" class="form-control">
                  </div>
                  <div class="mb-3">
                    <label>Email</label>
                    <input type="email" name="email" class="form-control">
                  </div>
                  <div class="mb-3">
                    <label>Message</label>
                    <textarea name="message" class="form-control" rows="4"></textarea>
                  </div>
                  <button type="submit" class="btn btn-primary">Send</button>
                </form>
                {% endblock %}
│
│   └── admin.html
│       └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Admin Dashboard</h1>
                {% if messages %}
                  <table class="table table-bordered">
                    <thead>
                      <tr>
                        <th>ID</th><th>Name</th><th>Email</th><th>Message</th>
                      </tr>
                    </thead>
                    <tbody>
                      {% for msg in messages %}
                      <tr>
                        <td>{{ msg.id }}</td>
                        <td>{{ msg.name }}</td>
                        <td>{{ msg.email }}</td>
                        <td>{{ msg.message }}</td>
                      </tr>
                      {% endfor %}
                    </tbody>
                  </table>
                {% else %}
                  <p>No messages found.</p>
                {% endif %}
                {% endblock %}

└── requirements.txt
    └──
        fastapi
        uvicorn
        jinja2
        sqlalchemy
        werkzeug
        itsdangerous
