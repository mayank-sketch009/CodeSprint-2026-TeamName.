from flask import Flask, render_template, request, redirect
import sqlite3
import os
from werkzeug.utils import secure_filename

app = Flask(_name_)

app.config["UPLOAD_FOLDER"] = "uploads"
app.config["MAX_CONTENT_LENGTH"] = 5 * 1024 * 1024

os.makedirs("uploads", exist_ok=True)

ALLOWED_EXTENSIONS = {"png", "jpg", "jpeg"}

def get_db():
    conn = sqlite3.connect("database.db")
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db()
    conn.execute("""
        CREATE TABLE IF NOT EXISTS complaints (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            category TEXT,
            description TEXT,
            location TEXT,
            image TEXT,
            status TEXT DEFAULT 'Pending'
        )
    """)
    conn.commit()
    conn.close()

def allowed_file(filename):
    return (
        "." in filename and
        filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS
    )

@app.route("/")
def home():
    conn = get_db()
    total = conn.execute(
        "SELECT COUNT(*) FROM complaints"
    ).fetchone()[0]

    pending = conn.execute(
        "SELECT COUNT(*) FROM complaints WHERE status='Pending'"
    ).fetchone()[0]

    resolved = conn.execute(
        "SELECT COUNT(*) FROM complaints WHERE status='Resolved'"
    ).fetchone()[0]

    conn.close()

    return render_template(
        "index.html",
        total=total,
        pending=pending,
        resolved=resolved
    )

@app.route("/report", methods=["GET", "POST"])
def report():
    if request.method == "POST":
        category = request.form["category"]
        description = request.form["description"]
        location = request.form["location"]

        file = request.files.get("image")
        filename = ""

        if file and file.filename:
            if not allowed_file(file.filename):
                return "Only JPG, JPEG and PNG files are allowed", 400

            filename = secure_filename(file.filename)
            file.save(os.path.join(
                app.config["UPLOAD_FOLDER"], filename
            ))

        conn = get_db()
        conn.execute("""
            INSERT INTO complaints
            (category, description, location, image)
            VALUES (?, ?, ?, ?)
        """, (category, description, location, filename))

        conn.commit()
        conn.close()

        return redirect("/complaints")

    return render_template("report.html")

@app.route("/complaints")
def complaints():
    conn = get_db()
    data = conn.execute(
        "SELECT * FROM complaints ORDER BY id DESC"
    ).fetchall()
    conn.close()

    return render_template("complaints.html", complaints=data)

@app.route("/admin", methods=["GET", "POST"])
def admin():
    conn = get_db()

    if request.method == "POST":
        complaint_id = request.form["complaint_id"]
        status = request.form["status"]

        if status not in ["Pending", "Assigned", "Resolved"]:
            conn.close()
            return "Invalid status", 400

        conn.execute(
            "UPDATE complaints SET status=? WHERE id=?",
            (status, complaint_id)
        )
        conn.commit()

    data = conn.execute(
        "SELECT * FROM complaints ORDER BY id DESC"
    ).fetchall()

    conn.close()
    return render_template("admin.html", complaints=data)

if _name_ == "_main_":
    init_db()
    app.run(debug=True)
   # CodeSprint-2026-TeamName.
