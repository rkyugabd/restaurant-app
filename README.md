# restaurant-app from flask import Flask, request, send_from_directory
from fpdf import FPDF
import os, json
from datetime import datetime

app = Flask(__name__)

# ================== 基础目录 ==================
DATA_DIR = "data"
PDF_DIR = "pdf_records"
ORDERS_DIR = "orders_data"
os.makedirs(DATA_DIR, exist_ok=True)
os.makedirs(PDF_DIR, exist_ok=True)
os.makedirs(ORDERS_DIR, exist_ok=True)

# ================== 配置 ==================
DEPARTMENTS = {
    "hot_dishes": "Hot Dishes",
    "dessert": "Dessert",
    "bread": "Bread",
    "cheese": "Cheese",
    "supermarket": "Supermarket"
}

TABLES = {
    "used_materials": ["Date", "Employee Name", "Material Name", "Quantity", "Notes"],
    "production": ["Date", "Employee Name", "Product / Dish Name", "Quantity", "Notes"],
    "transfer": ["Date", "Employee Name", "Transfer To Dept", "Item Name", "Quantity", "Notes"]
}

# ================== 工具函数 ==================
def today():
    return datetime.now().strftime("%Y-%m-%d")

def json_file(dep, table):
    return f"{DATA_DIR}/{dep}_{table}_{today()}.json"

def read_data(dep, table):
    if os.path.exists(json_file(dep, table)):
        with open(json_file(dep, table), "r", encoding="utf-8") as f:
            return json.load(f)
    return []

