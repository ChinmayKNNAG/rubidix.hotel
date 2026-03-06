# rubidix.hotel
from flask import Flask, render_template, request, redirect, url_for, session, flash
from datetime import datetime

app = Flask(__name__)
app.secret_key = "super-secret-key-123"   # needed for session & flash messages

# ---------------- MENU (same as your code) ----------------
menu = {
    "Tea": 15,
    "Coffee": 25,
    "Idli": 40,
    "Dosa": 50,
    "Vada": 30,
    "Poori": 45,
    "Meals": 120
}

# For simplicity — we store cart in session (per browser tab)
# In real app → you would use database + user login

# ---------------- Helper functions ----------------
def calculate_totals(cart):
    if not cart:
        return 0, 0, 0, 0

    total = sum(menu[item] * qty for item, qty in cart.items())
    gst = total * 0.18
    discount = (total + gst) * 0.05
    net = total + gst - discount

    return total, gst, discount, net


# ---------------- Routes ----------------

@app.route("/")
def home():
    return render_template("home.html")


@app.route("/customer-details", methods=["GET", "POST"])
def customer_details():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        phone = request.form.get("phone", "").strip()

        if not name or not phone:
            flash("Please enter both name and phone number.", "error")
            return redirect(url_for("customer_details"))

        session["customer_name"] = name
        session["customer_phone"] = phone
        session["cart"] = {}   # start fresh cart

        return redirect(url_for("menu"))

    return render_template("customer_details.html")


@app.route("/menu")
def menu():
    if "cart" not in session:
        return redirect(url_for("home"))

    return render_template("menu.html", menu=menu)


@app.route("/order", methods=["POST"])
def order():
    if "cart" not in session:
        flash("Session expired. Please start again.", "error")
        return redirect(url_for("home"))

    cart = session.get("cart", {})

    for item in menu:
        qty_str = request.form.get(f"qty_{item}", "0").strip()
        try:
            qty = int(qty_str)
            if qty > 0:
                cart[item] = cart.get(item, 0) + qty
        except ValueError:
            pass  # ignore invalid input

    session["cart"] = cart
    flash("Items added to cart!", "success")

    return redirect(url_for("bill"))


@app.route("/bill")
def bill():
    if "cart" not in session:
        return redirect(url_for("home"))

    cart = session.get("cart", {})
    total, gst, discount, net = calculate_totals(cart)

    return render_template("bill.html",
                           cart=cart,
                           menu=menu,
                           total=total,
                           gst=gst,
                           discount=discount,
                           net=net)


@app.route("/payment", methods=["GET", "POST"])
def payment():
    if "cart" not in session or not session["cart"]:
        flash("Cart is empty.", "error")
        return redirect(url_for("home"))

    if request.method == "POST":
        payment_mode = request.form.get("payment_mode")
        if not payment_mode:
            flash("Please select a payment method.", "error")
            return redirect(url_for("payment"))

        # Here you could save order to database, generate PDF bill, etc.
        # For now — just show thank you

        session.pop("cart", None)           # clear cart
        session.pop("customer_name", None)
        session.pop("customer_phone", None)

        flash(f"Thank you for your order! Paid via {payment_mode} 🙏", "success")
        return redirect(url_for("thank_you"))

    return render_template("payment.html")


@app.route("/thank-you")
def thank_you():
    return render_template("thank_you.html")


@app.route("/reset")
def reset():
    session.clear()
    return redirect(url_for("home"))


if __name__ == "__main__":
    app.run(debug=True)