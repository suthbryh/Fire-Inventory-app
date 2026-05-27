import streamlit as st
import pandas as pd
import sqlite3
from datetime import datetime, date
import plotly.express as px
from datetime import timedelta

# Initialize database
def init_db():
    conn = sqlite3.connect('fire_inventory.db')
    c = conn.cursor()
    
    # Items table
    c.execute('''CREATE TABLE IF NOT EXISTS items (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        category TEXT,
        item_type TEXT,
        serial_number TEXT,
        quantity INTEGER DEFAULT 1,
        location TEXT,
        status TEXT,
        purchase_date TEXT,
        last_inspection TEXT,
        next_inspection TEXT,
        notes TEXT
    )''')
    
    # Maintenance log table
    c.execute('''CREATE TABLE IF NOT EXISTS maintenance (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        item_id INTEGER,
        date TEXT,
        action TEXT,
        performed_by TEXT,
        notes TEXT,
        FOREIGN KEY (item_id) REFERENCES items (id)
    )''')
    conn.commit()
    conn.close()

init_db()

# Page config
st.set_page_config(page_title="FD Inventory", layout="wide", page_icon="🔥")
st.title("🔥 Fire Department Inventory System")
st.markdown("**Tools • Equipment • Apparatus**")

# Sidebar navigation
menu = st.sidebar.selectbox(
    "Navigation",
    ["Dashboard", "View Inventory", "Add New Item", "Maintenance Log", "Reports"]
)

conn = sqlite3.connect('fire_inventory.db')

# Helper functions
def get_all_items():
    return pd.read_sql_query("SELECT * FROM items", conn)

# Dashboard
if menu == "Dashboard":
    st.header("Department Overview")
    
    items = get_all_items()
    
    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.metric("Total Items", len(items))
    with col2:
        st.metric("Apparatus", len(items[items['item_type'] == 'Apparatus']) if not items.empty else 0)
    with col3:
        st.metric("In Service", len(items[items['status'] == 'In Service']) if not items.empty else 0)
    with col4:
        st.metric("Needs Inspection", 
                 len(items[items['next_inspection'] <= date.today().isoformat()]) if not items.empty else 0)
    
    if not items.empty:
        # Status pie chart
        status_counts = items['status'].value_counts()
        fig = px.pie(names=status_counts.index, values=status_counts.values, title="Item Status")
        st.plotly_chart(fig, use_container_width=True)
        
        # Category breakdown
        cat_counts = items['category'].value_counts()
        fig2 = px.bar(x=cat_counts.index, y=cat_counts.values, title="Items by Category")
        st.plotly_chart(fig2, use_container_width=True)

# View Inventory
elif menu == "View Inventory":
    st.header("Full Inventory")
    
    items = get_all_items()
    
    if not items.empty:
        # Filters
        col1, col2 = st.columns(2)
        with col1:
            category_filter = st.multiselect("Category", options=items['category'].dropna().unique().tolist())
        with col2:
            status_filter = st.multiselect("Status", options=items['status'].dropna().unique().tolist())
        
        filtered = items.copy()
        if category_filter:
            filtered = filtered[filtered['category'].isin(category_filter)]
        if status_filter:
            filtered = filtered[filtered['status'].isin(status_filter)]
        
        st.dataframe(filtered, use_container_width=True, hide_index=True)
        
        if st.button("Export to CSV"):
            csv = filtered.to_csv(index=False)
            st.download_button("Download CSV", csv, "fire_inventory.csv", "text/csv")
    else:
        st.info("No items in inventory yet. Add some from the menu.")

# Add New Item
elif menu == "Add New Item":
    st.header("Add New Item")
    
    with st.form("add_item_form"):
        name = st.text_input("Item Name *", placeholder="Engine 1 or Halligan Bar")
        category = st.selectbox("Category", ["Apparatus", "PPE", "Tools", "Equipment", "Hose", "Other"])
        item_type = st.selectbox("Type", ["Apparatus", "Equipment", "Tool"])
        serial_number = st.text_input("Serial Number / ID")
        quantity = st.number_input("Quantity", min_value=1, value=1)
        location = st.text_input("Location", placeholder="Station 1 - Bay 3 or Truck 51")
        status = st.selectbox("Status", ["In Service", "Out of Service", "In Maintenance", "Reserve"])
        purchase_date = st.date_input("Purchase Date", value=date.today())
        last_inspection = st.date_input("Last Inspection", value=date.today())
        next_inspection = st.date_input("Next Inspection Due", value=date.today() + timedelta(days=365))
        notes = st.text_area("Notes")
        
        submitted = st.form_submit_button("Add Item")
        if submitted and name:
            new_item = {
                "name": name,
                "category": category,
                "item_type": item_type,
                "serial_number": serial_number,
                "quantity": quantity,
                "location": location,
                "status": status,
                "purchase_date": purchase_date.isoformat(),
                "last_inspection": last_inspection.isoformat(),
                "next_inspection": next_inspection.isoformat(),
                "notes": notes
            }
            df = pd.DataFrame([new_item])
            df.to_sql('items', conn, if_exists='append', index=False)
            st.success(f"✅ {name} added successfully!")

# Maintenance Log
elif menu == "Maintenance Log":
    st.header("Maintenance & Inspection Log")
    
    items = get_all_items()
    if not items.empty:
        item_options = dict(zip(items['id'], items['name']))
        selected_item_id = st.selectbox("Select Item", options=item_options.keys(), format_func=lambda x: item_options[x])
        
        with st.form("maintenance_form"):
            action_date = st.date_input("Date", value=date.today())
            action = st.text_input("Action Performed", placeholder="Annual inspection, repaired pump, etc.")
            performed_by = st.text_input("Performed By", placeholder="FF Smith")
            maint_notes = st.text_area("Maintenance Notes")
            
            if st.form_submit_button("Log Maintenance"):
                c = conn.cursor()
                c.execute("""INSERT INTO maintenance 
                           (item_id, date, action, performed_by, notes) 
                           VALUES (?, ?, ?, ?, ?)""",
                         (selected_item_id, action_date.isoformat(), action, performed_by, maint_notes))
                conn.commit()
                
                # Update last inspection
                c.execute("UPDATE items SET last_inspection = ? WHERE id = ?", 
                         (action_date.isoformat(), selected_item_id))
                conn.commit()
                st.success("Maintenance logged!")
    else:
        st.warning("Add items first before logging maintenance.")

# Reports
elif menu == "Reports":
    st.header("Reports & Compliance")
    
    items = get_all_items()
    
    if not items.empty:
        st.subheader("Items Needing Inspection")
        today = date.today().isoformat()
        due = items[items['next_inspection'] <= today]
        if not due.empty:
            st.dataframe(due[['name', 'location', 'next_inspection', 'status']], use_container_width=True)
        else:
            st.success("✅ All items are up to date!")
        
        st.subheader("Maintenance History")
        maint = pd.read_sql_query("""
            SELECT m.*, i.name 
            FROM maintenance m 
            JOIN items i ON m.item_id = i.id 
            ORDER BY m.date DESC
        """, conn)
        if not maint.empty:
            st.dataframe(maint, use_container_width=True)
        else:
            st.info("No maintenance records yet.")
    else:
        st.info("No data available for reports.")

conn.close()

st.sidebar.markdown("---")
st.sidebar.caption("Fire Department Inventory App v1.0")