def save_data(dep, table, data):
    with open(json_file(dep, table), "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2)

def generate_pdf(dep_key, dep_name, table_key):
    data = read_data(dep_key, table_key)
    if not data:
        return None

    headers = TABLES[table_key]
    now = datetime.now()

    pdf = FPDF(orientation="L")
    pdf.add_page()
    pdf.set_font("Arial", "B", 14)
    pdf.cell(0, 10, f"{dep_name} - {table_key.replace('_',' ').title()}", ln=True)
    pdf.set_font("Arial", "", 12)
    pdf.cell(0, 10, f"Date: {today()}  Time: {now.strftime('%H:%M:%S')}", ln=True)
    pdf.ln(5)

    col_width = 280 // len(headers)
    row_h = 8

    pdf.set_font("Arial", "B", 11)
    for h in headers:
        pdf.cell(col_width, row_h, h, border=1)
    pdf.ln(row_h)

    pdf.set_font("Arial", "", 11)
    for row in data:
        for h in headers:
            pdf.cell(col_width, row_h, str(row.get(h, "")), border=1)
        pdf.ln(row_h)

    filename = f"{dep_name}_{table_key}_{now.strftime('%Y%m%d_%H-%M-%S')}.pdf"
    path = f"{PDF_DIR}/{filename}"
    pdf.output(path)
    return filename

# ================== 主页 ==================
@app.route("/")
def home():
    return """
    <h1>Angelo App</h1>
    <ul>
        <li><a href="/daily_record">Daily Record</a></li>
        <li><a href="/recipes">Recipe Library</a></li>
        <li><a href="/training">Employee Training</a></li>
        <li><a href="/schedule">Employee Schedule</a></li>
        <li><a href="/orders">Orders</a></li>
        <li><a href="/boss">Boss Login</a></li>
    </ul>
    """

# ================== Daily Record ==================
@app.route("/daily_record")
def daily_record():
    html = "<h1>Daily Record</h1><ul>"
    for k, v in DEPARTMENTS.items():
        html += f'<li><a href="/daily_record/{k}">{v}</a></li>'
    html += "</ul><a href='/'>Back Home</a>"
    return html

@app.route("/daily_record/<dep>")
def department(dep):
    if dep not in DEPARTMENTS:
        return "Invalid department"

    html = f"<h2>{DEPARTMENTS[dep]}</h2><ul>"
    for t in TABLES:
        html += f'<li><a href="/daily_record/{dep}/{t}">{t.replace("_"," ").title()}</a></li>'
    html += "</ul><a href='/daily_record'>Back</a>"
    return html

@app.route("/daily_record/<dep>/<table>", methods=["GET", "POST"])
def table_page(dep, table):
    if dep not in DEPARTMENTS or table not in TABLES:
        return "Invalid page"

    headers = TABLES[table]
    data = read_data(dep, table)
    msg = ""

    if request.method == "POST" and "save" in request.form:
        row = {h: request.form.get(h, "") for h in headers}
        data.append(row)
        save_data(dep, table, data)
        msg = "Saved successfully."

    html = f"<h2>{DEPARTMENTS[dep]} - {table.replace('_',' ').title()}</h2>"
    html += f"<p style='color:green'>{msg}</p>"
    html += "<form method='post'><table border='1' cellpadding='5'><tr>"

    for h in headers:
        html += f"<th>{h}</th>"
    html += "</tr>"

    for r in data:
        html += "<tr>"
        for h in headers:
            html += f"<td>{r.get(h,'')}</td>"
        html += "</tr>"

    html += "<tr>"
    for h in headers:
        html += f"<td><input name='{h}'></td>"
    html += "</tr></table><br>"

    html += "<button name='save'>Save</button></form>"

    html += f"""
    <form method="post" action="/daily_record/{dep}/{table}/pdf">
        <button>Generate PDF</button>
    </form>
    """

    html += "<h3>History PDFs</h3><ul>"
    for f in sorted(os.listdir(PDF_DIR), reverse=True):
        if f.startswith(DEPARTMENTS[dep]) and table in f:
            html += f'<li><a href="/pdf/{f}" target="_blank">{f}</a></li>'
    html += f"</ul><a href='/daily_record/{dep}'>Back</a>"

    return html

@app.route("/daily_record/<dep>/<table>/pdf", methods=["POST"])
def make_pdf(dep, table):
    filename = generate_pdf(dep, DEPARTMENTS[dep], table)
    if not filename:
        return f"<p>No data to generate PDF.</p><a href='/daily_record/{dep}/{table}'>Back</a>"

    return f"""
    <h3>PDF Generated</h3>
    <a href="/pdf/{filename}" target="_blank">Download PDF</a><br>
    <a href="/daily_record/{dep}/{table}">Back</a>
    """

@app.route("/pdf/<filename>")
def pdf(filename):
    return send_from_directory(PDF_DIR, filename)

# ================== Recipes ==================
@app.route("/recipes")
def recipes():
    recipe_folder = "recipes"
    os.makedirs(recipe_folder, exist_ok=True)

    search = request.args.get("search", "").lower()
    files = sorted(os.listdir(recipe_folder))

    if search:
        files = [f for f in files if search in f.lower()]

    html = f"""
    <h1>Recipe Library</h1>
    <form method="get">
        <input type="text" name="search" placeholder="Search recipe name"
               value="{search}">
        <button type="submit">Search</button>
    </form>
    <br>
    <ul>
    """
    for f in files:
        if f.lower().endswith(".pdf"):
            html += f'<li><a href="/recipes/{f}" target="_blank">{f.replace("_"," ").replace(".pdf","")}</a></li>'
    html += "</ul><a href='/'>Back Home</a>"
    return html

@app.route("/recipes/<filename>")
def recipe_file(filename):
    return send_from_directory("recipes", filename)

# ================== Training ==================
@app.route("/training", methods=["GET", "POST"])
def training():
    folder = "training_pdfs"
    os.makedirs(folder, exist_ok=True)

    if request.method == "POST":
        file = request.files.get("pdf")
        if file and file.filename.lower().endswith(".pdf"):
            filename = file.filename.replace(" ", "_")
            file.save(os.path.join(folder, filename))

    files = sorted(os.listdir(folder))

    html = """
    <h1>Employee Training</h1>
    <h3>Upload Training PDF</h3>
    <form method="post" enctype="multipart/form-data">
        <input type="file" name="pdf" accept=".pdf">
        <button type="submit">Upload</button>
    </form>
    <h3>Training Documents</h3>
    <ul>
    """
    for f in files:
        if f.lower().endswith(".pdf"):
            html += f'<li><a href="/training/{f}" target="_blank">{f.replace("_"," ").replace(".pdf","")}</a></li>'
    html += "</ul><a href='/'>Back Home</a>"
    return html

@app.route("/training/<filename>")
def training_pdf(filename):
    return send_from_directory("training_pdfs", filename)

# ================== Schedule ==================
@app.route("/schedule")
def schedule():
    html = "<h1>Employee Schedule</h1><ul>"
    for k, v in DEPARTMENTS.items():
        html += f'<li><a href="/schedule/{k}">{v}</a></li>'
    html += "</ul><a href='/'>Back Home</a>"
    return html

@app.route("/schedule/<dep>", methods=["GET", "POST"])
def schedule_department(dep):
    if dep not in DEPARTMENTS:
        return "Invalid department"

    base_folder = "schedule_pdfs"
    dep_folder = os.path.join(base_folder, dep)
    os.makedirs(dep_folder, exist_ok=True)

    if request.method == "POST":
        file = request.files.get("pdf")
        if file and file.filename.lower().endswith(".pdf"):
            filename = file.filename.replace(" ", "_")
            file.save(os.path.join(dep_folder, filename))

    files = sorted(os.listdir(dep_folder), reverse=True)

    html = f"""
    <h2>{DEPARTMENTS[dep]} - Schedule</h2>
    <h3>Upload Schedule PDF</h3>
    <form method="post" enctype="multipart/form-data">
        <input type="file" name="pdf" accept=".pdf">
        <button type="submit">Upload</button>
    </form>
    <h3>Schedule Files</h3>
    <ul>
    """
    for f in files:
        if f.lower().endswith(".pdf"):
            html += f'<li><a href="/schedule/{dep}/{f}" target="_blank">{f.replace("_"," ").replace(".pdf","")}</a></li>'
    html += f"</ul><a href='/schedule'>Back to Departments</a>"
    return html

@app.route("/schedule/<dep>/<filename>")
def schedule_pdf(dep, filename):
    if dep not in DEPARTMENTS:
        return "Invalid department"
    return send_from_directory(os.path.join("schedule_pdfs", dep), filename)

# ================== Boss ==================
@app.route("/boss")
def boss():
    return "<h1>Boss Login</h1><a href='/'>Back</a>"

# ================== ORDERS MODULE ==================
import json
from flask import request, send_from_directory
from fpdf import FPDF
import os
from datetime import datetime

ORDERS_DATA_DIR = "orders_data"
ORDERS_PDF_DIR = "orders_pdf"
os.makedirs(ORDERS_DATA_DIR, exist_ok=True)
os.makedirs(ORDERS_PDF_DIR, exist_ok=True)

DEPARTMENTS = {
    "hot_dishes": "Hot Dishes",
    "dessert": "Dessert",
    "bread": "Bread",
    "cheese": "Cheese",
    "supermarket": "Supermarket"
}

# ========== 工具函数 ==========
def today_str():
    return datetime.now().strftime("%Y-%m-%d")

def generate_order_number():
    date = today_str()
    # 读取当天所有订单数量
    orders_file = f"{ORDERS_DATA_DIR}/orders_{date}.json"
    if os.path.exists(orders_file):
        with open(orders_file, "r", encoding="utf-8") as f:
            orders = json.load(f)
        count = len(orders) + 1
    else:
        count = 1
        orders = []
    return f"{date}-{count:03d}", orders_file, orders

def save_orders(orders_file, orders):
    with open(orders_file, "w", encoding="utf-8") as f:
        json.dump(orders, f, indent=2)

def generate_order_pdf(order):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", "B", 14)
    pdf.cell(0, 10, f"Order #{order['order_number']} - {order['customer_name']}", ln=True)
    pdf.set_font("Arial", "", 12)
    pdf.ln(5)
    for key in ["reservation_time", "pickup_time", "payment_status", "notes"]:
        pdf.cell(0, 8, f"{key.replace('_',' ').title()}: {order.get(key,'')}", ln=True)
    pdf.ln(5)
    pdf.set_font("Arial", "B", 12)
    pdf.cell(0, 8, "Ordered Items:", ln=True)
    pdf.set_font("Arial", "", 12)
    pdf.ln(3)
    pdf.cell(60, 8, "Department", border=1)
    pdf.cell(80, 8, "Dish", border=1)
    pdf.cell(20, 8, "Qty", border=1)
    pdf.ln()
    for dept, items in order.get("items_by_dept", {}).items():
        for dish, qty in items:
            pdf.cell(60, 8, DEPARTMENTS[dept], border=1)
            pdf.cell(80, 8, dish, border=1)
            pdf.cell(20, 8, str(qty), border=1)
            pdf.ln()
    filename = f"{ORDERS_PDF_DIR}/{order['order_number']}_{order['customer_name'].replace(' ','_')}.pdf"
    pdf.output(filename)
    return filename

# ========== ROUTES ==========
@app.route("/orders")
def orders_home():
    return """
    <h1>Orders Dashboard</h1>
    <ul>
        <li><a href='/orders/new'>New Orders</a></li>
        <li><a href='/orders/pending'>Pending Orders</a></li>
        <li><a href='/orders/completed'>Completed Orders</a></li>
    </ul>
    <a href='/'>Back Home</a>
    """

# ----- New Orders -----
@app.route("/orders/new", methods=["GET", "POST"])
def orders_new():
    if request.method == "POST":
        # 获取表单
        customer_name = request.form.get("customer_name", "")
        reservation_time = request.form.get("reservation_time", "")
        pickup_time = request.form.get("pickup_time", "")
        payment_status = request.form.get("payment_status", "")
        notes = request.form.get("notes", "")
        dishes = request.form.getlist("dish")
        qtys = request.form.getlist("qty")
        depts = request.form.getlist("dept")

        # 整理 items_by_dept
        items_by_dept = {}
        for dept, dish, qty in zip(depts, dishes, qtys):
            if not dish.strip():
                continue
            items_by_dept.setdefault(dept, []).append((dish.strip(), int(qty)))

        order_number, orders_file, orders = generate_order_number()
        order = {
            "order_number": order_number,
            "customer_name": customer_name,
            "reservation_time": reservation_time,
            "pickup_time": pickup_time,
            "payment_status": payment_status,
            "notes": notes,
            "items_by_dept": items_by_dept,
            "status": "pending"
        }
        orders.append(order)
        save_orders(orders_file, orders)
        pdf_file = generate_order_pdf(order)
        return f"""
        <h3>Order Confirmed</h3>
        <p>Order Number: {order_number}</p>
        <a href='/orders/pdf/{pdf_file.split('/')[-1]}' target='_blank'>Download PDF</a><br>
        <a href='/orders/new'>Add Another Order</a><br>
        <a href='/orders/pending'>Go to Pending Orders</a>
        """

    # GET 表单
    html = """
    <h2>New Order</h2>
    <form method='post'>
        Customer Name: <input name='customer_name' required><br>
        Reservation Time: <input name='reservation_time' type='time'><br>
        Pickup Time: <input name='pickup_time' type='time'><br>
        Payment Status: <input name='payment_status' placeholder='Paid / Pending'><br>
        Notes: <input name='notes'><br><br>
    """

    # 可以填写多个菜
    for i in range(5):
        html += f"""
        Dish {i+1}: <input name='dish'>
        Quantity: <input name='qty' type='number' min='1' value='1'>
        Department: 
        <select name='dept'>
            <option value='hot_dishes'>Hot Dishes</option>
            <option value='dessert'>Dessert</option>
            <option value='bread'>Bread</option>
            <option value='cheese'>Cheese</option>
            <option value='supermarket'>Supermarket</option>
        </select><br>
        """
    html += "<br><button>Confirm Order</button></form>"
    html += "<a href='/orders'>Back</a>"
    return html

# ----- Pending Orders -----
@app.route("/orders/pending")
def orders_pending():
    today_file = f"{ORDERS_DATA_DIR}/orders_{today_str()}.json"
    if os.path.exists(today_file):
        with open(today_file, "r", encoding="utf-8") as f:
            orders = json.load(f)
    else:
        orders = []

    html = "<h2>Pending Orders</h2><ul>"
    for order in orders:
        if order["status"] == "pending":
            html += f"<li>Order #{order['order_number']} - {order['customer_name']} "
            html += f"<a href='/orders/mark_done/{order['order_number']}'>Mark Done</a></li>"
    html += "</ul><a href='/orders'>Back</a>"
    return html

# ----- Completed Orders -----
@app.route("/orders/completed")
def orders_completed():
    today_file = f"{ORDERS_DATA_DIR}/orders_{today_str()}.json"
    if os.path.exists(today_file):
        with open(today_file, "r", encoding="utf-8") as f:
            orders = json.load(f)
    else:
        orders = []

    html = "<h2>Completed Orders</h2><ul>"
    for order in orders:
        if order["status"] == "done":
            pdf_file = generate_order_pdf(order)
            html += f"<li>Order #{order['order_number']} - {order['customer_name']} "
            html += f"<a href='/orders/pdf/{pdf_file.split('/')[-1]}' target='_blank'>PDF</a></li>"
    html += "</ul><a href='/orders'>Back</a>"
    return html

# ----- Mark Done -----
@app.route("/orders/mark_done/<order_number>")
def orders_mark_done(order_number):
    today_file = f"{ORDERS_DATA_DIR}/orders_{today_str()}.json"
    if os.path.exists(today_file):
        with open(today_file, "r", encoding="utf-8") as f:
            orders = json.load(f)
    else:
        orders = []

    for order in orders:
        if order["order_number"] == order_number:
            order["status"] = "done"
            break
    save_orders(today_file, orders)
    return f"<p>Order {order_number} marked as done.</p><a href='/orders/pending'>Back to Pending</a>"

# ----- PDF Download -----
@app.route("/orders/pdf/<filename>")
def orders_pdf(filename):
    return send_from_directory(ORDERS_PDF_DIR, filename)

# ================== Run ==================
if __name__ == "__main__":
    app.run(debug=True)
