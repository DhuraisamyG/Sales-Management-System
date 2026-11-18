# Sales-Management-System
#Sales Intelligence Hub is a multi-branch sales system built with MySQL, Python, and Streamlit. It tracks sales and split payments, updates received and pending amounts using triggers and generated columns, and provides a role-based dashboard with KPIs and branch analytics
# ==============================================================================
# STEP 1: ENVIRONMENT SETUP (MYSQL SERVER, PYTHON LIBRARIES & CLOUDFLARE TUNNEL)
# ==============================================================================
import os
import re
import time
import subprocess

print("⏳ Setting up MySQL server and Python environment...")
os.system("apt-get update -qq && apt-get install -y mysql-server -qq")
os.system("pip install -q streamlit mysql-connector-python pandas plotly")
os.system("wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O /usr/local/bin/cloudflared")
os.system("chmod +x /usr/local/bin/cloudflared")

os.system("service mysql start")
os.system("mysql -e \"ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root123'; FLUSH PRIVILEGES;\"")
print("✅ MySQL Server started successfully.")

# ==============================================================================
# STEP 2: DATABASE SCHEMA, TRIGGERS & SEED DATA (schema.sql)
# ==============================================================================
schema_sql = """
DROP DATABASE IF EXISTS sales_management_system;
CREATE DATABASE sales_management_system;
USE sales_management_system;

-- 1. Branches Table
CREATE TABLE branches (
    branch_id INT PRIMARY KEY AUTO_INCREMENT,
    branch_name VARCHAR(100) NOT NULL,
    branch_admin_name VARCHAR(100) NOT NULL
);

-- 2. Customer Sales Table (Main Financial Table with Generated Column)
CREATE TABLE customer_sales (
    sale_id INT PRIMARY KEY AUTO_INCREMENT,
    branch_id INT NOT NULL,
    date DATE NOT NULL,
    name VARCHAR(100) NOT NULL,
    mobile_number VARCHAR(15) UNIQUE NOT NULL,
    product_name VARCHAR(30) NOT NULL,
    gross_sales DECIMAL(12, 2) NOT NULL,
    received_amount DECIMAL(12, 2) DEFAULT 0.00,
    pending_amount DECIMAL(12, 2) GENERATED ALWAYS AS (gross_sales - received_amount) STORED,
    status ENUM('Open', 'Close') DEFAULT 'Open',
    FOREIGN KEY (branch_id) REFERENCES branches(branch_id) ON DELETE CASCADE
);

-- 3. Users Table (Super Admin & Branch Admin Roles)
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL,
    password VARCHAR(255) NOT NULL,
    branch_id INT,
    role ENUM('Super Admin', 'Admin') NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    FOREIGN KEY (branch_id) REFERENCES branches(branch_id) ON DELETE SET NULL
);

-- 4. Payment Splits Table
CREATE TABLE payment_splits (
    payment_id INT PRIMARY KEY AUTO_INCREMENT,
    sale_id INT NOT NULL,
    payment_date DATE NOT NULL,
    amount_paid DECIMAL(12, 2) NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    FOREIGN KEY (sale_id) REFERENCES customer_sales(sale_id) ON DELETE CASCADE
);

-- 5. Automation Trigger: Recalculate received_amount & update status on payment insert
DELIMITER //
CREATE TRIGGER after_payment_insert
AFTER INSERT ON payment_splits
FOR EACH ROW
BEGIN
    DECLARE total_paid DECIMAL(12, 2);
    DECLARE total_gross DECIMAL(12, 2);

    SELECT IFNULL(SUM(amount_paid), 0) INTO total_paid
    FROM payment_splits
    WHERE sale_id = NEW.sale_id;

    SELECT gross_sales INTO total_gross
    FROM customer_sales
    WHERE sale_id = NEW.sale_id;

    UPDATE customer_sales
    SET received_amount = total_paid,
        status = IF(total_paid >= total_gross, 'Close', 'Open')
    WHERE sale_id = NEW.sale_id;
END //
DELIMITER ;

-- 6. Seed Data: Branches
INSERT INTO branches (branch_name, branch_admin_name) VALUES
('Chennai', 'Arun Kumar'),
('Delhi', 'Priya Sharma'),
('Bangalore', 'Rahul Verma'),
('Mumbai', 'Vikram Desai');

-- 7. Seed Data: Users
INSERT INTO users (username, password, branch_id, role, email) VALUES
('superadmin', 'admin123', NULL, 'Super Admin', 'superadmin@saleshub.com'),
('chennai_admin', 'chennai123', 1, 'Admin', 'chennai@saleshub.com'),
('delhi_admin', 'delhi123', 2, 'Admin', 'delhi@saleshub.com'),
('bangalore_admin', 'bangalore123', 3, 'Admin', 'bangalore@saleshub.com'),
('mumbai_admin', 'mumbai123', 4, 'Admin', 'mumbai@saleshub.com');

-- 8. Seed Data: Customer Sales
INSERT INTO customer_sales (branch_id, date, name, mobile_number, product_name, gross_sales) VALUES
(1, '2026-08-12', 'Karthik Raja', '9876543210', 'DS', 55000.00),
(1, '2026-09-04', 'Divya Bharathi', '9876543211', 'DA', 42000.00),
(1, '2026-10-19', 'Suresh Raina', '9876543212', 'FSD', 68000.00),
(2, '2026-08-22', 'Sneha Gupta', '9876543213', 'FSD', 72000.00),
(2, '2026-09-15', 'Amit Verma', '9876543214', 'BA', 38000.00),
(3, '2026-10-03', 'Rohan Nair', '9876543215', 'DS', 60000.00),
(3, '2026-11-01', 'Meera Krishnan', '9876543216', 'FSD', 75000.00),
(4, '2026-11-10', 'Ananya Joshi', '9876543217', 'DA', 48000.00);

-- 9. Seed Data: Payment Splits (Triggers automatically update customer_sales)
INSERT INTO payment_splits (sale_id, payment_date, amount_paid, payment_method) VALUES
(1, '2026-08-12', 30000.00, 'UPI'),
(1, '2026-08-28', 15000.00, 'Card'),
(2, '2026-09-04', 42000.00, 'Cash'),
(3, '2026-10-19', 40000.00, 'UPI'),
(4, '2026-08-22', 50000.00, 'Card'),
(4, '2026-09-10', 22000.00, 'UPI'),
(5, '2026-09-15', 20000.00, 'Cash'),
(6, '2026-10-03', 60000.00, 'Card'),
(7, '2026-11-01', 45000.00, 'UPI'),
(8, '2026-11-10', 48000.00, 'Cash');
"""

