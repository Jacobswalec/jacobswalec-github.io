# jacobswalec-github.io
finance tracker

import tkinter as tk
from tkinter import ttk, messagebox
import csv
from datetime import datetime
import os
from collections import defaultdict
import matplotlib.pyplot as plt

FILENAME = "finance_data.csv"

# --- Initialize the CSV file if missing ---
def init_file():
    if not os.path.exists(FILENAME):
        with open(FILENAME, "w", newline="") as f:
            writer = csv.writer(f)
            writer.writerow(["Date", "Type", "Category", "Amount", "Description"])

# --- Add income or expense ---
def add_transaction(t_type, category, amount, desc):
    try:
        amount = float(amount)
    except ValueError:
        messagebox.showerror("Invalid Input", "Amount must be a number.")
        return

    if not category:
        messagebox.showerror("Missing Data", "Please enter a category.")
        return

    with open(FILENAME, "a", newline="") as f:
        writer = csv.writer(f)
        writer.writerow([
            datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            t_type,
            category,
            f"{amount:.2f}",
            desc
        ])
    messagebox.showinfo("Success", f"{t_type.capitalize()} of ${amount:.2f} added!")
    clear_inputs()
    load_transactions()

# --- Load transactions into the table and update summary ---
def load_transactions():
    for row in tree.get_children():
        tree.delete(row)
    income = expense = 0
    with open(FILENAME, "r") as f:
        reader = csv.DictReader(f)
        for row in reader:
            tree.insert("", "end", values=(
                row["Date"], row["Type"], row["Category"],
                f"${row['Amount']}", row["Description"]
            ))
            if row["Type"] == "income":
                income += float(row["Amount"])
            else:
                expense += float(row["Amount"])
    balance_label.config(
        text=f"Balance: ${income - expense:.2f} | Income: ${income:.2f} | Expense: ${expense:.2f}"
    )

# --- Clear input fields ---
def clear_inputs():
    category_entry.delete(0, tk.END)
    amount_entry.delete(0, tk.END)
    desc_entry.delete(0, tk.END)

# --- Plot expenses by category ---
def show_charts():
    if not os.path.exists(FILENAME) or os.stat(FILENAME).st_size == 0:
        messagebox.showerror("No Data", "No transaction data available.")
        return

    expenses = defaultdict(float)
    with open(FILENAME, "r") as f:
        reader = csv.DictReader(f)
        for row in reader:
            if row["Type"] == "expense":
                expenses[row["Category"]] += float(row["Amount"])

    if not expenses:
        messagebox.showinfo("No Expenses", "No expense data to visualize yet.")
        return

    categories = list(expenses.keys())
    values = list(expenses.values())

    plt.figure(figsize=(6, 4))
    plt.bar(categories, values)
    plt.title("Expenses by Category")
    plt.ylabel("Amount ($)")
    plt.xlabel("Category")
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.show()

# --- Setup GUI ---
init_file()
root = tk.Tk()
root.title("💰 Personal Finance Tracker")
root.geometry("750x550")
root.resizable(False, False)

# Input Frame
frame_top = tk.Frame(root, pady=10)
frame_top.pack()

tk.Label(frame_top, text="Category:").grid(row=0, column=0, padx=5)
category_entry = tk.Entry(frame_top)
category_entry.grid(row=0, column=1, padx=5)

tk.Label(frame_top, text="Amount:").grid(row=0, column=2, padx=5)
amount_entry = tk.Entry(frame_top)
amount_entry.grid(row=0, column=3, padx=5)

tk.Label(frame_top, text="Description:").grid(row=0, column=4, padx=5)
desc_entry = tk.Entry(frame_top, width=20)
desc_entry.grid(row=0, column=5, padx=5)

tk.Button(frame_top, text="Add Income", bg="lightgreen", width=12,
          command=lambda: add_transaction("income", category_entry.get(), amount_entry.get(), desc_entry.get())
).grid(row=1, column=1, pady=5)

tk.Button(frame_top, text="Add Expense", bg="salmon", width=12,
          command=lambda: add_transaction("expense", category_entry.get(), amount_entry.get(), desc_entry.get())
).grid(row=1, column=3, pady=5)

tk.Button(frame_top, text="Show Charts", bg="lightblue", width=12,
          command=show_charts
).grid(row=1, column=5, pady=5)

# Transaction Table
columns = ("Date", "Type", "Category", "Amount", "Description")
tree = ttk.Treeview(root, columns=columns, show="headings", height=15)
for col in columns:
    tree.heading(col, text=col)
    tree.column(col, width=130)
tree.pack(pady=10)

# Scrollbar
scrollbar = ttk.Scrollbar(root, orient="vertical", command=tree.yview)
tree.configure(yscroll=scrollbar.set)
scrollbar.pack(side="right", fill="y")

# Summary Label
balance_label = tk.Label(root, text="Balance: $0.00", font=("Arial", 12, "bold"))
balance_label.pack(pady=5)

load_transactions()
root.mainloop()