with open("schema.sql", "w") as f:
    f.write(schema_sql)

os.system("mysql -u root -proot123 < schema.sql")
print("✅ Database, tables, triggers, and seed data loaded.")

# ==============================================================================
# STEP 3: DATABASE CONNECTION MODULE (db.py)
# ==============================================================================
db_code = """
import mysql.connector
import pandas as pd

DB_CONFIG = {
    "host": "127.0.0.1",
    "user": "root",
    "password": "root123",
    "database": "sales_management_system"
}

def get_connection():
    return mysql.connector.connect(**DB_CONFIG)

def fetch_df(query, params=None):
    conn = get_connection()
    cursor = conn.cursor(dictionary=True)
    try:
        cursor.execute(query, params or ())
        rows = cursor.fetchall()
        return pd.DataFrame(rows)
    finally:
        cursor.close()
        conn.close()

def execute_write(query, params=None):
    conn = get_connection()
    cursor = conn.cursor()
    try:
        cursor.execute(query, params or ())
        conn.commit()
        return cursor.lastrowid
    finally:
        cursor.close()
        conn.close()
"""

with open("db.py", "w") as f:
    f.write(db_code)

# ==============================================================================
# STEP 4: STREAMLIT APPLICATION (app.py)
# ==============================================================================
app_code = """
import streamlit as st
import pandas as pd
import plotly.express as px
from datetime import date
from db import fetch_df, execute_write

st.set_page_config(
    page_title="Sales Intelligence Hub",
    page_icon="📈",
    layout="wide",
    initial_sidebar_state="expanded"
)

# High-Contrast KPI Card Styling (Visible in both Dark Mode & Light Mode)
st.markdown('''
<style>
    .block-container { padding-top: 1.5rem; padding-bottom: 2rem; }
    .kpi-card {
        background-color: #1e293b;
        border-left: 5px solid #38bdf8;
        padding: 18px 20px;
        border-radius: 10px;
        box-shadow: 0 2px 6px rgba(0,0,0,0.15);
        margin-bottom: 10px;
    }
    .kpi-card.green { border-left-color: #22c55e; }
    .kpi-card.red { border-left-color: #ef4444; }
    .kpi-card.amber { border-left-color: #f59e0b; }
    .kpi-title {
        color: #cbd5e1;
        font-size: 0.9rem;
        font-weight: 600;
        text-transform: uppercase;
        letter-spacing: 0.5px;
        margin-bottom: 6px;
    }
    .kpi-value {
        color: #ffffff;
        font-size: 1.75rem;
        font-weight: 700;
        margin: 0;
    }
</style>
''', unsafe_allow_html=True)

# Session State Initialization
if "authenticated" not in st.session_state:
    st.session_state.authenticated = False
    st.session_state.user = None

# ------------------------------------------------------------------------------
# LOGIN MODULE
# ------------------------------------------------------------------------------
def render_login():
    col1, col2, col3 = st.columns([1, 1.4, 1])
    with col2:
        st.title("📈 Sales Intelligence Hub")
        st.caption("Branch-Based Sales Analytics & Financial Tracking System")
        st.divider()

        with st.form("login_form"):
            username = st.text_input("Username", placeholder="Enter your username")
            password = st.text_input("Password", type="password", placeholder="Enter your password")
            submitted = st.form_submit_button("Sign In", use_container_width=True, type="primary")

            if submitted:
                if not username or not password:
                    st.warning("Please enter both username and password.")
                else:
                    user_df = fetch_df(
                        '''SELECT u.*, b.branch_name
                           FROM users u
                           LEFT JOIN branches b ON u.branch_id = b.branch_id
                           WHERE u.username = %s AND u.password = %s''',
                        (username.strip(), password.strip())
                    )
                    if not user_df.empty:
                        st.session_state.authenticated = True
                        st.session_state.user = user_df.iloc[0].to_dict()
                        st.rerun()
                    else:
                        st.error("Invalid credentials. Please check your username and password.")

        with st.expander("🔑 Quick Demo Credentials"):
            st.markdown('''
            - **Super Admin (All Branches):** `superadmin` / `admin123`
            - **Chennai Admin:** `chennai_admin` / `chennai123`
            - **Delhi Admin:** `delhi_admin` / `delhi123`
            - **Bangalore Admin:** `bangalore_admin` / `bangalore123`
            ''')

# ------------------------------------------------------------------------------
# MAIN APPLICATION DASHBOARD
# ------------------------------------------------------------------------------
def render_app():
    user = st.session_state.user
    role = user["role"]
    assigned_branch_id = user["branch_id"]
    assigned_branch_name = user.get("branch_name") or "All Branches"

    where_clause = "" if role == "Super Admin" else f"WHERE cs.branch_id = {int(assigned_branch_id)}"

    with st.sidebar:
        st.title("📈 Sales Hub")
        st.markdown(f"**Logged in as:** `{user['username']}`")
        st.markdown(f"**Role:** `{role}`")
        st.markdown(f"**Scope:** `{assigned_branch_name}`")
        st.divider()

        nav = st.radio(
            "Navigation",
            [
                "Executive Dashboard",
                "Record New Sale",
                "Record Split Payment",
                "Pending & Payment Splits",
                "SQL Analytical Lab (20 Queries)"
            ]
        )

        st.divider()
        if st.button("🚪 Logout", use_container_width=True):
            st.session_state.authenticated = False
            st.session_state.user = None
            st.rerun()

    # ==========================================================================
    # VIEW 1: EXECUTIVE DASHBOARD
    # ==========================================================================
    if nav == "Executive Dashboard":
        st.header("📊 Executive Sales & Financial Overview")

        sales_df = fetch_df(f'''
            SELECT cs.sale_id, b.branch_name, b.branch_admin_name, cs.date,
                   cs.name AS customer_name, cs.mobile_number, cs.product_name,
                   cs.gross_sales, cs.received_amount, cs.pending_amount, cs.status
            FROM customer_sales cs
            JOIN branches b ON cs.branch_id = b.branch_id
            {where_clause}
            ORDER BY cs.date DESC
        ''')

        if sales_df.empty:
            st.info("No sales records available for your branch yet.")
            return

        for col in ["gross_sales", "received_amount", "pending_amount"]:
            sales_df[col] = sales_df[col].astype(float)

        # Interactive Filters
        f_col1, f_col2, f_col3 = st.columns(3)
        with f_col1:
            if role == "Super Admin":
                branch_options = ["All Branches"] + sorted(sales_df["branch_name"].unique().tolist())
                selected_branch = st.selectbox("Filter by Branch", branch_options)
                if selected_branch != "All Branches":
                    sales_df = sales_df[sales_df["branch_name"] == selected_branch]
            else:
                st.text_input("Assigned Branch", value=assigned_branch_name, disabled=True)

        with f_col2:
            status_filter = st.selectbox("Filter by Sale Status", ["All", "Open", "Close"])
            if status_filter != "All":
                sales_df = sales_df[sales_df["status"] == status_filter]

        with f_col3:
            prod_options = ["All Products"] + sorted(sales_df["product_name"].unique().tolist())
            prod_filter = st.selectbox("Filter by Product", prod_options)
            if prod_filter != "All Products":
                sales_df = sales_df[sales_df["product_name"] == prod_filter]

        # Financial KPIs (High-Contrast Custom HTML Cards)
        total_gross = sales_df["gross_sales"].sum()
        total_received = sales_df["received_amount"].sum()
        total_pending = sales_df["pending_amount"].sum()
        pending_pct = (total_pending / total_gross * 100) if total_gross > 0 else 0.0

        k1, k2, k3, k4 = st.columns(4)
        with k1:
            st.markdown(f'''
                <div class="kpi-card">
                    <div class="kpi-title">Total Gross Revenue</div>
                    <div class="kpi-value">₹ {total_gross:,.2f}</div>
                </div>
            ''', unsafe_allow_html=True)
        with k2:
            st.markdown(f'''
                <div class="kpi-card green">
                    <div class="kpi-title">Total Collected</div>
                    <div class="kpi-value">₹ {total_received:,.2f}</div>
                </div>
            ''', unsafe_allow_html=True)
        with k3:
            st.markdown(f'''
                <div class="kpi-card red">
                    <div class="kpi-title">Total Pending</div>
                    <div class="kpi-value">₹ {total_pending:,.2f}</div>
                </div>
            ''', unsafe_allow_html=True)
        with k4:
            st.markdown(f'''
                <div class="kpi-card amber">
                    <div class="kpi-title">Pending Collection %</div>
                    <div class="kpi-value">{pending_pct:.1f}%</div>
                </div>
            ''', unsafe_allow_html=True)

        st.divider()

        # Charts Row 1
        c1, c2 = st.columns(2)
        with c1:
            st.subheader("Branch-Wise Revenue Breakdown")
            branch_summary = sales_df.groupby("branch_name", as_index=False)[["received_amount", "pending_amount"]].sum()
            fig_branch = px.bar(
                branch_summary,
                x="branch_name",
                y=["received_amount", "pending_amount"],
                barmode="group",
                labels={"value": "Amount (₹)", "branch_name": "Branch", "variable": "Category"},
                color_discrete_map={"received_amount": "#22c55e", "pending_amount": "#ef4444"}
            )
            st.plotly_chart(fig_branch, use_container_width=True)

        with c2:
            st.subheader("Collections by Payment Method")
            pay_df = fetch_df(f'''
                SELECT ps.payment_method, SUM(ps.amount_paid) AS total_collected
                FROM payment_splits ps
                JOIN customer_sales cs ON ps.sale_id = cs.sale_id
                {where_clause}
                GROUP BY ps.payment_method
            ''')
            if not pay_df.empty:
                pay_df["total_collected"] = pay_df["total_collected"].astype(float)
                fig_pay = px.pie(
                    pay_df,
                    names="payment_method",
                    values="total_collected",
                    hole=0.45,
                    color_discrete_sequence=px.colors.qualitative.Set2
                )
                st.plotly_chart(fig_pay, use_container_width=True)
            else:
                st.info("No payment splits recorded yet.")

        # Charts Row 2
        st.subheader("Sales Trend Over Time")
        trend_df = sales_df.groupby("date", as_index=False)["gross_sales"].sum().sort_values("date")
        fig_trend = px.line(
            trend_df,
            x="date",
            y="gross_sales",
            markers=True,
            labels={"date": "Sale Date", "gross_sales": "Gross Sales (₹)"}
        )
        st.plotly_chart(fig_trend, use_container_width=True)

        # Detailed Table with Clear Currency Formatting
        st.subheader("Detailed Sales Ledger")
        st.dataframe(
            sales_df,
            use_container_width=True,
            hide_index=True,
            column_config={
                "gross_sales": st.column_config.NumberColumn("Gross Sales", format="₹ %.2f"),
                "received_amount": st.column_config.NumberColumn("Received Amount", format="₹ %.2f"),
                "pending_amount": st.column_config.NumberColumn("Pending Amount", format="₹ %.2f"),
            }
        )

    # ==========================================================================
    # VIEW 2: RECORD NEW SALE
    # ==========================================================================
    elif nav == "Record New Sale":
        st.header("➕ Create New Customer Sale")
        branches_df = fetch_df("SELECT * FROM branches")

        with st.form("new_sale_form", clear_on_submit=True):
            col_a, col_b = st.columns(2)
            with col_a:
                if role == "Super Admin":
                    branch_dict = dict(zip(branches_df["branch_name"], branches_df["branch_id"]))
                    chosen_branch = st.selectbox("Select Branch", list(branch_dict.keys()))
                    target_branch_id = branch_dict[chosen_branch]
                else:
                    target_branch_id = int(assigned_branch_id)
                    st.text_input("Branch", value=assigned_branch_name, disabled=True)

                cust_name = st.text_input("Customer Full Name", placeholder="e.g., Arjun Mehta")
                mobile_num = st.text_input("Mobile Number (10 digits, Unique)", placeholder="e.g., 9876500001")

            with col_b:
                sale_date = st.date_input("Sale Date", value=date.today())
                product = st.selectbox("Course / Product", ["DS", "DA", "BA", "FSD"])
                gross_amt = st.number_input("Gross Sales Amount (₹)", min_value=500.0, step=500.0, value=25000.0)

            submitted = st.form_submit_button("Save Sale Record", type="primary")
            if submitted:
                if not cust_name.strip() or not mobile_num.strip():
                    st.error("Customer Name and Mobile Number are mandatory.")
                else:
                    try:
                        new_id = execute_write(
                            '''INSERT INTO customer_sales (branch_id, date, name, mobile_number, product_name, gross_sales)
                               VALUES (%s, %s, %s, %s, %s, %s)''',
                            (target_branch_id, sale_date, cust_name.strip(), mobile_num.strip(), product, gross_amt)
                        )
                        st.success(f"Sale #{new_id} created successfully! Pending amount initialized to ₹{gross_amt:,.2f}.")
                    except Exception as e:
                        st.error(f"Could not save sale (check if mobile number already exists): {e}")

    # ==========================================================================
    # VIEW 3: RECORD SPLIT PAYMENT
    # ==========================================================================
    elif nav == "Record Split Payment":
        st.header("💳 Record Customer Split Payment")
        st.caption("Adding a payment automatically triggers database recalculation of received_amount, pending_amount, and sale status.")

        open_sales = fetch_df(f'''
            SELECT cs.sale_id, b.branch_name, cs.name, cs.product_name,
                   cs.gross_sales, cs.received_amount, cs.pending_amount, cs.status
            FROM customer_sales cs
            JOIN branches b ON cs.branch_id = b.branch_id
            {where_clause}
            ORDER BY cs.status ASC, cs.sale_id DESC
        ''')

        if open_sales.empty:
            st.info("No sales records found.")
            return

        for col in ["gross_sales", "received_amount", "pending_amount"]:
            open_sales[col] = open_sales[col].astype(float)

        st.dataframe(
            open_sales,
            use_container_width=True,
            hide_index=True,
            column_config={
                "gross_sales": st.column_config.NumberColumn("Gross Sales", format="₹ %.2f"),
                "received_amount": st.column_config.NumberColumn("Received Amount", format="₹ %.2f"),
                "pending_amount": st.column_config.NumberColumn("Pending Amount", format="₹ %.2f"),
            }
        )

        pending_only = open_sales[open_sales["pending_amount"] > 0]
        if pending_only.empty:
            st.success("All sales in this view are fully paid and closed!")
            return

        with st.form("payment_split_form"):
            sale_labels = {
                f"Sale #{row['sale_id']} - {row['name']} (Pending: ₹{row['pending_amount']:,.2f})": row["sale_id"]
                for _, row in pending_only.iterrows()
            }
            selected_label = st.selectbox("Select Open Sale", list(sale_labels.keys()))
            selected_sale_id = int(sale_labels[selected_label])

            max_pending = float(
                pending_only.loc[pending_only["sale_id"] == selected_sale_id, "pending_amount"].iloc[0]
            )

            p_col1, p_col2, p_col3 = st.columns(3)
            with p_col1:
                pay_date = st.date_input("Payment Date", value=date.today())
            with p_col2:
                amt_paid = st.number_input(
                    "Amount Paid (₹)",
                    min_value=1.0,
                    max_value=max_pending,
                    value=min(10000.0, max_pending),
                    step=500.0
                )
            with p_col3:
                pay_method = st.selectbox("Payment Method", ["UPI", "Card", "Cash"])

            if st.form_submit_button("Submit Payment", type="primary"):
                try:
                    execute_write(
                        '''INSERT INTO payment_splits (sale_id, payment_date, amount_paid, payment_method)
                           VALUES (%s, %s, %s, %s)''',
                        (selected_sale_id, pay_date, amt_paid, pay_method)
                    )
                    st.success("Payment recorded! Trigger updated received_amount and pending_amount.")
                    st.rerun()
                except Exception as e:
                    st.error(f"Error recording payment: {e}")

    # ==========================================================================
    # VIEW 4: PENDING & PAYMENT SPLITS LEDGER
    # ==========================================================================
    elif nav == "Pending & Payment Splits":
        st.header("📑 Pending Collections & Payment Split History")

        tab1, tab2 = st.tabs(["⏳ Pending Payments (Open Sales)", "🧾 Complete Payment Split Log"])

        with tab1:
            pend_df = fetch_df(f'''
                SELECT cs.sale_id, b.branch_name, cs.date, cs.name, cs.mobile_number,
                       cs.product_name, cs.gross_sales, cs.received_amount, cs.pending_amount
                FROM customer_sales cs
                JOIN branches b ON cs.branch_id = b.branch_id
                {where_clause}
            ''')
            for col in ["gross_sales", "received_amount", "pending_amount"]:
                pend_df[col] = pend_df[col].astype(float)
            pend_df = pend_df[pend_df["pending_amount"] > 0]
            if not pend_df.empty:
                st.dataframe(
                    pend_df,
                    use_container_width=True,
                    hide_index=True,
                    column_config={
                        "gross_sales": st.column_config.NumberColumn("Gross Sales", format="₹ %.2f"),
                        "received_amount": st.column_config.NumberColumn("Received Amount", format="₹ %.2f"),
                        "pending_amount": st.column_config.NumberColumn("Pending Amount", format="₹ %.2f"),
                    }
                )
            else:
                st.success("Zero pending payments!")

        with tab2:
            splits_df = fetch_df(f'''
                SELECT ps.payment_id, ps.sale_id, b.branch_name, cs.name AS customer_name,
                       ps.payment_date, ps.amount_paid, ps.payment_method
                FROM payment_splits ps
                JOIN customer_sales cs ON ps.sale_id = cs.sale_id
                JOIN branches b ON cs.branch_id = b.branch_id
                {where_clause}
                ORDER BY ps.payment_id DESC
            ''')
            if not splits_df.empty:
                splits_df["amount_paid"] = splits_df["amount_paid"].astype(float)
            st.dataframe(
                splits_df,
                use_container_width=True,
                hide_index=True,
                column_config={
                    "amount_paid": st.column_config.NumberColumn("Amount Paid", format="₹ %.2f"),
                }
            )

    # ==========================================================================
    # VIEW 5: ALL 20 SQL ANALYTICAL QUERIES
    # ==========================================================================
    elif nav == "SQL Analytical Lab (20 Queries)":
        st.header("🔍 Predefined SQL Analytical Queries (All 20)")
        st.caption("Select any of the 20 project queries below to inspect the SQL syntax and execute it live against MySQL.")

        sql_queries = {
            "Q01 [Basic]: Retrieve all records from customer_sales":
                "SELECT * FROM customer_sales;",
            "Q02 [Basic]: Retrieve all records from branches":
                "SELECT * FROM branches;",
            "Q03 [Basic]: Retrieve all records from payment_splits":
                "SELECT * FROM payment_splits;",
            "Q04 [Basic]: Display all sales with status = 'Open'":
                "SELECT * FROM customer_sales WHERE status = 'Open';",
            "Q05 [Basic]: Retrieve all sales belonging to the Chennai branch":
                '''SELECT cs.*
FROM customer_sales cs
JOIN branches b ON cs.branch_id = b.branch_id
WHERE b.branch_name = 'Chennai';''',
            "Q06 [Aggregation]: Calculate total gross sales across all branches":
                "SELECT SUM(gross_sales) AS total_gross_sales FROM customer_sales;",
            "Q07 [Aggregation]: Calculate total received amount across all sales":
                "SELECT SUM(received_amount) AS total_received_amount FROM customer_sales;",
            "Q08 [Aggregation]: Calculate total pending amount across all sales":
                "SELECT SUM(pending_amount) AS total_pending_amount FROM customer_sales;",
            "Q09 [Aggregation]: Count total number of sales per branch":
                '''SELECT b.branch_name, COUNT(cs.sale_id) AS total_sales
FROM branches b
LEFT JOIN customer_sales cs ON b.branch_id = cs.branch_id
GROUP BY b.branch_name;''',
            "Q10 [Aggregation]: Find the average gross sales amount":
                "SELECT AVG(gross_sales) AS average_gross_sales FROM customer_sales;",
            "Q11 [Join]: Retrieve sales details along with the branch name":
                '''SELECT cs.sale_id, b.branch_name, cs.date, cs.name, cs.product_name,
       cs.gross_sales, cs.received_amount, cs.pending_amount, cs.status
FROM customer_sales cs
JOIN branches b ON cs.branch_id = b.branch_id;''',
            "Q12 [Join]: Retrieve sales details along with total payment received (using payment_splits)":
                '''SELECT cs.sale_id, cs.name, cs.gross_sales,
       IFNULL(SUM(ps.amount_paid), 0) AS total_payment_received
FROM customer_sales cs
LEFT JOIN payment_splits ps ON cs.sale_id = ps.sale_id
GROUP BY cs.sale_id, cs.name, cs.gross_sales;''',
            "Q13 [Join]: Show branch-wise total gross sales (using JOIN & GROUP BY)":
                '''SELECT b.branch_name, SUM(cs.gross_sales) AS branch_total_gross_sales
FROM branches b
JOIN customer_sales cs ON b.branch_id = cs.branch_id
GROUP BY b.branch_name;''',
            "Q14 [Join]: Display sales along with payment method used":
                '''SELECT cs.sale_id, cs.name, ps.payment_date, ps.amount_paid, ps.payment_method
FROM customer_sales cs
JOIN payment_splits ps ON cs.sale_id = ps.sale_id;''',
            "Q15 [Join]: Retrieve sales along with branch admin name":
                '''SELECT cs.sale_id, b.branch_name, b.branch_admin_name, cs.name, cs.gross_sales, cs.status
FROM customer_sales cs
JOIN branches b ON cs.branch_id = b.branch_id;''',
            "Q16 [Financial]: Find sales where pending amount is greater than 5000":
                "SELECT * FROM customer_sales WHERE pending_amount > 5000;",
            "Q17 [Financial]: Retrieve top 3 highest gross sales":
                "SELECT * FROM customer_sales ORDER BY gross_sales DESC LIMIT 3;",
            "Q18 [Financial]: Find the branch with highest total gross sales":
                '''SELECT b.branch_name, SUM(cs.gross_sales) AS total_gross_sales
FROM branches b
JOIN customer_sales cs ON b.branch_id = cs.branch_id
GROUP BY b.branch_name
ORDER BY total_gross_sales DESC
LIMIT 1;''',
            "Q19 [Financial]: Retrieve monthly sales summary (group by month & year)":
                '''SELECT YEAR(date) AS sales_year,
       MONTH(date) AS sales_month,
       COUNT(sale_id) AS total_orders,
       SUM(gross_sales) AS monthly_gross_sales,
       SUM(received_amount) AS monthly_received,
       SUM(pending_amount) AS monthly_pending
FROM customer_sales
GROUP BY YEAR(date), MONTH(date)
ORDER BY sales_year, sales_month;''',
            "Q20 [Financial]: Calculate payment method-wise total collection (Cash / UPI / Card)":
                '''SELECT payment_method, SUM(amount_paid) AS total_collection
FROM payment_splits
GROUP BY payment_method;'''
        }

        chosen_query = st.selectbox("Choose a Query", list(sql_queries.keys()))
        st.code(sql_queries[chosen_query], language="sql")

        if st.button("▶ Run Query", type="primary"):
            result_df = fetch_df(sql_queries[chosen_query])
            st.dataframe(result_df, use_container_width=True, hide_index=True)

# Entry Point
if not st.session_state.authenticated:
    render_login()
else:
    render_app()
"""

with open("app.py", "w") as f:
    f.write(app_code)

print("✅ Modular files created: schema.sql, db.py, app.py")

# ==============================================================================
# STEP 5: LAUNCH STREAMLIT & CLOUDFLARE PUBLIC URL (NO PASSWORD)
# ==============================================================================
subprocess.Popen(["streamlit", "run", "app.py", "--server.headless", "true"], stdout=open("streamlit.log", "w"), stderr=subprocess.STDOUT)
time.sleep(3)

tunnel = subprocess.Popen(
    ["cloudflared", "tunnel", "--url", "http://localhost:8501"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True
)

for line in tunnel.stdout:
    match = re.search(r"https://[a-zA-Z0-9-]+\.trycloudflare\.com", line)
    if match:
        print("\n" + "=" * 65)
        print("🚀 PROJECT IS LIVE! CLICK HERE (NO PASSWORD REQUIRED):")
        print(match.group(0))
        print("=" * 65)
        break
